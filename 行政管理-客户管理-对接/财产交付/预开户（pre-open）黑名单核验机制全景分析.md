
> 适用系统：ms-customer-center（客户中心微服务） 分析日期：2026-06-16 分支：develop 用途：技术评审 / 安全审计
## 1. 核验触发点

黑名单核验在 `/third/pre-open` 接口中触发，位于 **Controller 方法调用业务逻辑之前**，通过 Spring 同步事件机制执行。

### 1.1 触发位置

**Controller:** `ms-customer-core/src/main/java/com/boot/customer/core/controller/CustomerController.java:434-446`

public RestfulResponse<PreOpenCustResultVo, GeneralMeta> preOpen(@RequestBody PreOpenCustInfo custInfo) {  
    // ① 发布同步事件 → 触发黑名单+实名校验  
    applicationContext.publishEvent(new CheckPreOpenEvent(this, custInfo));  
​  
    // ② 根据校验结果决定是否放行  
    if (!Boolean.FALSE.equals(custInfo.getDataCheckFlag())) {  
        return custMainService.preOpen(custInfo);   // 校验通过，执行业务逻辑  
    } else {  
        // 校验不通过，返回失败及详细错误列表  
        PreOpenCustResultVo resultVo = new PreOpenCustResultVo();  
        resultVo.setErrorBlackRealValidationList(custInfo.getErrorBlackRealValidationList());  
        RestfulResponse<PreOpenCustResultVo, GeneralMeta> failure = RestfulResponse.failure();  
        failure.setData(resultVo);  
        return failure;  
    }  
}

### 1.2 在业务流程中的位置

请求到达  
  │  
  ├─ [1] Controller 接收 PreOpenCustInfo  
  ├─ [2] ★ 发布 CheckPreOpenEvent (同步) ★  
  │      ├─ 黑名单核验 (BlackListService.checkPreOpenBlackList)  
  │      └─ 实名校验   (RealNmCtfnService.checkPreOpenRealName)  
  │      └─ 设置 dataCheckFlag  
  ├─ [3] 判断 dataCheckFlag  
  │      ├─ true  → custMainService.preOpen() → 入库、返回成功  
  │      └─ false → 直接返回失败 + errorBlackRealValidationList

**核心要点：** 黑名单核验发生在 Controller 接收请求之后、数据入库之前。`@EventListener` 为同步执行（非 `@TransactionalEventListener`），因此核验在同一个线程和事务上下文中完成。

### 1.3 校验开关

由配置项 `ms.customer.event.listener.ctc.customer-check.enabled` 控制：

- `true`：启用黑名单+实名校验（sit/uat/prod 环境均为 `true`）
    
- `false` 或缺失：**PreOpenSyncListener 不会被 Spring 加载**，跳过所有核验
    

  
# application-sit.yaml / application-uat.yaml / application-prod.yaml  
ms:  
  customer:  
    event:  
      listener:  
        ctc:  
          customer-check:  
            enabled: true

> 注意：`application-dev.yaml` 中未配置此项，因此开发环境默认不启用黑名单核验。

---

## 2. 黑名单类型与来源

### 2.1 核验类型总览

|类型|数据来源|实现方式|适用客户类型|
|---|---|---|---|
|反洗钱/制裁黑名单|外部 AML 系统 `blacklist-service`|Feign 远程调用|个人（10）、机构（11）|
|实名认证|外部 KYC 系统 `realname-service`|Feign 远程调用|个人（10）|

> 本系统**不维护本地黑名单库**，不进行本地数据库黑名单查询。所有黑名单判断均委托给外部反洗钱系统。

### 2.2 外部 AML 系统返回的命中详情

当 AML 系统判定命中黑名单时，`HitBlack` 结构体包含以下信息用于追溯：

|字段|含义|示例|
|---|---|---|
|`lskd`|名单种类|制裁名单、监管名单、行内黑名单等|
|`lssc`|名单来源|监管机构名称、数据供应商|
|`name`|匹配到的名单名称|-|
|`score`|匹配分值|0-100|
|`level`|主体名单级别|-|
|`rlevel`|地区名单级别|-|
|`certTp`/`certNO`|名单中的证件信息|-|
|`country`|名单中的国家|-|

这些信息由外部 AML 系统返回，本地系统不维护名单分类逻辑，仅根据 `result` 码（0/1/-1）做放行/预警/拦截判断。

---

## 3. 核验方式与实现细节

### 3.1 黑名单核验：外部 AML 系统调用

#### 3.1.1 接口定义

**Feign Client：** `BlackListApi` **服务名：** `blacklist-service` **接口路径：** `POST /aml/blackmonitor/interface/lst/multi/query.do` **认证方式：** 通过 `BlackListAuthInterceptor` 自动注入 `X-API-Key` 请求头 **端点来源：** 配置项 `ms.customer.feign.ctc.blacklist.url`

|环境|AML 系统地址|
|---|---|
|SIT|`https://aml-sit.crctrust.com`|
|UAT|`https://aml-uat.crctrust.com`|
|PROD|`https://aml.crctrust.com`|

#### 3.1.2 请求参数（BlackListBatchQueryVo）

{  
  "channel": "0",      // 介入方式：0=实时判断（姓名+证件号码）  
  "mode": "1",         // 模式：1=复用已有处理结果，无结果则首次查询  
  "reqList": [  
    {  
      "seq": 1,           // 序号（用于结果匹配）  
      "name": "张三",      // 客户名称  
      "certTp": "110001", // 证件类型（映射后的反洗钱标准码）  
      "certNO": "110101199001011234", // 证件号码  
      "sex": "1",         // 性别（1=男, 0=女），预开户场景不传  
      "country": "CHN",   // 国家（3位ISO码），预开户场景不传  
      "birthday": null,   // 出生日期，预开户场景不传  
      "cstp": "1",        // 主体类型：1=个人, 2=机构  
      "rscd": "13",       // 来源系统代码  
      "mbrc": "D001",     // 申请部门  
      "username": "admin" // 申请用户名称  
    }  
  ]  
}

**核心参数说明：**

- `channel="0"`：实时判断模式，AML 系统基于姓名+证件号码进行实时匹配
    
- `mode="1"`：如果该主体三要素之前已有处理结果，则复用（避免重复人工处理），否则按首次查询处理
    
- `cstp`：`"1"` = 个人（IDV_CSTP），`"2"` = 机构（ORG_CSTP）
    

#### 3.1.3 响应判定逻辑（BlackListBatchResultVo）

{  
  "returnCode": "10000",  // 10000=调用成功，非10000=失败  
  "batchNo": "BN20251201xxxx",  
  "result": 0,            // 总体匹配结果  
  "alertCount": 0,        // 预警主体总数  
  "detail": [  
    {  
      "seq": 1,           // 对应请求中的序号  
      "result": 0,        // 单个客户匹配结果 ★  
      "count": 0,         // 命中名单数量  
      "hitBlacks": []     // 命中详情  
    }  
  ]  
}

**单客户判定标准（最核心）：**

|`result` 值|含义|系统行为|
|---|---|---|
|`0`|**通过** — 未命中任何黑名单|放行，继续后续流程|
|`1`|**预警** — 命中低风险名单|**拦截**，视为不通过|
|`-1`|**禁止** — 命中高风险名单|**拦截**，视为不通过|

**判定逻辑代码：** `BlackListServiceImpl.java:549`

  
return blackRealValidationList.stream()  
    .map(BlackRealValidation::getBlackResult)  
    .allMatch(AUTH_RESULT::equals);  // AUTH_RESULT = "0"

即：**只有全部客户主体的 result 都为 "0" 才通过，任意一个为 "1" 或 "-1" 均视为整体不通过。**

#### 3.1.4 证件类型映射

系统内部证件类型码（A/B/C/…）会通过以下映射表转换为反洗钱标准码后发送给 AML 系统：

**个人证件类型映射（IDV_CERT_TP_MAP）：**

|内部码|含义|反洗钱标准码|
|---|---|---|
|A|居民身份证|110001|
|B|临时居民身份证|110003|
|C|户口簿|110005|
|D|港澳居民来往内地通行证|110019|
|E|台湾居民来往大陆通行证|110021|
|F1|港澳居民居住证|119019|
|F2|台湾居民居住证|119021|
|Q|外国人永久居留身份证|110029|
|U1/U2/U5/U6/U7/U8|各类护照|110028|
|U3|外国护照|110027|
|K/M|军官证/士兵证|110007|
|L|人民警察证|110009|
|其他|社保卡/居住证/通行证等|129999（其他境外）或 119999（其他境内）|

**机构证件类型映射（ORG_CERT_TP_MAP）：**

|内部码|含义|反洗钱标准码|
|---|---|---|
|A|组织机构代码证|610001|
|B|营业执照号|610047|
|C|统一社会信用代码|610099|
|J1|事业单位登记证书|610019|
|J2|社会团体登记证书|610023|
|O|金融许可证|610009|
|P1|国税登记证号码|610007|
|P2|地税登记证号码|610051|
|V|民办非企业登记号|610025|
|其他|-|639999|

#### 3.1.5 来源系统映射（SYSTEM_MAP）

|sysId|来源系统|rscd|
|---|---|---|
|ECIF|客户中心|13|
|CRM|CRM|05|
|FIW|家办工作台|06|
|MCS|普惠金融|03|
|SIW|结构工作台|11|
|TYIW|同业工作台|09|
|其他|默认|13|

### 3.2 实名校验（关联核验）

实名校验与黑名单核验在同一个事件监听器中串行执行。详细逻辑不在本次黑名单分析范围内，但简要说明：

- **Feign Client：** `RealNameApi` → `POST /kyc/api/perscredit/query`
    
- **服务名：** `realname-service`
    
- **认证方式：** `RealNameAuthInterceptor` 注入 `X-API-Key`
    
- **适用对象：** 仅个人客户（custTpCd=10）
    
- **特殊处理：** 证件类型为 "Q"（外国人永久居留身份证）时默认通过
    
- **认证级别选择：** 2A（姓名+证件号）、3A（+性别）、6A（+更多字段）等
    
- **结果：** `"1"` = 通过，`"0"` = 未通过
    
- **缓存：** 历史认证结果存入 `REAL_NM_CTFN` 表，通过分布式 Redis 锁防并发
    

---

## 4. 核验输入参数

### 4.1 从请求中提取的字段

预开户请求对象 `PreOpenCustInfo` 包含一个 `cusInfotList`（List<PreOpenCustMain>），每个 `PreOpenCustMain` 继承自 `CustMain`（数据库实体），包含以下用于黑名单核验的字段：

|字段|来源|黑名单查询参数|说明|
|---|---|---|---|
|`custNm`|`PreOpenCustMain.custNm`|`name`|客户名称 ★必传|
|`certTpCd`|`PreOpenCustMain.certTpCd`|`certTp`（经映射）|证件类型 ★必传|
|`certNo`|`PreOpenCustMain.certNo`|`certNO`|证件号码 ★必传|
|`custTpCd`|`PreOpenCustMain.custTpCd`|`cstp`|客户类型（10→"1", 11→"2"）|
|`sysId`|`PreOpenCustInfo.sysId`|`rscd`|来源系统，经 SYSTEM_MAP 映射|
|`blngDeptId`|`PreOpenCustInfo.blngDeptId`|`mbrc`|申请部门|
|`usrName`|`PreOpenCustInfo.usrName`|`username`|申请用户名称|

### 4.2 预开户场景不传的字段

与个人单笔开户（`personalSingle`）的黑名单核验不同，预开户场景**不传递**以下字段：

|字段|原因|
|---|---|
|`sex`（性别）|预开户请求不包含性别信息|
|`country`（国家）|预开户请求不包含国籍信息|
|`birthday`（出生日期）|预开户请求不包含出生日期|

这意味着预开户的黑名单核验**仅基于姓名+证件号码**进行匹配，精度低于单笔开户的核验（后者额外使用性别、国籍、出生日期）。

### 4.3 客户类型过滤

|custTpCd|客户类型|是否进行黑名单核验|
|---|---|---|
|10|个人（INDIVIDUAL）|是|
|11|机构（ORGANIZATION）|是|
|12|产品（PRODUCT）|**否，直接返回 true**|

---

## 5. 结果处理与异常控制

### 5.1 正常流程的分支处理

  
黑名单核验结果:  
​  
  ├─ ALL result = "0" (通过)  
  │    └─ 返回 true，继续实名校验  
  │  
  └─ 任意 result = "1" 或 "-1" (预警/禁止)  
       └─ 返回 false  
            └─ dataCheckFlag = false  
                 └─ Controller 返回 RestfulResponse.failure()  
                      └─ 响应体包含 errorBlackRealValidationList

### 5.2 异常处理

**场景一：AML 系统调用失败（returnCode ≠ "10000"）**

- 抛出 `BlackListException("调用黑名单校验接口失败")`
    
- 被 `checkBlackList` 方法的 `catch` 块捕获
    
- 所有查询项 `blackResult` 设为 `"-1"`
    
- `dataCheckFlag` 设为 `false`
    
- 返回 `false`
    

**场景二：AML 系统返回数据缺失（seq 不匹配）**

- 抛出 `BlackListException("黑名单校验接口返回错误")`
    
- 处理同场景一
    

**场景三：其他未知异常**

- 被 `catch (Exception e)` 捕获
    
- 所有查询项 `blackResult` 设为 `"-1"`，`blackResultDesc` 设为 `"黑名单校验服务异常：" + e.getMessage()`
    
- `dataCheckFlag` 设为 `false`
    
- 返回 `false`
    

**场景四：BlackListException 逃逸到 Controller 层**

- 如果 `BlackListException` 未被 `checkBlackList` 的 catch 捕获（例如在构造请求参数阶段抛出），会被全局异常处理器 `CustomerGlobalExceptionHandlerAdvice.blackException()` 捕获
    
- 返回 `RestfulResponse.failure(e.getMessage())`，日志级别为 `WARN`
    

### 5.3 错误信息返回

校验失败时，`errorBlackRealValidationList` 会通过 `PreOpenCustResultVo` 返回给调用方：

  
{  
  "success": false,  
  "data": {  
    "custResultList": null,  
    "errorBlackRealValidationList": [  
      {  
        "custDesc": null,  
        "custTpCd": "10",  
        "custNm": "张三",  
        "certTpCd": "A",  
        "certNo": "110101199001011234",  
        "blackResult": "1",           // 1=预警, -1=禁止  
        "blackResultDesc": null,      // 异常时才有值  
        "realResult": null,           // 黑名单失败后不执行实名  
        "realResultDesc": null  
      }  
    ]  
  }  
}

### 5.4 日志与审计

- 黑名单核验成功/失败均通过 `log.info` 记录每个主体的核验结果
    
- 异常通过 `log.error` 记录完整堆栈
    
- **不涉及**数据库审计日志写入、告警触发——这些由外部 AML 系统自行负责
    
- 本地 `cust_blacklist_confirm` 表存储的是**经人工确认的黑名单预警结果**，属于事后的黑名单确认流程，不在预开户实时核验路径内
    

---

## 6. 调用链与关键类

### 6.1 完整时序

  
Client  
  │  POST /third/pre-open  { PreOpenCustInfo }  
  ▼  
CustomerController.preOpen()                              [Controller]  
  │  
  ├─ applicationContext.publishEvent(                     [Spring 同步事件]  
  │       new CheckPreOpenEvent(this, custInfo))  
  │  
  │  ┌─────────────────────────────────────────────────┐  
  │  │ PreOpenSyncListener.orgCustCheck(event)          │  [事件监听器]  
  │  │                                                  │  
  │  │ ① blackListService.checkPreOpenBlackList()       │  [BlackListService]  
  │  │    │                                             │  
  │  │    ├─ 遍历 cusInfotList                          │  
  │  │    │  ├─ custTpCd=10 → cstp="1" + IDV映射       │  
  │  │    │  ├─ custTpCd=11 → cstp="2" + ORG映射       │  
  │  │    │  └─ custTpCd=12 → return true (跳过)       │  
  │  │    │                                             │  
  │  │    └─ checkBlackList(reqList, custInfo)          │  
  │  │       │                                          │  
  │  │       ├─ 构造 BlackListBatchQueryVo               │  
  │  │       │   mode="1", channel="0"                  │  
  │  │       │                                          │  
  │  │       ├─ blackListApi.queryBatch(vo)  ──────────┼──► AML 系统  
  │  │       │   [Feign + BlackListAuthInterceptor]     │   POST /aml/blackmonitor/  
  │  │       │   [Header: X-API-Key]                    │   interface/lst/multi/query.do  
  │  │       │                                          │  
  │  │       ├─ 检查 returnCode == "10000"              │  
  │  │       │                                          │  
  │  │       ├─ 逐 seq 映射结果为 BlackRealValidation    │  
  │  │       │   set errorBlackRealValidationList       │  
  │  │       │                                          │  
  │  │       └─ 判断 blackResult 全为 "0" → 返回 bool   │  
  │  │                                                  │  
  │  │ ② realNmCtfnService.checkPreOpenRealName()      │  [RealNmCtfnService]  
  │  │    │                                             │  
  │  │    ├─ 遍历 cusInfotList                          │  
  │  │    │  ├─ custTpCd≠10 → return true (跳过)       │  
  │  │    │  ├─ certTpCd="Q" → 默认通过                 │  
  │  │    │  └─ 其他 → checkRealResult() ──────────────┼──► KYC 系统  
  │  │    │                                             │   POST /kyc/api/perscredit/query  
  │  │    │                                             │  
  │  │    └─ 判断 realResult 全为 "1" → 返回 bool       │  
  │  │                                                  │  
  │  │ ③ dataCheckFlag = blackCheck && realCheck        │  
  │  └─────────────────────────────────────────────────┘  
  │  
  ├─ if (dataCheckFlag != false)  
  │    └─ custMainService.preOpen(custInfo)              [CustMainService]  
  │         ├─ 判断存量/新客户                            [CustMainServiceImpl]  
  │         ├─ 生成 custNo  
  │         ├─ 保存 CustMain + 子表 + CertInfo + DeptRel  
  │         ├─ 处理数据权限  
  │         └─ 返回 PreOpenCustResultVo  
  │  
  └─ else  
       └─ 返回 failure + errorBlackRealValidationList

### 6.2 关键类与文件清单

|层级|类/文件|路径|职责|
|---|---|---|---|
|Controller|`CustomerController`|`core/controller/CustomerController.java:431-446`|接收请求，发布事件，判断 gate|
|Event|`CheckPreOpenEvent`|`core/event/define/ctc/CheckPreOpenEvent.java`|携带 PreOpenCustInfo 的事件载体|
|Listener|`PreOpenSyncListener`|`core/event/listener/ctc/PreOpenSyncListener.java`|编排黑名单+实名校验|
|Service 接口|`BlackListService`|`core/service/BlackListService.java`|黑名单校验服务接口|
|Service 实现|`BlackListServiceImpl`|`core/service/impl/BlackListServiceImpl.java`|核心：参数构造、API 调用、结果判断|
|Service 实现|`CustMainServiceImpl`|`core/service/impl/CustMainServiceImpl.java:579-707`|预开户业务逻辑|
|Service 接口|`RealNmCtfnService`|`core/service/RealNmCtfnService.java`|实名校验服务接口|
|Service 实现|`RealNmCtfnServiceImpl`|`core/service/impl/RealNmCtfnServiceImpl.java:541-639`|实名校验核心逻辑|
|Feign Client|`BlackListApi`|`feign/ctc/BlackListApi.java`|AML 系统 HTTP 调用接口|
|Feign Client|`RealNameApi`|`feign/ctc/RealNameApi.java`|KYC 系统 HTTP 调用接口|
|Interceptor|`BlackListAuthInterceptor`|`interceptor/BlackListAuthInterceptor.java`|注入 X-API-Key 认证头|
|Config|`BlackListAuthConfig`|`config/BlackListAuthConfig.java`|读取 API Key 配置|
|Domain|`PreOpenCustInfo`|`entity/.../domain/PreOpenCustInfo.java`|预开户请求 DTO|
|Domain|`PreOpenCustMain`|`entity/.../domain/PreOpenCustMain.java`|预开户客户主体（继承 CustMain）|
|Domain|`BlackRealValidation`|`entity/.../domain/BlackRealValidation.java`|黑名单+实名联合校验结果|
|Domain|`BlackListBatchQueryVo`|`entity/.../domain/black/BlackListBatchQueryVo.java`|批量查询请求 VO|
|Domain|`BlackListBatchQueryInfo`|`entity/.../domain/black/BlackListBatchQueryInfo.java`|单个查询项|
|Domain|`BlackListBaseQueryInfo`|`entity/.../domain/black/BlackListBaseQueryInfo.java`|查询基类（含 name/certNO/cstp 等）|
|Domain|`BlackListBatchResultVo`|`entity/.../domain/black/BlackListBatchResultVo.java`|批量查询响应|
|Domain|`BlackListBaseResultInfo`|`entity/.../domain/black/BlackListBaseResultInfo.java`|单条结果（含 result/seq/hitBlacks）|
|Domain|`HitBlack`|`entity/.../domain/black/HitBlack.java`|命中黑名单详情|
|Domain|`CustBaseInfo`|`entity/.../domain/CustBaseInfo.java`|基础信息父类（含 dataCheckFlag）|
|Entity|`CustMain`|`entity/.../entity/CustMain.java`|客户主表实体|
|Exception|`BlackListException`|`exception/BlackListException.java`|黑名单异常类|
|Handler|`CustomerGlobalExceptionHandlerAdvice`|`exception/CustomerGlobalExceptionHandlerAdvice.java`|全局异常拦截（含 BlackListException）|
|Config|`application-{env}.yaml`|`ms-customer/src/main/resources/`|外部服务 URL 及 API Key|

---

## 7. 安全评估摘要

### 7.1 设计优点

1. **同步校验+前置拦截**：黑名单核验在数据持久化之前执行，命中即拒绝，不会产生脏数据
    
2. **外部系统专业化**：AML 名单库由专业反洗钱系统维护，本系统不持有名单数据，降低敏感数据泄漏面
    
3. **API Key 认证**：与 AML 系统间通过 `X-API-Key` 头进行服务间认证，密钥通过配置文件管理（生产环境需确保使用加密配置如 Jasypt）
    
4. **开关可控**：通过 `customer-check.enabled` 配置项可在应急场景下快速关闭核验
    
5. **细粒度结果返回**：核验失败时返回每个主体的具体结果码，便于调用方定位问题
    

### 7.2 需关注事项

1. **mode="1" 复用历史结果**：`mode="1"` 表示如果同一主体有历史处理结果则直接复用。这意味着如果某个客户之前被人工标记为"通过"，后续即使新增了黑名单数据，也不会立即拦截。这是反洗钱业务流程设计（避免重复人工处理），但审计时需了解此行为。
    
2. **预开户信息字段有限**：预开户不传性别、国籍、出生日期，匹配精度低于单笔开户场景，存在一定的误匹配/漏匹配风险。
    
3. **产品客户未核验**：产品客户（custTpCd=12）在预开户流程中直接返回 `true`，不进行黑名单核验，需确认业务上是否合理。
    
4. **异常兜底策略**：外部 AML 系统调用异常时，所有参与核验的主体统一标记为 `blackResult="-1"` 并拦截，属于"fail-closed"策略（安全优先，可用性次之）。
    

---

_本文档基于代码静态分析生成，建议与实际运行环境配置交叉验证。_