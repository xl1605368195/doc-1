# 检测算法

> 沉淀、分享、成长，让自己和他人都能有所收获！😄

## 一、源码

`open-rasp用js语言实现检测，我这里使用Java语言重写🌶`
```js
plugin.register('command', function (params, context) {
    var cmd        = params.command
    var server     = context.server
    var message    = undefined
    var raw_tokens = []


    // 算法1: 根据堆栈，检查是否为反序列化攻击。
    // 理论上，此算法不存在误报

    if (algorithmConfig.command_reflect.action != 'ignore') {
        // Java 检测逻辑
        if (server.language == 'java') {
            message = validate_stack_java(params.stack)
            if (message) {
                message = _("Reflected command execution - %1%", [message])
            }
        }

        // PHP 检测逻辑
        else if (server.language == 'php' && validate_stack_php(params.stack))
        {
            message = _("WebShell activity - Detected reflected command execution")
        }

        if (message)
        {
            return {
                action:     algorithmConfig.command_reflect.action,
                message:    message,
                confidence: 100,
                algorithm:  'command_reflect'
            }
        }
    }

    // 从 v0.31 开始，当命令执行来自非HTTP请求的，我们也会检测反序列化攻击
    // 但是不应该拦截正常的命令执行，所以这里加一个 context.url 检查
    if (! context.url) {
        return clean
    }

    // 算法2: 检测命令注入，或者命令执行后门
    if (algorithmConfig.command_userinput.action != 'ignore') {
        var reason     = false
        var min_length = algorithmConfig.command_userinput.min_length
        var parameters = context.parameter || {}
        var json_parameters = context.json || {}
        var unexploitable_filter = algorithmConfig.command_userinput.java_unexploitable_filter

        // 检查命令逻辑是否被用户参数所修改
        function _run(values, name)
        {
            var reason = false

            values.some(function (value) {
                if (value.length <= min_length) {
                    return false
                }
                
                // 检查用户输入是否存在于命令中
                var userinput_idx = cmd.indexOf(value)
                if (userinput_idx == -1) {
                    return false
                }

                if (cmd.length == value.length) {
                    reason = _("WebShell detected - Executing command: %1%", [cmd])
                    return true
                }
                
                // 懒加载，需要的时候初始化 token
                if (raw_tokens.length == 0) {
                    raw_tokens = RASP.cmd_tokenize(cmd)
                }

                if (is_token_changed(raw_tokens, userinput_idx, value.length)) {
                    reason = _("Command injection - command structure altered by user input, request parameter name: %1%, value: %2%", [name, value])
                    return true
                }
            })

            return reason
        }

        // 过滤java无法利用的命令注入
        if (server.language != 'java' || !unexploitable_filter || cmdJavaExploitable.test(cmd)) {
            // 匹配 GET/POST/multipart 参数
            Object.keys(parameters).some(function (name) {
                // 覆盖场景，后者仅PHP支持
                // ?id=XXXX
                // ?data[key1][key2]=XXX
                var value_list = []
                Object.values(parameters[name]).forEach(function (value){
                    if (typeof value == 'string') {
                        value_list.push(value)
                    } else {
                        value_list = value_list.concat(Object.values(value))
                    }
                })
                reason = _run(value_list, name)
                if (reason) {
                    return true
                }
            })
            // 匹配 header 参数
            if (reason == false && context.header != null) {
                Object.keys(context.header).some(function (name) {
                    if ( name.toLowerCase() == "cookie") {
                        var cookies = get_cookies(context.header.cookie)
                        for (name in cookies) {
                            reason = _run([cookies[name]], "cookie:" + name)
                            if (reason) {
                                return true
                            }
                        }
                    }
                    else if ( headerInjection.indexOf(name.toLowerCase()) != -1) {
                        reason = _run([context.header[name]], "header:" + name)
                        if (reason) {
                            return true
                        }
                    }
                    
                })
            }

            // 匹配json参数
            if (reason == false && Object.keys(json_parameters).length > 0) {
                var jsons = [ [json_parameters, "input_json"] ]
                while (jsons.length > 0 && reason === false) {
                    var json_arr = jsons.pop()
                    var crt_json_key = json_arr[1]
                    var json_obj = json_arr[0]
                    for (item in json_obj) {
                        if (typeof json_obj[item] == "string") {
                            reason = _run([json_obj[item]], crt_json_key + "->" + item)
                            if(reason !== false) {
                                break;
                            }
                        }
                        else if (typeof json_obj[item] == "object") {
                            jsons.push([json_obj[item], crt_json_key + "->" + item])
                        }
                    }
                }
            }
        }

        if (reason !== false)
        {
            return {
                action:     algorithmConfig.command_userinput.action,
                confidence: 90,
                message:    reason,
                algorithm:  'command_userinput'
            }
        }
    }

    // 算法3: 常用渗透命令
    if (algorithmConfig.command_common.action != 'ignore')
    {
        var reason = false
        if (cmdPostPattern.test(params.command))
        {           
            return {
                action:     algorithmConfig.command_common.action,
                message:    _("Webshell detected - Executing potentially dangerous command, command is %1%", [params.command]),
                confidence: 95,
                algorithm:  'command_common'
            }
        }     
    }

    // 算法4: 查找语法错误和敏感操作
    if (algorithmConfig.command_error.action != 'ignore') {
        if (raw_tokens.length == 0) {
            raw_tokens = RASP.cmd_tokenize(cmd)
        }
        var concat_char = algorithmConfig.command_error.concat_char
        var sensitive_cmd = algorithmConfig.command_error.sensitive_cmd
        var alarm_token = algorithmConfig.command_error.alarm_token

        var double_quote = 0
        var ticks = 0
        for (var i=0; i<raw_tokens.length; i++) {
            // 敏感token检测
            if (algorithmConfig.command_error.alarm_token_enable) {
                if (alarm_token.indexOf(raw_tokens[i].text) != -1) {
                    if ( !(i > 0 && i < raw_tokens.length-1 && raw_tokens[i-1].text == '"' && raw_tokens[i+1].text == '"')) {
                        return {
                            action:     algorithmConfig.command_error.action,
                            confidence: 90,
                            message:    _("Command execution - Sensitive command token detect: %1%", [raw_tokens[i].text]),
                            algorithm:  'command_error'
                        }
                    }
                }
            }

            // 敏感连接命令检测
            if (algorithmConfig.command_error.sensitive_cmd_enable) {
                if (raw_tokens[i+1] !== undefined &&
                    concat_char.indexOf(raw_tokens[i].text) != -1 &&
                    sensitive_cmd.indexOf(raw_tokens[i+1].text) != -1) {
                    return {
                        action:     algorithmConfig.command_error.action,
                        confidence: 70,
                        message:    _("Command execution - Sensitive command concat detect: %1% %2%", [raw_tokens[i].text, raw_tokens[i+1].text]),
                        algorithm:  'command_error'
                    }
                }
            }

            if (raw_tokens[i].text == "\"") {
                double_quote ++
            }
            else if (raw_tokens[i].text == "`") {
                ticks ++
            }
            else if (raw_tokens[i].text == "'" && algorithmConfig.command_error.unbalanced_quote_enable) {
                if ( !(i > 0 && i < raw_tokens.length-1 && raw_tokens[i-1].text == '"' && raw_tokens[i+1].text == '"')) {
                    return {
                        action:     algorithmConfig.command_error.action,
                        confidence: 70,
                        message:    _("Command execution - Detected unbalanced single quote!"),
                        algorithm:  'command_error'
                    }
                }
            }
        }

        // 引号不匹配检测
        if (algorithmConfig.command_error.unbalanced_quote_enable) {
            if (double_quote % 2 != 0) {
                return {
                    action:     algorithmConfig.command_error.action,
                    confidence: 70,
                    message:    _("Command execution - Detected unbalanced double quote!"),
                    algorithm:  'command_error'
                }
            }
            if (ticks % 2 != 0) {
                return {
                    action:     algorithmConfig.command_error.action,
                    confidence: 70,
                    message:    _("Command execution - Detected unbalanced backtick!"),
                    algorithm:  'command_error'
                }
            }
        }
    }

    // 算法5: 记录所有的命令执行
    if (algorithmConfig.command_other.action != 'ignore') 
    {
        return {
            action:     algorithmConfig.command_other.action,
            message:    _("Command execution - Logging all command execution by default, command is %1%", [params.command]),
            confidence: 90,
            algorithm:  'command_other'
        }
    }

    // 算法6: DNSlog检测
    if (algorithmConfig.command_dnslog.action != 'ignore') 
    {
        if (cmdDNSlogPatternCmd.test(params.command))
        {
            if (cmdDNSlogPatternDomain.test(params.command)) {
                return {
                    action:     algorithmConfig.command_dnslog.action,
                    message:    _("Command injection - Executing dnslog command, command is %1%", [params.command]),
                    confidence: 95,
                    algorithm:  'command_dnslog'
                }
            }
        }
    }

    return clean
})
```

## 二、分析

### 算法1：堆栈检测算法
不管是在离线分析还是在实时检测中，本算法是RASP中使用最为广泛的算法之一。
算法的检测原理也比较简单，当执行到命令执行时，获取当前调用栈，如果调用栈中存在非法栈，即可判定为攻击。
对于已经出现过的漏洞或者攻击的栈特征总结处特征。常用的非法攻击栈如下：
```
'com.thoughtworks.xstream.XStream.unmarshal':                                   "Using xstream library",
'java.beans.XMLDecoder.readObject':                                             "Using WebLogic XMLDecoder library",
'org.apache.commons.collections4.functors.InvokerTransformer.transform':        "Using Transformer library (v4)",
'org.apache.commons.collections.functors.InvokerTransformer.transform':         "Using Transformer library",
'org.apache.commons.collections.functors.ChainedTransformer.transform':         "Using Transformer library",
'org.jolokia.jsr160.Jsr160RequestDispatcher.dispatchRequest':                   "Using JNDI library (JSR 160)",
'com.sun.jndi.rmi.registry.RegistryContext.lookup':                             "Using JNDI registry service",
'org.apache.xbean.propertyeditor.JndiConverter':                                "Using JNDI binding class",
'com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig':                "Using JTA transaction manager",
'com.sun.jndi.url.ldap.ldapURLContext.lookup':                                  "Using LDAP factory service",
'com.alibaba.fastjson.JSON.parse':                                              "Using fastjson library",
'com.alibaba.fastjson.JSON.parseObject':                                        "Using fastjson library",
'com.alibaba.fastjson.JSON.parseArray':                                         "Using fastjson library",
'org.springframework.expression.spel.support.ReflectiveMethodExecutor.execute': "Using SpEL expressions",
'freemarker.template.utility.Execute.exec':                                     "Using FreeMarker template",
'org.jboss.el.util.ReflectionUtil.invokeMethod':                                "Using JBoss EL method",
'org.codehaus.groovy.runtime.ProcessGroovyMethods.execute':                     "Using Groovy library",
'bsh.Reflect.invokeMethod':                                                     "Using BeanShell library",
'jdk.scripting.nashorn/jdk.nashorn.internal.runtime.ScriptFunction.invoke':     "Using Nashorn engine",
'org.apache.shiro.io.DefaultSerializer.deserialize':                            "Using Shiro framework (DefaultSerializer)",
'com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase.readObject':                 "Using C3p0 library"
```
上面为攻击利用栈列表，其中左边为栈，右边为栈的攻击信息 （来源于open-rasp）

使用java语言实现如下：
```java
 public static String check(String[] stack) {                                  
     boolean userCode = false, reachedInvoke = false;                           
     String message = "";                                                      
     for (int i = 0; i < stack.length; i++) {                                 
         String method = stack[i];
         // 检查反射调用 -> 命令执行之间，是否包含用户代码
         if (!reachedInvoke) {                                                
             if (REFLECT_INVOKE.equals(method)) {                             
                 reachedInvoke = true;                                         
             }                                                                
             // 用户代码，即非 JDK、com.jrasp 相关的函数                                   
             if (!method.startsWith("java.")                                  
                     && !method.startsWith("sun.")                            
                     && !method.startsWith("com.sun.")                        
                     && !method.startsWith("com.jrasp.")) {                   
                 userCode = true;                                             
             }                                                                
         }
         
        if (method.startsWith('ysoserial.Pwner')) {
            message = "Using YsoSerial tool"
            break
        }

        if (method.startsWith('org.su18')) {
            message = "Using ysuserial tool"
            break
        }

        if (method.startsWith('net.rebeyond.behinder')) {
            message = "Using BeHinder defineClass webshell"
            break
        }

        if (method.startsWith('com.fasterxml.jackson.databind.')) {
            message = "Using Jackson deserialze method"
            break
        }
        
         // 对于如下类型的反射调用,                                                      
         // 1. 仅当命令直接来自反射调用才拦截                                                
         if (!userCode) {                                                      
             if ("ognl.OgnlRuntime.invokeMethod".equals(method)) {            
                 message = "Using OGNL library";                              
                 break;                                                       
             } else if (REFLECT_INVOKE.equals(method)) {                      
                 message = "Unknown vulnerability detected";                  
                 // 笔者注：这里没有立即返回，而是继续检测栈                                      
             }                                                                
         }                                                                    
                                                                              
         // 本算法的核心检测逻辑                                                        
         if (knowns.containsKey(method)) {                                    
             message = method;                                                
         }                                                                    
     }                                                                        
     return message;                                                           
 }                                                                            
```
上面算法的核心是：反射+命令执行。正常的用户也有命令执行的需求，但是一般的正常用户执行命令时不会使用反射，因为直接调用命令执行的api更加简单快捷，
看如下调用栈,这是一段用户正常的命令执行：
> 场景1：直接调用命令执行api
```
java.lang.ProcessImpl.start(ProcessImpl.java)      [1]<-----命令执行
java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
java.lang.Runtime.exec(Runtime.java:620)
java.lang.Runtime.exec(Runtime.java:485)
com.alibaba.inf.cto.util.ProcessInfoUtil.getSystemInfoByCommand(ProcessInfoUtil.java:256)  [2]<-----用户代码
com.alibaba.inf.cto.util.ProcessInfoUtil.getHostInfoByIp(ProcessInfoUtil.java:242)
com.alibaba.adsc.predict.monitor.ponitor.getHostName(PMonitor.java:105)
com.alibaba.adsc.predict.monitor.ponitor.lambda$makeSureExist$0(PMonitor.java:94)
com.alibaba.adsc.predict.monitor.ponitor$$Lambda$427/2097793174.run(Unknown Source)
java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
java.util.concurrent.FutureTask.run(FutureTask.java:266)
java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
java.lang.Thread.run(Thread.java:745)
```
用户正常的命令执行特征：用户代码直接执行了命令，中间无反射操作，是正常的业务命令执行。

> 场景2：正常用户执行命令的场景：命令执行的类的方法被框架代理

```
java.lang.ProcessImpl.start(ProcessImpl.java)                     [1]<-----命令执行
java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
java.lang.Runtime.exec(Runtime.java:620)
java.lang.Runtime.exec(Runtime.java:485)
com.alibaba.in.utils.ProcessUtils.execCommand(ProcessUtils.java:53)
com.alibaba.in.face_recognition.DeviceTokenService.getDeviceToken(DeviceTokenService.java:74)
com.alibaba.in.thrift.CryptoTServiceImpl.getDeviceToken(CryptoV3TServiceImpl.java:99)   [2]<-----用户代码
sun.reflect.GeneratedMethodAccessor53.invoke(Unknown Source)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:497)                                        [3]<-----反射   
com.alibaba.service.mobile.thrift.proxy.ThriftServerInvoker.doInvoke(ThriftServerInvoker.java:363)
com.alibaba.service.mobile.thrift.proxy.ThriftServerInvoker$1.handle(ThriftServerInvoker.java:389)
com.alibaba.service.mobile.thrift.server.filter.AccessLogFilter.filter(AccessLogFilter.java:39)
com.alibaba.dorado.rpc.handler.filter.InvokeChainBuilder$2.handle(InvokeChainBuilder.java:106)
com.alibaba.service.mobile.mtthrift.server.filter.ServerLimiterFilter.filter(ServerLimiterFilter.java:73)
com.alibaba.dorado.rpc.handler.filter.InvokeChainBuilder$2.handle(InvokeChainBuilder.java:106)
com.alibaba.service.mobile.thrift.proxy.ThriftServerInvoker.invoke(ThriftServerInvoker.java:304)
com.sun.proxy.$Proxy123.getDeviceToken(Unknown Source)                                  
com.alibaba.in.tservice.CryptoService$Processor$getDeviceToken.getResult(CryptoService.java:702)
com.alibaba.in.tservice.CryptoService$Processor$getDeviceToken.getResult(CryptoService.java:691)  [4]<-----抽象实现
org.apache.thrift.ProcessFunction.process(ProcessFunction.java:39)
org.apache.thrift.TBaseProcessor.process(TBaseProcessor.java:35)
com.alibaba.service.mobile.thrift.proxy.ThriftServerPublisher$MtTProcessor.process(ThriftServerPublisher.java:577)
com.alibaba.service.mobile.thrift.server.netty.DefaultServerHandler.handleRequest(DefaultServerHandler.java:273)
com.alibaba.service.mobile.thrift.server.netty.DefaultServerHandler$1.run(DefaultServerHandler.java:164)
java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
java.lang.Thread.run(Thread.java:745)
```
用户正常的命令执行特征：反射的执行[3]和命令执行[1]之间有用户代码[2]，也是正常的业务命令执行。     

> 场景2的漏报  
```java
java.lang.UNIXProcess.<init>(UNIXProcess.java:245)                                                     [1]<-----命令执行 
java.lang.ProcessImpl.start(ProcessImpl.java:134)
java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
java.lang.Runtime.exec(Runtime.java:621)
java.lang.Runtime.exec(Runtime.java:451)
java.lang.Runtime.exec(Runtime.java:348)
Exploit.<clinit>(Exploit.java:6)                                                                       [2]<-----用户代码  
sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
java.lang.reflect.Constructor.newInstance(Constructor.java:423)
java.lang.Class.newInstance(Class.java:442)
javax.naming.spi.NamingManager.getObjectFactoryFromReference(NamingManager.java:173)
javax.naming.spi.DirectoryManager.getObjectInstance(DirectoryManager.java:189)
com.sun.jndi.ldap.LdapCtx.c_lookup(LdapCtx.java:1114)
com.sun.jndi.toolkit.ctx.ComponentContext.p_lookup(ComponentContext.java:542)
com.sun.jndi.toolkit.ctx.PartialCompositeContext.lookup(PartialCompositeContext.java:177)
com.sun.jndi.toolkit.url.GenericURLContext.lookup(GenericURLContext.java:205)
com.sun.jndi.url.ldap.ldapURLContext.lookup(ldapURLContext.java:94)
javax.naming.InitialContext.lookup(InitialContext.java:417)
com.newrelic.agent.deps.ch.qos.logback.core.db.JNDIConnectionSource.lookupDataSource(JNDIConnectionSource.java:97)
com.newrelic.agent.deps.ch.qos.logback.core.db.JNDIConnectionSource.getConnection(JNDIConnectionSource.java:57)
sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:498)                                                       [3]<-----反射 
com.fasterxml.jackson.databind.ser.BeanPropertyWriter.serializeAsField(BeanPropertyWriter.java:688)    [4]<----栈黑名单
com.fasterxml.jackson.databind.ser.std.BeanSerializerBase.serializeFields(BeanSerializerBase.java:719)
com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(BeanSerializer.java:155)
com.fasterxml.jackson.databind.ser.DefaultSerializerProvider._serialize(DefaultSerializerProvider.java:480)
com.fasterxml.jackson.databind.ser.DefaultSerializerProvider.serializeValue(DefaultSerializerProvider.java:319)
com.fasterxml.jackson.databind.ObjectMapper._configAndWriteValue(ObjectMapper.java:3906)
com.fasterxml.jackson.databind.ObjectMapper.writeValueAsString(ObjectMapper.java:3220)
com.example.cve202036188.Controller.json(Controller.java:18)
sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:498)
org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:209)
org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:136)
org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:102)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:877)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:783)
org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:991)
org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:925)
org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:974)
org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:877)
javax.servlet.http.HttpServlet.service.md(HttpServlet.java:661)
org.springframework.web.servlet.FrameworkServlet.service.md(FrameworkServlet.java:851)
javax.servlet.http.HttpServlet.service.md(HttpServlet.java:742)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:109)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:93)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:198)
org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:496)
org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140)
org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81)
org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
org.apache.catalina.connector.CoyoteAdapter.service.md(CoyoteAdapter.java:342)
org.apache.coyote.http11.Http11Processor.service.md(Http11Processor.java:803)
org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:790)
org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1468)
org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
java.lang.Thread.run(Thread.java:748)
```
仅仅通过业务代码+反射的特征存在漏报的可能，需要加上栈黑名单。

#### 总结

+ [1] 命令执行来直接来源于用户代码调用，是正常命令执行；
+ [2] 框架代理用户的命令执行，是正常命令执行；
+ [3] 如果调用栈中匹配到黑名单，确认是攻击;

### 算法2：检测命令注入，或者命令执行后门
检测用户的输入参数中是否有包含命令执行，用户的输入包括：http 参数、cookie 和 header




### 算法3: 常用渗透命令

