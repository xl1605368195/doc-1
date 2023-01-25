# XXE 的基本原理

## 简介
XXE 漏洞全称 XML External Entity Injection 即 xml 外部实体注入漏洞，XXE 漏洞发生在应用程序解析 XML 输入时，没有禁止外部实体的加载，导致可加载恶意外部文件和代码，造成任意文件读取、命令执行、内网端口扫描、攻击内网网站、发起 DoS 攻击等危害。

## 防护
防止 XXE 的最安全方法始终是完全禁用 DTD（外部实体）。根据解析器的不同，该方法应类似于以下内容：
```java
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
```
禁用DTD还可以使解析器免受拒绝服务 (DoS) 攻击。如果不可能完全禁用 DTD，则必须以特定于每个解析器的方式禁用外部实体和外部文档类型声明。

下面提供了针对 Java 语言中常用的 XML 解析器的详细 XXE 预防指南。

## 禁用

使用 XML 库的 Java 应用程序特别容易受到 XXE 的攻击，因为大多数 Java XML 解析器的默认设置是启用 XXE。要安全地使用这些解析器，必须在使用的解析器中显式禁用 XXE。下面介绍如何在 Java 最常用的 XML 解析器中禁用 XXE 。


`DocumentBuilderFactory`, `SAXParserFactory` 和 `DOM4J XML` 解析器可以使用相同的方式进行配置，以保护它们免受 XXE 的侵害。

### JAXP DocumentBuilderFactory

这里以 DocumentBuilderFactory 为例，JAXP DocumentBuilderFactory.setFeature 方法允许开发人员控制启用或禁用哪些特定于实现的 XML 处理器功能。

可以在工厂或底层XMLReader setFeature方法上设置功能。

每个 XML 处理器实现都有自己的特性来控制 DTD 和外部实体的处理方式。

有关使用 的语法突出显示的示例代码片段SAXParserFactory，

```java
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException; // catching unsupported features

... 

DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
String FEATURE = null;
try {
    // 这个是基本的防御方式。 如果DTDs被禁用, 能够防止绝大部分的XXE
    FEATURE = "http://apache.org/xml/features/disallow-doctype-decl";
    dbf.setFeature(FEATURE, true);

    // 如果不能完全禁用DTDs，至少下面的几个需要禁用:
    FEATURE = "http://xml.org/sax/features/external-general-entities";
    dbf.setFeature(FEATURE, false);

    FEATURE = "http://xml.org/sax/features/external-parameter-entities";
    dbf.setFeature(FEATURE, false);

    FEATURE = "http://apache.org/xml/features/nonvalidating/load-external-dtd";
    dbf.setFeature(FEATURE, false);

    dbf.setXIncludeAware(false);
    dbf.setExpandEntityReferences(false);

    // remaining parser logic
    ...
} catch (ParserConfigurationException e) {

} catch (SAXException e) {
    
} catch (IOException e) {
   
}

// Load XML file or stream using a XXE agnostic configured parser...
DocumentBuilder safebuilder = dbf.newDocumentBuilder();
```

### XMLInputFactory

```java
// This disables DTDs entirely for that factory
xmlInputFactory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
// disable external entities
xmlInputFactory.setProperty("javax.xml.stream.isSupportingExternalEntities", false);
```

### 其他中间件禁用方式参考 OWASP 文档

[XML External Entity Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)

## RASP防护

RASP 防护 XXE 漏洞具备天然的优势： 利用 java agent 动态修改代码,禁用 xml 解析器禁用DTD,而不需要业务来修改任何代码。

下面的例子是 JRASP 禁用 javax.xml.parsers.DocumentBuilderFactory 的外部 DTD 功能。
newInstance 方法构造 xml 解析器对象之前禁用外部DTD功能
```java
private static final String[] FEATURE_DEFAULTS = new String[]{"http://apache.org/xml/features/disallow-doctype-decl#true", //       
        "http://xml.org/sax/features/external-general-entities#false", //                                                           
        "http://xml.org/sax/features/external-parameter-entities#false", //                                                         
        "http://apache.org/xml/features/nonvalidating/load-external-dtd#false"};                                                    
                                                                                                                                    
// javax.xml.parsers.DocumentBuilderFactory                                                                                         
public void closeDocumentBuilderFactoryConfigXXE() {                                                                                
    new EventWatchBuilder(moduleEventWatcher)                                                                                       
            .onClass("javax.xml.parsers.DocumentBuilderFactory")                                                                     
            .includeBootstrap()                                                                                                     
            .onBehavior("newInstance")                                                                                               
            .withEmptyParameterTypes()                                                                                              
            .onWatch(new AdviceListener() {                                                                                         
                @Override                                                                                                           
                public void afterReturning(Advice advice) throws Throwable {                                                        
                    long start = System.nanoTime();                                                                                 
                    DocumentBuilderFactory instance = (DocumentBuilderFactory) advice.getReturnObj();                               
                    instance.setXIncludeAware(false);                                                                               
                    instance.setExpandEntityReferences(false);                                                                      
                    for (String featureDefault : FEATURE_DEFAULTS) {                                                                
                        String[] featureValue = featureDefault.split("#");                                                          
                        try {                                                                                                       
                            instance.setFeature(featureValue[0], Boolean.valueOf(featureValue[1]));                                 
                        } catch (Exception e) {                                                                                     
                            // No worries if one feature is not supported.                                                          
                        }                                                                                                           
                    }                                                                                                               
                    long end = System.nanoTime();                                                                                   
                    logger.info("method: {}, 耗时: {} ms", "javax.xml.parsers.newInstance.", (end - start) / 1000000.0);              
                }                                                                                                                   
            });                                                                                                                     
}                                                                                                                                   
```

值得注意的是： JRASP对多种xml解析中间件都提拱了对应的安全防护模块，并且可以做到按需加载。