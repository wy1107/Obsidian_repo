## 一、问题描述

微服务通过 Feign 客户端调用 HTTPS 接口（目标地址为 IP 地址而非域名）时，出现 SSL 握手异常：

javax.net.ssl.SSLHandshakeException: java.security.cert.CertificateException:   
No subject alternative names matching IP address 10.132.137.163 found

### 异常场景

- Feign 客户端 `MsCustomerApi` 以 HTTPS 方式调用 `https://10.132.137.163/gateway/ms-cust/api/v1/cust-main/have-permission`
    
- 目标服务器的 SSL 证书仅包含域名形式的 SAN（Subject Alternative Names），不包含 IP 地址 `10.132.137.163`
    
- Java SSL 校验要求证书 SAN 中必须包含目标主机名/IP，否则拒绝连接
### 根本原因

SSL 证书的 SAN 扩展中没有目标 IP 地址。Java 的 `HostnameChecker.matchIP()` 在校验时发现证书 SAN 列表中没有匹配的 IP 地址条目，抛出 `CertificateException`。

---

## 二、当前解决方案

项目中通过自定义 `IgnoreHttpsSSLClient` 配置类，注册一个跳过 SSL 证书校验的 Feign `Client` Bean：

@Configuration  
public class IgnoreHttpsSSLClient {  
​  
    @Bean  
    @ConditionalOnMissingBean  
    public Client feignClient() {  
        try {  
            SSLContext ctx = SSLContext.getInstance("SSL");  
            X509TrustManager tm = new X509TrustManager() {  
                @Override  
                public void checkClientTrusted(X509Certificate[] chain, String authType) {}  
​  
                @Override  
                public void checkServerTrusted(X509Certificate[] chain, String authType) {}  
​  
                @Override  
                public X509Certificate[] getAcceptedIssuers() { return null; }  
            };  
            ctx.init(null, new TrustManager[]{tm}, null);  
            return new Client.Default(ctx.getSocketFactory(), (hostname, session) -> true);  
        } catch (Exception e) {  
            return null;  
        }  
    }  
}

### 解决思路

1. **跳过 TrustManager 校验**：自定义 `X509TrustManager`，所有 `checkClientTrusted` / `checkServerTrusted` 方法为空实现，不做任何证书合法性校验
    
2. **跳过 HostnameVerifier 校验**：`Client.Default` 构造时传入 `(hostname, session) -> true`，对所有主机名/IP 都返回 true，绕过 SAN 匹配检查
    
3. **替换默认 Feign Client**：通过 Spring Bean 注册，将默认的 `feign.Client.Default`（使用 JVM 默认 SSL 校验）替换为自定义的 SSL 忽略版本
    

---

## 三、潜在问题与注意事项

### 1. `@ConditionalOnMissingBean` 与 Seata 冲突

**现象**：项目中集成了 Seata 分布式事务，其自动配置会注册 `SeataFeignClient`（一个 Feign `Client` Bean），该 Bean 在 Spring 容器中已存在。

**影响**：`@ConditionalOnMissingBean` 的语义是"只有容器中没有 `Client` 类型 Bean 时才创建"。由于 `SeataFeignClient` 已存在，自定义的 `IgnoreHttpsSSLClient.feignClient()` Bean **不会被注册**，导致 SSL 忽略逻辑完全失效。

**堆栈佐证**：

  
at feign.Client$Default.execute(Client.java:104)        ← 使用的是默认 Client，不是自定义的  
at com.alibaba.cloud.seata.feign.SeataFeignClient.execute(SeataFeignClient.java:59) ← Seata 包装了默认 Client

**修复建议**：将 `@ConditionalOnMissingBean` 改为 `@Primary`，确保自定义 Bean 为首选，Seata 的 `SeataFeignClient` 会自动包装它：

  
@Bean  
@Primary  // 替换 @ConditionalOnMissingBean  
public Client feignClient() { ... }

### 2. 安全风险

此方案跳过了所有 SSL 证书校验（信任管理 + 主机名验证），存在以下风险：

- **中间人攻击（MITM）**：无法验证服务器身份，攻击者可伪造服务器拦截通信
    
- **数据泄露**：敏感数据（如用户信息、Token）可能在未验证的连接上传输
    
- **仅适用于内网/开发环境**：生产环境强烈不建议使用
    

### 3. `getAcceptedIssuers()` 返回 null

部分 JVM 实现对 `getAcceptedIssuers()` 返回 null 有兼容性问题，建议改为返回空数组：

  
@Override  
public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }

### 4. `feignClient()` 异常时返回 null

`try-catch` 中异常时返回 `null`，这会导致 Spring 注册一个 `null` Bean，可能引发 `NullPointerException`。建议改为抛出异常或返回默认 Client：

  
} catch (Exception e) {  
    return new Client.Default(null, null);  // 退回到默认行为  
}

---

## 四、其他可选方案

| 方案                            | 描述                                                                | 适用场景                               |
| ----------------------------- | ----------------------------------------------------------------- | ---------------------------------- |
| **方案 A：证书 SAN 添加 IP**         | 让目标服务在证书 SAN 中添加 IP 地址条目                                          | 生产环境首选，安全合规                        |
| **方案 B：使用域名替代 IP**            | Feign 调用使用域名而非 IP 地址                                              | 证书已有域名 SAN 时适用                     |
| **方案 C：JVM 参数绕过**             | 启动参数 `-Djdk.internal.httpclient.disableHostnameVerification=true` | 仅 JDK 11+ 内置 HttpClient，不适用于 Feign |
| **方案 D：自定义 SSLSocketFactory** | 当前项目方案，全局跳过 SSL 校验                                                | 内网/开发环境快速解决                        |

---

## 五、结论

当前 `IgnoreHttpsSSLClient` 配置类的设计思路正确，但存在 `@ConditionalOnMissingBean` 与 Seata 的 Bean 冲突问题，导致自定义 Client **实际未生效**。需要改为 `@Primary` 注解才能确保 SSL 忽略逻辑被 Feign 调用链使用。同时需注意此方案仅适用于内网/开发环境，生产环境应优先考虑证书合规方案。