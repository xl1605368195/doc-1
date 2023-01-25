# tomcat-hook 模块参数完整性测试

# jdk8+tomcat10
+ java version "1.8.0_301"
+ apache-tomcat-10.0.22

```java
{
    "@timestamp":"2022-07-16T00:38:25.558+08:00",
    "@version":1,
    "message":{
        "attackParameters":"sh -c ls -la ",
        "attackTime":1657903105525,
        "attackType":"rce",
        "blocked":false,
        "checkType":"other-command",
        "httpInfo":{
            "protocol":"HTTP/1.1",
            "method":"GET",
            "remoteHost":"0:0:0:0:0:0:0:1",
            "requestURL":"http://localhost:8080/vulns/004-command-2.jsp",
            "header":{
                "sec-fetch-mode":"navigate",
                "referer":"http://localhost:8080/vulns/004-command-2.jsp",
                "sec-fetch-site":"same-origin",
                "accept-language":"zh-CN,zh;q=0.9",
                "cookie":"JSESSIONID=22BE555EA137B0F76C2B094DB2F56F49; context-profile-id=87e81720-9566-4055-b63b-ed08dbf5f3fb; OFBiz.Visitor=10000; Idea-56e01fa7=485bff85-5dd9-40a0-b248-3ea4ffce9002; Hm_lvt_5819d05c0869771ff6e6a81cdec5b2e8=1646641488; Hm_lvt_0b31b4c146bf7126aed5009e1a4a11c8=1657164265; Idea-c02f644a=c54b37f8-89c0-45a5-95f7-5b86308a47dc; Hm_lpvt_0b31b4c146bf7126aed5009e1a4a11c8=1657878606",
                "sec-fetch-user":"?1",
                "accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9",
                "sec-ch-ua":"\\\".Not/A)Brand\\\";v=\\\"99\\\", \\\"Google Chrome\\\";v=\\\"103\\\", \\\"Chromium\\\";v=\\\"103\\\"",
                "sec-ch-ua-mobile":"?0",
                "sec-ch-ua-platform":"\\\"macOS\\\"",
                "host":"localhost:8080",
                "upgrade-insecure-requests":"1",
                "connection":"keep-alive",
                "accept-encoding":"gzip, deflate, br",
                "user-agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36",
                "sec-fetch-dest":"document"
            },
            "localAddr":"0:0:0:0:0:0:0:1",
            "parameterMap":{
                "cmd":[
                    "ls -la /"
                ]
            }
        },
        "level":50,
        "stackTrace":[
            "java.lang.UNIXProcess.forkAndExec(UNIXProcess.java)",
            "java.lang.UNIXProcess.<init>(UNIXProcess.java:247)",
            "java.lang.ProcessImpl.start(ProcessImpl.java:134)",
            "java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)",
            "org.apache.jsp._004_002dcommand_002d2_jsp._jspService(_004_002dcommand_002d2_jsp.java:151)",
            "org.apache.jasper.runtime.HttpJspBase.service(HttpJspBase.java:70)",
            "jakarta.servlet.http.HttpServlet.service(HttpServlet.java:777)",
            "org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:466)",
            "org.apache.jasper.servlet.JspServlet.serviceJspFile(JspServlet.java:380)",
            "org.apache.jasper.servlet.JspServlet.service(JspServlet.java:328)",
            "jakarta.servlet.http.HttpServlet.service(HttpServlet.java:777)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:223)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:158)",
            "org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:185)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:158)",
            "org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:197)",
            "org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97)",
            "org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542)",
            "org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:119)",
            "org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)",
            "org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:690)",
            "org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78)",
            "org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:356)",
            "org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:399)",
            "org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)",
            "org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:867)",
            "org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1760)",
            "org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)",
            "org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191)",
            "org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)",
            "org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)",
            "java.lang.Thread.run(Thread.java:748)"
        ]
    },
    "logger_name":"com.jrasp.module.rcenative.algorithm.RceAlgorithm",
    "thread_name":"http-nio-8080-exec-1",
    "level":"WARN",
    "level_value":30000,
    "log_id":3012,
    "host_name":"MacBook-Pro.local",
    "pid":"87535"
}
```

参数获取正常，符合预期。

## # jdk11+tomcat10
+ java version "11"
+ apache-tomcat-10.0.22
```java
{
    "@timestamp":"2022-07-16T00:55:38.900+08:00",
    "@version":1,
    "message":{
        "attackParameters":"sh -c ls -la ",
        "attackTime":1657904138898,
        "attackType":"rce",
        "blocked":false,
        "checkType":"other-command",
        "httpInfo":{
            "protocol":"HTTP/1.1",
            "method":"GET",
            "remoteHost":"0:0:0:0:0:0:0:1",
            "requestURL":"http://localhost:8080/vulns/004-command-2.jsp",
            "header":{
                "sec-fetch-mode":"navigate",
                "referer":"http://localhost:8080/vulns/004-command-2.jsp",
                "sec-fetch-site":"same-origin",
                "accept-language":"zh-CN,zh;q=0.9",
                "cookie":"JSESSIONID=AAEE11104AAC1E13811B2EDBB813DE67; context-profile-id=87e81720-9566-4055-b63b-ed08dbf5f3fb; OFBiz.Visitor=10000; Idea-56e01fa7=485bff85-5dd9-40a0-b248-3ea4ffce9002; Hm_lvt_5819d05c0869771ff6e6a81cdec5b2e8=1646641488; Hm_lvt_0b31b4c146bf7126aed5009e1a4a11c8=1657164265; Idea-c02f644a=c54b37f8-89c0-45a5-95f7-5b86308a47dc; Hm_lpvt_0b31b4c146bf7126aed5009e1a4a11c8=1657903318",
                "sec-fetch-user":"?1",
                "accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9",
                "sec-ch-ua":"\\\".Not/A)Brand\\\";v=\\\"99\\\", \\\"Google Chrome\\\";v=\\\"103\\\", \\\"Chromium\\\";v=\\\"103\\\"",
                "sec-ch-ua-mobile":"?0",
                "sec-ch-ua-platform":"\\\"macOS\\\"",
                "host":"localhost:8080",
                "upgrade-insecure-requests":"1",
                "connection":"keep-alive",
                "accept-encoding":"gzip, deflate, br",
                "user-agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36",
                "sec-fetch-dest":"document"
            },
            "localAddr":"0:0:0:0:0:0:0:1",
            "parameterMap":{
                "cmd":[
                    "ls -la /"
                ]
            }
        },
        "level":50,
        "stackTrace":[
            "java.base/java.lang.ProcessImpl.forkAndExec(ProcessImpl.java)",
            "java.base/java.lang.ProcessImpl.<init>(ProcessImpl.java:340)",
            "java.base/java.lang.ProcessImpl.start(ProcessImpl.java:271)",
            "java.base/java.lang.ProcessBuilder.start(ProcessBuilder.java:1107)",
            "java.base/java.lang.ProcessBuilder.start(ProcessBuilder.java:1071)",
            "org.apache.jsp._004_002dcommand_002d2_jsp._jspService(_004_002dcommand_002d2_jsp.java:151)",
            "org.apache.jasper.runtime.HttpJspBase.service(HttpJspBase.java:70)",
            "jakarta.servlet.http.HttpServlet.service(HttpServlet.java:777)",
            "org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:466)",
            "org.apache.jasper.servlet.JspServlet.serviceJspFile(JspServlet.java:380)",
            "org.apache.jasper.servlet.JspServlet.service(JspServlet.java:328)",
            "jakarta.servlet.http.HttpServlet.service(HttpServlet.java:777)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:223)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:158)",
            "org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:185)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:158)",
            "org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:197)",
            "org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97)",
            "org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542)",
            "org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:119)",
            "org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)",
            "org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:690)",
            "org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78)",
            "org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:356)",
            "org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:399)",
            "org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)",
            "org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:867)",
            "org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1760)",
            "org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)",
            "org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191)",
            "org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)",
            "org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)",
            "java.base/java.lang.Thread.run(Thread.java:834)"
        ]
    },
    "logger_name":"com.jrasp.module.rcenative.algorithm.RceAlgorithm",
    "thread_name":"http-nio-8080-exec-4",
    "level":"WARN",
    "level_value":30000,
    "log_id":3012,
    "host_name":"MacBook-Pro.local",
    "pid":"89049"
}
```

参数获取正常，符合预期。

# jdk8+tomcat8

POST 请求
```java
{
    "@timestamp":"2022-07-16T01:20:46.868+08:00",
    "@version":1,
    "message":{
        "attackParameters":"touch /tmp/success",
        "attackTime":1657905646863,
        "attackType":"rce",
        "blocked":false,
        "checkType":"org.springframework.expression.spel.support.ReflectiveMethodExecutor.execute",
        "httpInfo":{
            "protocol":"HTTP/1.1",
            "method":"PATCH",
            "remoteHost":"0:0:0:0:0:0:0:1",
            "requestURL":"http://localhost:8080/people/1",
            "header":{
                "content-length":"234",
                "postman-token":"500813ea-5341-422c-94af-db101021afbf",
                "host":"localhost:8080",
                "content-type":"application/json-patch+json",
                "connection":"keep-alive",
                "cache-control":"no-cache",
                "accept-encoding":"gzip, deflate, br",
                "user-agent":"PostmanRuntime/7.28.4",
                "accept":"*/*"
            },
            "body":"[\\n    {\\n        \\\"op\\\": \\\"replace\\\",\\n        \\\"path\\\": \\\"T(java.lang.Runtime).getRuntime().exec(new java.lang.String(new byte[]{116,111,117,99,104,32,47,116,109,112,47,115,117,99,99,101,115,115}))/lastName\\\",\\n        \\\"value\\\": \\\"vulhub\\\"\\n    }\\n]",
            "localAddr":"0:0:0:0:0:0:0:1",
            "contentType":"application/json-patch+json"
        },
        "level":100,
        "stackTrace":[
            "java.lang.UNIXProcess.forkAndExec(UNIXProcess.java)",
            "java.lang.UNIXProcess.<init>(UNIXProcess.java:247)",
            "java.lang.ProcessImpl.start(ProcessImpl.java:134)",
            "java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)",
            "java.lang.Runtime.exec(Runtime.java:620)",
            "java.lang.Runtime.exec(Runtime.java:450)",
            "java.lang.Runtime.exec(Runtime.java:347)",
            "sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)",
            "sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)",
            "sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)",
            "java.lang.reflect.Method.invoke(Method.java:498)",
            "org.springframework.expression.spel.support.ReflectiveMethodExecutor.execute(ReflectiveMethodExecutor.java:113)",
            "org.springframework.expression.spel.ast.MethodReference.getValueInternal(MethodReference.java:129)",
            "org.springframework.expression.spel.ast.MethodReference.getValueInternal(MethodReference.java:85)",
            "org.springframework.expression.spel.ast.CompoundExpression.getValueRef(CompoundExpression.java:57)",
            "org.springframework.expression.spel.ast.CompoundExpression.setValue(CompoundExpression.java:95)",
            "org.springframework.expression.spel.standard.SpelExpression.setValue(SpelExpression.java:438)",
            "org.springframework.data.rest.webmvc.json.patch.PatchOperation.setValueOnTarget(PatchOperation.java:167)",
            "org.springframework.data.rest.webmvc.json.patch.ReplaceOperation.perform(ReplaceOperation.java:41)",
            "org.springframework.data.rest.webmvc.json.patch.Patch.apply(Patch.java:64)",
            "org.springframework.data.rest.webmvc.config.JsonPatchHandler.applyPatch(JsonPatchHandler.java:91)",
            "org.springframework.data.rest.webmvc.config.JsonPatchHandler.apply(JsonPatchHandler.java:83)",
            "org.springframework.data.rest.webmvc.config.PersistentEntityResourceHandlerMethodArgumentResolver.readPatch(PersistentEntityResourceHandlerMethodArgumentResolver.java:206)",
            "org.springframework.data.rest.webmvc.config.PersistentEntityResourceHandlerMethodArgumentResolver.read(PersistentEntityResourceHandlerMethodArgumentResolver.java:184)",
            "org.springframework.data.rest.webmvc.config.PersistentEntityResourceHandlerMethodArgumentResolver.resolveArgument(PersistentEntityResourceHandlerMethodArgumentResolver.java:141)",
            "org.springframework.web.method.support.HandlerMethodArgumentResolverComposite.resolveArgument(HandlerMethodArgumentResolverComposite.java:121)",
            "org.springframework.web.method.support.InvocableHandlerMethod.getMethodArgumentValues(InvocableHandlerMethod.java:158)",
            "org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:128)",
            "org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:97)",
            "org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:827)",
            "org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:738)",
            "org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)",
            "org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:967)",
            "org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:901)",
            "org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)",
            "org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:843)",
            "javax.servlet.http.HttpServlet.service(HttpServlet.java:742)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)",
            "org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)",
            "org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)",
            "org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)",
            "org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:105)",
            "org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)",
            "org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)",
            "org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)"
        ]
    },
    "logger_name":"com.jrasp.module.rcenative.algorithm.RceAlgorithm",
    "thread_name":"http-nio-8080-exec-5",
    "level":"WARN",
    "level_value":30000,
    "log_id":3012,
    "host_name":"MacBook-Pro.local",
    "pid":"90121"
}
```


参数名称修改：

"attackParameters":"touch /tmp/success" ----> 这个是方法的参数名称`hookMethodParameters`
