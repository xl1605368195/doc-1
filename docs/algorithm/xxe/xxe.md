# RASP漏洞防御之 XXE 漏洞


## 简介

当应用是通过用户上传的XML文件或POST请求进行数据的传输，并且应用没有禁止XML引用外部实体，也没有过滤用户提交的XML数据，那么就会产生XML外部实体注入漏洞。

XXE 漏洞在owasp2021中位置：

A05:2021 – Security Misconfiguration **CWE-611** Improper Restriction of XML External Entity Reference（XXE）

## 防护

> 使用语言中推荐的禁用外部实体的方法

这里以 Java 语言为例子说明。

使用XML库的Java应用程序特别容易受到XXE攻击，因为大多数Java XML解析器的**默认设置是启用XXE**。为了安全地使用这些解析器，必须在使用的解析器中显式禁用XXE。下面描述如何在最常用的Java XML解析器中禁用XXE。

### DocumentBuilderFactory
             
`javax.xml.parsers.DocumentBuilderFactory`

``` java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
String FEATURE = null;
FEATURE = "http://apache.org/xml/features/disallow-doctype-decl";
dbf.setFeature(FEATURE, true);

FEATURE = "http://xml.org/sax/features/external-general-entities";
dbf.setFeature(FEATURE, false);

FEATURE = "http://xml.org/sax/features/external-parameter-entities";
dbf.setFeature(FEATURE, false);

FEATURE = "http://apache.org/xml/features/nonvalidating/load-external-dtd";
dbf.setFeature(FEATURE, false);

dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```

### Dom4j

`org.dom4j.io.SAXReader`

``` java
saxReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
saxReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
saxReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

Based on testing, if you are missing one of these, you can still be vulnerable to an XXE attack.

### Jdom

 `org.jdom2.input.SAXBuilder`、 `org.jdom.input.SAXBuilder`

``` java
SAXBuilder builder = new SAXBuilder();
builder.setFeature("http://apache.org/xml/features/disallow-doctype-decl",true);
builder.setFeature("http://xml.org/sax/features/external-general-entities", false);
builder.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
builder.setExpandEntities(false);
Document doc = builder.build(new File(fileName));
```

### XMLInputFactory

`javax.xml.stream.XMLInputFactory`


``` java
// This disables DTDs entirely for that factory
xmlInputFactory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
// This causes XMLStreamException to be thrown if external DTDs are accessed.
xmlInputFactory.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, "");
// disable external entities
xmlInputFactory.setProperty("javax.xml.stream.isSupportingExternalEntities", false);
```


### XMLReader

 `org.xml.sax.XMLReader`

``` java
XMLReader reader = XMLReaderFactory.createXMLReader();
reader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
// This may not be strictly required as DTDs shouldn't be allowed at all, per previous line.
reader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
reader.setFeature("http://xml.org/sax/features/external-general-entities", false);
reader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

## RASP 

上面的修复方式在具体实施时存在缺陷：
+ 如果读取xml的代码在第三方包中（或者依赖框架），一般很难修改，必须pr框架/包提供者来修复；

这里以dom4j为例子来说RASP如何修复XX漏洞

## 这里提供一个有漏洞的案例
```java
package com.example.controller;

import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.ByteArrayInputStream;

@RestController
@RequestMapping("/xxe/dom4j")
public class Dom4jController {
    
    @PostMapping("/post/dom4j.do")
    public ResponseEntity<String> documentBuilder1(String xml, int fix) throws Exception {
        return ResponseEntity.ok(dom4j(xml, fix));
    }

    public static String dom4j(String xml, int fix) throws Exception {
        SAXReader saxReader = new SAXReader();
        // 启用修复方式
        if (fix == 1) {
            saxReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            saxReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
            saxReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            saxReader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        }
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(xml.getBytes());
        Document document = saxReader.read(byteArrayInputStream);
        Element element = document.getRootElement();
        return element.getText();
    }
}
```
> dom4j版本
```java
 <dependency>
    <groupId>org.dom4j</groupId>
    <artifactId>dom4j</artifactId>
    <version>2.0.0</version>
</dependency>
```

### POC

```shell
curl --location --request POST 'http://localhost:8080/xxe/dom4j/post/dom4j.do' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'xml=<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><root>&xxe;</root>' \
--data-urlencode 'fix=0'
```
读取 `/etc/passwd`文本内容

返回结果：

```shell
##
# User Database
# 
# Note that this file is consulted directly only when the system is running
# in single-user mode.  At other times this information is provided by
# Open Directory.
#
# See the opendirectoryd(8) man page for additional information about
# Open Directory.
##
nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0:System Administrator:/var/root:/bin/sh
daemon:*:1:1:System Services:/var/root:/usr/bin/false
...
```

### 模块插件举例

```java
    /**
     * read重载方法最终调用 read(InputSource)
     *
     * @see org.dom4j.io.SAXReader#read(InputSource)
     */
    public void closeDom4jXXE() {
        final String className = "org.dom4j.io.SAXReader";
        final String methdName = "read";
        new EventWatchBuilder(moduleEventWatcher)
                .onClass(className)
                .includeBootstrap()
                .onBehavior(methdName)
                .withParameterTypes("org.xml.sax.InputSource")
                .onWatch(new AdviceListener() {
                    @Override
                    public void before(Advice advice) throws Throwable {
                        if (!enableCheck) {
                            return;
                        }
                        SAXReader saxReader = (SAXReader) advice.getTarget();
                        saxReader.setFeature(FEATURE_DEFAULTS_1, true);
                        saxReader.setFeature(FEATURE_DEFAULTS_2, false);
                        saxReader.setFeature(FEATURE_DEFAULTS_3, false);
                    }

                    @Override
                    protected void afterThrowing(Advice advice) throws Throwable {
                        requestInfoThreadLocal.remove();
                    }
                });
    }
```

目前JRASP已经具备上面5类常用XML解析器的漏洞防护插件。



