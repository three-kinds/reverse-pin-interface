# 某聘APP接口逆向

* 声明：仅用作个人逆向思路展示，不提供具体工具

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


### 4. 响应解密

1. 根据请求逻辑，发现响应处理的核心在`package com.hpbr.apm.common.net;`的`public static <R extends com.hpbr.apm.common.net.ApmResponse> R a(okhttp3.Response r8, java.lang.Class<R> r9)`函数中
2. 但这个函数jadx没有逆向成功，看看smali
3. 手动跟踪到`package com.hpbr.apm.common.b;`的`static String a(byte[] bArr, String str, byte[] bArr2)`，这是核心的将二进制数据解为字符串的关键函数
4. 发现使用的加密方法为`AES/CBC/PKCS5Padding`，但没有直观的找到密钥，准备动态跟踪一下
