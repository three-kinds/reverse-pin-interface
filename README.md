# 某聘APP接口逆向

* 声明：仅用作个人逆向思路展示，不提供具体工具与成品代码

## 场景

* 最近在找工作，某聘APP个人比较喜欢；因官方提供的订阅鸡肋、每天都要进入APP搜索多次，想做一个自动的订阅、按照需求自动周期搜索；想直接通过调用官方接口实现

## 逆向流程

### 1. 先抓包看看接口情况

1. 使用mitmproxy抓http包（个人熟悉）
2. 已root手机的安装目标APP最新版`11.160`，配置手机系统级中间人证书，通过LSPosed+TrustMeAlready过ssl-pinning检测
3. 使用默认的走代理抓包，可行，mitmproxy拿到了接口数据，核心接口是`batchRunV2`
4. 但接口的请求参数复杂、响应结果是二进制数据、而且是多功能接口（通过路径名也能猜出来），明显经过了封装与加密，必须要逆向APP

### 2. 逆向APP

1. 解压APK，从目录结构与动态库搜索中没有发现没有明显的加壳，感觉很友好；如果有壳且市面上的工具、系统都无法脱壳的话，目前的我就搞不定了
2. 通过jadx逆向APP，发现大部分的代码都能成功逆向为Java，但工具没有解混淆功能，凑合看吧
3. 通过搜索http请求参数关键字，定位请求逻辑位置

### 3. 请求参数解密

1. http的header中有traceid，直接定位到请求前的逻辑
2. traceid就是`UUID.randomUUID().toString()`
3. todo: 还有很多参数待解


### 4. 响应解密

1. 根据请求逻辑，发现响应处理的核心在`package com.hpbr.apm.common.net;`的`public static <R extends com.hpbr.apm.common.net.ApmResponse> R a(okhttp3.Response r8, java.lang.Class<R> r9)`函数中
2. 但这个函数jadx没有逆向成功，看看smali
3. 手动跟踪到`package com.hpbr.apm.common.b;`的`static String a(byte[] bArr, String str, byte[] bArr2)`，这是核心的将二进制数据解为字符串的关键函数
4. 发现使用的加密方法为`AES/CBC/PKCS5Padding`，但没有直观的找到密钥，使用frida来hook一下上面的函数，打印出第二个入参密钥、与返回结果
5. 得到密钥为`YNEIVsMlw7bp7pfd`，在代码中也搜到了在`package com.hpbr.bosszhipin.manager.a.a;`中；返回结果为`{"code":0,"message":"Success","zpData":xxxx}`这样的格式
6. 经检测，上面找到的加密只是apm相关接口的、如周期统计，业务相关的`batchRunV2`与`jobList`接口走的不是这个解密，大意了没有细筛（感觉也不应该这么简单）
7. 根据`GeekF1GetJobListResponse`这个详细的业务类，找到`package com.twl.http.callback.b；`其中`a方法`为处理请求与响应，其中响应已经是明文，继续向上排查
8. 找到`com.twl.http.callback.e`，其中`parseResponse方法`很像是最根的处理方法了，但响应内容仍是明文；怀疑对`okhttp`有中间操作，果然有一些`intercept`方法
9. 在hook`net.bosszhipin.base.a`的`intercept方法`时，发现入的响应体为加密的二进制数据，出来的为明文string，继续深入
10. 发现在`com.twl.signer.YZWG`的`decodeContentBytes`是解response方法，它是个`native`方法（共有7个native方法），核心逻辑封装在`libyzwg.so`动态库中
11. 下面就是要逆向`libyzwg.so`，文件有`3.4MB`，用工具从内存dump还更大了一点儿。。。；目前用java封装成轻量server运行在手机上来说对我更容易（直接用frida包成server也能凑合用）
