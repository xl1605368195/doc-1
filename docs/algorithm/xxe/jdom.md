#JDOM

## 使用
```shell
package com.example.controller;

import org.jdom.Content;
import org.jdom.Document;
import org.jdom.Element;
import org.jdom.JDOMException;
import org.jdom.input.SAXBuilder;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.util.List;

@RestController
@RequestMapping("/xxe/jdom")
public class JdomSAXBuilderController {

    @PostMapping("/post/jdom.do")
    public ResponseEntity<String> documentBuilder1(String xml, int fix) throws Exception {
        return ResponseEntity.ok(jdom(xml, fix));
    }

    public static String jdom(String xml, int fix) throws JDOMException, IOException {
        StringBuilder stringBuffer = new StringBuilder();
        SAXBuilder saxBuilder;
        if (fix == 1) {
            // 修复方式1
            saxBuilder = new SAXBuilder(true);
        } else if (fix == 2) {
            // 修复方式2
            saxBuilder = new SAXBuilder();
            saxBuilder.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            saxBuilder.setFeature("http://xml.org/sax/features/external-general-entities", false);
            saxBuilder.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            saxBuilder.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        } else {
            // 有漏洞
            saxBuilder = new SAXBuilder();
        }
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(xml.getBytes());
        Document document = saxBuilder.build(byteArrayInputStream);
        Element element = document.getRootElement();
        List<Content> contents = element.getContent();
        for (Content content : contents) {
            stringBuffer.append(content.getValue());
        }
        return stringBuffer.toString();
    }
}
```
