---
title: Tomcat7项目迁移到Tomcat8中文乱码问题
toc: true
date: 2016-07-18 00:09:09
categories:
- Tomcat系列
tags:
- 乱码问题
---

> 我打算开始使用Tomcat8了,先解决中文乱码问题,在解决其它的问题!
> **个人推荐:修改server.xml方式**
> 对于SpringMVC报的错误我稍后在补充问题

<!-- more -->

# 1.问题描述
Tomcat 7下项目切换到Tomcat 8后，出现乱码。 无论Google还是百度，多数解决方法是server.xml设置URIEncoding="UTF-8",这种配置为了解决GET请求的中文乱码问题。 对于Tomcat 7下遇到乱码问题，这样配置是正确的；但是对”Tomcat 7正常，切换到Tomcat 8”乱码的情况无效。

# 2. 解决方案[个人不喜欢]
Tomcat8的server.xml配置Connector节点添加属性URIEncoding="ISO-8859-1"。 
参考： http://tomcat.apache.org/migration-8.html

# 3. 官方文档解释
https://tomcat.apache.org/tomcat-7.0-doc/config/http.html

> URIEncoding
This specifies the character encoding used to decode the URI bytes, after %xx decoding the URL. If not specified, ISO-8859-1 will be used.

https://tomcat.apache.org/tomcat-8.0-doc/config/http.html
> URIEncoding
This specifies the character encoding used to decode the URI bytes, after %xx decoding the URL. If not specified, UTF-8 will be used unless the org.apache.catalina.STRICT_SERVLET_COMPLIANCE system property is set to true in which case ISO-8859-1 will be used.

Tomcat7对URI默认编码是ISO-8859-1 
Tomcat8对URI默认编码是UTF-8

#  4. 关于编码问题

## 4.1 Tomcat7这个URI默认的编码带来很多问题，下面这个应该很常见：
```java
new String(value.getBytes("ISO-8859-1"), param);
```
> 如果server.xml配置上URIEncoding="UTF-8"就不需要了。 进而项目直接迁移到Tomcat8，不修改server.xml，或者再次加上URIEncoding="UTF-8"也是不会有问题。

## 4.2 Tomcat8是不是就是因为开发者服务端转码麻烦，URI默认的编码改为"UTF-8"。
对于在Tomcat8开发项目，就简单很多，不需要上面的那段繁琐的代码。但是从Tomcat7迁移上来的项目，要么，去掉上面的代码，要么server.xml添加URIEncoding="ISO-8859-1"，浪费Tomcat8一番美意。

## 4.3 当然，通过判断参数值是否乱码，进行编码也是很不错的

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class ChineseUtill {

	private static boolean isChinese(char c) {
		Character.UnicodeBlock ub = Character.UnicodeBlock.of(c);
		if (ub == Character.UnicodeBlock.CJK_UNIFIED_IDEOGRAPHS
				|| ub == Character.UnicodeBlock.CJK_COMPATIBILITY_IDEOGRAPHS
				|| ub == Character.UnicodeBlock.CJK_UNIFIED_IDEOGRAPHS_EXTENSION_A
				|| ub == Character.UnicodeBlock.GENERAL_PUNCTUATION
				|| ub == Character.UnicodeBlock.CJK_SYMBOLS_AND_PUNCTUATION
				|| ub == Character.UnicodeBlock.HALFWIDTH_AND_FULLWIDTH_FORMS) {
			return true;
		}
		return false;
	}
	
	public static boolean isMessyCode(String strName) {
		Pattern p = Pattern.compile("\\s*|\t*|\r*|\n*");
		Matcher m = p.matcher(strName);
		String after = m.replaceAll("");
		String temp = after.replaceAll("\\p{P}", "");
		char[] ch = temp.trim().toCharArray();
		float chLength = 0 ;
		float count = 0;
		for (int i = 0; i < ch.length; i++) {
			char c = ch[i];
			if (!Character.isLetterOrDigit(c)) {
				if (!isChinese(c)) {
					count = count + 1;
				}
				chLength++; 
			}
		}
		float result = count / chLength ;
		if (result > 0.4) {
			return true;
		} else {
			return false;
		}
	}
	
	
	public static String toChinese(String msg){
		if(isMessyCode(msg)){
			try {
				return new String(msg.getBytes("ISO8859-1"), "UTF-8");
			} catch (Exception e) {
			}
		}
		return msg ; 
	}
}
```
