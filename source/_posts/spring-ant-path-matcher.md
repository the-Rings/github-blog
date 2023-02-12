---
title: Spring AntPathMatcher
date: 2023-02-11 17:56:55
categories:
- Spring
tags:
- Ant-style
---

### Ant-sytle
实例化`PathMatchingResourcePatternResolver`是SpringIOC容器在启动过程中的一个小步骤。
```java
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
	// some code ...
	private PathMatcher pathMatcher = new AntPathMatcher();
}
```
这里的`AntPathMatcher`是借用了一些Apcache Ant中的代码，称为Ant风格路径匹配策略，首先列举常用的例子：
<p>The mapping matches URLs using the following rules:<br>
<ul>
<li>{@code ?} matches one character</li>
<li>{@code *} matches zero or more characters</li>
<li>{@code **} matches zero or more <em>directories</em> in a path</li>
<li>{@code {spring:[a-z]+}} matches the regexp {@code [a-z]+} as a path variable named "spring"</li>
</ul>

注：最后一条，就知道为什么`@PathVariable`可以将花括号中的值赋给一个变量了

*Table Example Ant-Style Path Patterns*
| Path                  |	Description |
| `/api/*.x`            |	匹配所有在app路径下的.x文件 |
| `/api/p?ttern`        |	匹配/app/pattern 和 /app/pXttern 但是不包括/app/pttern |
| `/**/example`         |	匹配/app/example/app/foo/example, 和 /example |
| `/api/**/dir/file.`   |	匹配/app/dir/file.jsp, /app/foo/dir/file.html,/app/foo/bar/dir/file.pdf, 和 /app/dir/file.java |
| `/**/*.jsp`           |	匹配任何的.jsp 文件 |
| `/api/{id}`           |   

`@RequestMapping`, `@CompenentScan`, `@PropertySource`以及spring-cloud-gateway中的URL匹配等都是Ant-style

### RegEx VS Ant-style
正则表达式几乎所有编程语言都支持的通用模式，URL相较于普通的字符串具有很强的规律性：标准的分段式。因此，使用轻量级Ant风格表达式作为URL的匹配模式更为合适。

### 如何测试
很多时候，URL匹配规则写完之后不知道是否正确，用最简单的方式来测试
```java
public class AntPathMatcherTest {
	private static final PathMatcher M = new AntPathMatcher();
	public static void main(String[] args) {
		String pattern = "/api/*.html";
		 System.out.println("是否匹配成功：" + MATCHER.match(pattern, "/api/index.html"));
	}
}
```
