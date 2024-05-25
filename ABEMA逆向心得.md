ABEMA的逆向相对而言比较简单，没有涉及到Native层的内容，也没有太恶心的混淆。
万变不离其宗，先从检测到区域限制时弹出的错误提示`このサービスはお住まいの地域からはご利用になれません` 开始入手。常规版的代码比TV版复杂，因此先从TV版开始。
Android开发中字符串是单独存储在资源文件当中的，因此直接在资源文件中查找，得到字段名`error_abroad_label`
![](https://s2.loli.net/2024/05/25/SlWdPOptJjio1r8.png)
但使用`error_abroad_label`进行查找时却没有什么合适的结果，可以确定没有直接通过`R.string`的方式读取文本![](https://s2.loli.net/2024/05/25/sVzQUfagoTDpjXt.png)
扩大查找范围，字段名的关键词是`abroad`，以此进行查找，很快就出现了突破口`p167Lc.EnumC2721p`和`p167Lc.EnumC2722q`![](https://s2.loli.net/2024/05/25/JxtBWfAZnuCeXSz.png)
跳转到代码，右键查找用例，选择看上去代码最简单的`m12168a`![](https://s2.loli.net/2024/05/25/QNMAkpuv8TxYfUP.png)
可以清楚的看到在程序中所对应的错误：![](https://s2.loli.net/2024/05/25/gdfN4oTUFVnShrX.png)
继续查找`AppLaunchUseCase.RegionException`的用例，只有最后两条是有用的![](https://s2.loli.net/2024/05/25/G8EC3NpW1YLisDv.png)
阅读代码，可以注意到似乎与`cVar.f77009e`的值相关联![](https://s2.loli.net/2024/05/25/rYVu9U82m5WKTpQ.png)
但无论是cVar的类型`InterfaceC0880d`还是用例中的`c.invokeSuspend`都没有什么有用的信息![](https://s2.loli.net/2024/05/25/bEOm9JTGRKty47c.png) ![](https://s2.loli.net/2024/05/25/rY2TLpjCq6ubVzm.png)
返回`m74529A`方法继续阅读代码，注意到else代码块内有`regionCheckRepository`与我们的意图相匹配![](https://s2.loli.net/2024/05/25/Ndn9VTq7GW1cZ8y.png)
但是是个接口![image.png](https://s2.loli.net/2024/05/25/APsaUVxTKz7i3Wn.png)
查找该接口的用例，注意到`RegionCheckRepositoryImpl`继承了该接口，符合我们的要求![image.png](https://s2.loli.net/2024/05/25/nMBctD5dEGTiwJW.png)
排除内联类，该类只有三个方法：`mo65875a`、`mo65876b`、`mo65877c`。有鉴于这些方法一个个长得要死，且调用链复杂难以排查，直接使用Frida进行注入调试。调试用脚本如下：
```javascript
setTimeout(function() {
    Java.performNow(function() {
        let RegionCheckRepositoryImpl = Java.use("pb.V");
            RegionCheckRepositoryImpl["a"].implementation = function (interfaceC0880d) {
                console.log(`RegionCheckRepositoryImpl.mo65875a is called: interfaceC0880d=${interfaceC0880d}`);
                let result = this["a"](interfaceC0880d);
                console.log(`RegionCheckRepositoryImpl.mo65875a result=${result}`);
                return result;
            };
    });
    Java.performNow(function() {
        let RegionCheckRepositoryImpl = Java.use("pb.V");
        RegionCheckRepositoryImpl["b"].implementation = function (interfaceC0880d) {
            console.log(`RegionCheckRepositoryImpl.mo65876b is called: interfaceC0880d=${interfaceC0880d}`);
            let result = this["b"](interfaceC0880d);
            console.log(`RegionCheckRepositoryImpl.mo65876b result=${result}`);
            return result;
        };
    });
    Java.performNow(function() {
        let RegionCheckRepositoryImpl = Java.use("pb.V");
        RegionCheckRepositoryImpl["c"].implementation = function (interfaceC0880d) {
            console.log(`RegionCheckRepositoryImpl.mo65877c is called: interfaceC0880d=${interfaceC0880d}`);
            let result = this["c"](interfaceC0880d);
            console.log(`RegionCheckRepositoryImpl.mo65877c result=${result}`);
            return result;
        };
    });
}, 200);
```
Frida控制台返回如下内容：
```
RegionCheckRepositoryImpl.mo65877c is called: interfaceC0880d=[object Object]
RegionCheckRepositoryImpl.mo65877c result=COROUTINE_SUSPENDED
RegionCheckRepositoryImpl.mo65877c is called: interfaceC0880d=[object Object]
RegionCheckRepositoryImpl.mo65877c result=Failure(AppForbiddenExceptionEntity(message=anonymous_ip, cause=tv.abema.AppError$ApiForbiddenException: anonymous_ip
Source: Request{method=GET, url=https://api.p-c3-e.abema-tv.com/v1/ip/check?device=androidtv, headers=[newrelic:**DELETED**, traceparent:**DELETED**, tracestate:**DELETED**], tags={class retrofit2.Invocation=tv.abema.data.api.L$a.a() [androidtv]}}, statusCode=403, errorCode=0))
RegionCheckRepositoryImpl.mo65877c is called: interfaceC0880d=[object Object]
RegionCheckRepositoryImpl.mo65877c result=COROUTINE_SUSPENDED
RegionCheckRepositoryImpl.mo65877c is called: interfaceC0880d=[object Object]
RegionCheckRepositoryImpl.mo65877c result=Failure(AppForbiddenExceptionEntity(message=anonymous_ip, cause=tv.abema.AppError$ApiForbiddenException: anonymous_ip
Source: Request{method=GET, url=https://api.p-c3-e.abema-tv.com/v1/ip/check?device=androidtv, headers=[X-NewRelic-ID:**DELETED**, traceparent:**DELETED**, newrelic:**DELETED**, tracestate:**DELETED**], tags={class retrofit2.Invocation=tv.abema.data.api.L$a.a() [androidtv]}}, statusCode=403, errorCode=0))
```
因而确定实际调用的方法为`mo65877c`。注意到在使用被限制IP时，`mo65877c`方法返回Failure，猜想是该值决定ABEMA的区域检测结果。使用未被限制的IP重新注入，控制台返回如下内容：
```
RegionCheckRepositoryImpl.mo65877c result=COROUTINE_SUSPENDED
RegionCheckRepositoryImpl.mo65877c is called: interfaceC0880d=[object Object]
RegionCheckRepositoryImpl.mo65877c result=IPCheckEntity(isoCountryCode=CountryCodeEntity(isoCountryCode=JP), division=JAPAN)
```
因此，猜测只需将mo65877c方法的返回值强行修改为`IPCheckEntity(isoCountryCode=CountryCodeEntity(isoCountryCode=JP), division=JAPAN)`即可。以下是Frida示例代码：
```javascript
setTimeout(function() {
    Java.performNow(function() {
        let RegionCheckRepositoryImpl = Java.use("pb.V");
        let IPCheckEntity = Java.use("Ab.L0");
        let CountryCodeEntity = Java.use("Ab.X");
        let CountryCodeJP = CountryCodeEntity.$new("JP")
        let DivisionType = Java.use("Bb.j");
        let DivisionTypeJAPAN = DivisionType.c.value;
        let IPCheckEntityJP = IPCheckEntity.$new(CountryCodeJP, DivisionTypeJAPAN)
        RegionCheckRepositoryImpl["c"].implementation = function (interfaceC0880d) {
            return IPCheckEntityJP;
        };
    });
}, 200);
```
使用该脚本注入后成功绕过ABEMA区域检测，猜测正确，再将Frida脚本转换成Xposed模块即可。