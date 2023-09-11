# 某直聘APP接口逆向

## 场景

* 最近在找工作，某聘APP个人比较喜欢；因官方提供的订阅鸡肋、每天都要进入APP搜索多次，想做一个自动的订阅、按照需求自动周期搜索；想直接通过调用官方接口实现

## 逆向流程

### 1. 先抓包看看接口情况

1. 使用mitmproxy抓http包（个人熟悉）
2. 已root手机的安装目标APP最新版`11.160`，配置手机系统级中间人证书，通过LSPosed+TrustMeAlready过ssl-pinning检测
3. 使用默认的走代理抓包，可行，mitmproxy拿到了接口数据，核心接口是`batchRunV2`
4. 但接口的请求参数复杂、响应结果是二进制数据、而且是多功能接口（通过路径名也能猜出来）
5. `jobList`就行满足需求，但其请求与响应同样明显经过了封装与加密，必须要逆向APP

### 2. 逆向APP

1. 解压APK，从目录结构与动态库搜索中没有发现没有明显的加壳，感觉很友好；如果有壳且市面上的工具、系统都无法脱壳的话，目前的我就搞不定了
2. 通过jadx逆向APP，发现大部分的代码都能成功逆向为Java，但工具没有解混淆功能，凑合看吧
3. 通过搜索http请求参数关键字，定位请求、响应逻辑位置

### 3. 响应解密

1. 根据请求逻辑，发现响应处理的核心在`package com.hpbr.apm.common.net;`的`public static <R extends com.hpbr.apm.common.net.ApmResponse> R a(okhttp3.Response r8, java.lang.Class<R> r9)`函数中
2. 但这个函数jadx没有逆向成功，看看smali
3. 手动跟踪到`package com.hpbr.apm.common.b;`的`static String a(byte[] bArr, String str, byte[] bArr2)`，这是核心的将二进制数据解为字符串的关键函数
4. 发现使用的加密方法为`AES/CBC/PKCS5Padding`，但没有直观的找到密钥，使用frida来hook一下上面的函数，打印出第二个入参密钥、与返回结果
5. 得到密钥为`YNEIVsMlw7bp7pfd`，在代码中也搜到了在`package com.hpbr.bosszhipin.manager.a.a;`中；返回结果为`{"code":0,"message":"Success","zpData":xxxx}`这样的格式
6. 经检测，上面找到的加密只是apm相关接口的、如周期统计，业务相关的`batchRunV2`与`jobList`接口走的不是这个解密，大意了没有细筛（感觉也不应该这么简单）；以上是弯路
7. 根据`GeekF1GetJobListResponse`这个详细的业务类，找到`package com.twl.http.callback.b；`其中`a方法`为处理请求与响应，其中响应已经是明文，继续向上排查
8. 找到`com.twl.http.callback.e`，其中`parseResponse方法`很像是最根的处理方法了，但响应内容仍是明文；怀疑对`okhttp`有中间操作，果然有一些`intercept`方法
9. 在hook`net.bosszhipin.base.a`的`intercept方法`时，发现入的响应体为加密的二进制数据，出来的为明文string，继续深入
10. 发现在`com.twl.signer.YZWG`的`decodeContentBytes`是解response方法，它是个`native`方法（共有7个native方法），核心逻辑封装在`libyzwg.so`动态库中
11. 下面就是要逆向`libyzwg.so`，文件有`3.4MB`，用工具从内存dump还更大了一点儿。。。（老版本、没加固的才18KB）；想直接用AndServer将so封成轻量app服务，实践发现封不了、过不了内部的校验，只能勉强用frida包一下凑合用

### 4. 请求参数解密

1. http的header中有traceid，直接定位到请求前的逻辑
2. header: traceid: `UUID.randomUUID().toString()`
3. header: zp-accept-encoding: 1 （固定）
4. header: zp-accept-compressing: 3（固定）
5. header: zp-accept-encrypting: 1（固定）
6. header: user-agent: NetType/wifi Screen/1080X1920 BossZhipin/11.160 Android 28
7. header: t2(重要、access-token登录后必须): 对应`token2`（叫2估计是为了兼容升级），相关类`AccountHelper、AccountRepository`保存在`SharedPreferences`中key为`acc_t2`；在登录成功返回的结构中会包含这个字段（如使用`codeLogin`接口）
8. query: app_id: 1003 （固定）
9. query: sp（重要、加密后的业务请求体）:使用加密函数`encodeRequest(requestMapString, secretKey)`得到；requestMap就是业务请求信息实体，每个请求都不同，下面有样例，map到string的过程在`net.bosszhipin.base`中
10. query: sig（重要、签名）:使用加密函数`nativeSignature((requestURL + requestMapString), secretKey)`得到；secretKey在登录成功返回的结构中

#### 3.1 样例：jobList 的请求体

* page
* pageSize
* sortType
* filterParams： cityCode、switchCity
* expectId
* expectPosition
* encryptExpectId
* 上面是接口业务参数
* curidentity：GEEK or BOSS
* req_time： 请求时间戳
* uniqid：应用初始化时生成
* v: app版本号
* client_info：手机各种信息
