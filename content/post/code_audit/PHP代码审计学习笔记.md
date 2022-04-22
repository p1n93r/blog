---
typora-root-url: ../../../static
title: "PHP代码审计学习笔记"
date: 2021-10-18T22:28:36+08:00
draft: false
categories: ["code_audit"]
---

## PHP的脆弱点汇总

### in_array()

这个函数常用于文件上传中的白名单检测，但是这个函数有一个可选的参数(第三个参数)，如果这个参数没有使用 `true` 的话，那么在进行比较的时候，是使用的松散比较，可能会出现强制类型转换，导致一些绕过：

```php
<?php
$whilte_list = range(1,10);
$name = "7hack.php";
if(in_array($name,$whilte_list)){
    echo "<h1>通过了验证！</h1>";
}
```

例如上面就可以绕过类型检查，上面 **之所以会发生强制类型转换，是因为目标数组中的元素为数字类型** 。

漏洞修复的话，就是将 `in_array()` 第三个参数设置成true，或者使用 `intval()` 强制将变量转换成数字：

```php
<?php
$whilte_list = range(1,10);
$name = "7hack.php";
if(in_array($name,$whilte_list,true)){
    echo "<h1>通过了验证！</h1>";
}
```

### filter_var()、htmlspecialchars()

filter_var() 函数通过指定的过滤器过滤一个变量。如果成功，则返回被过滤的数据。如果失败，则返回 FALSE。

htmlspecialchars() 函数把一些预定义的字符转换为 HTML 实体。

预定义的字符是：

- `&` （和号）成为 `&amp;`
- `"` （双引号）成为 `&quot;`
- `'` （单引号）成为 `'`
- `<` （小于）成为 `&lt;`
- `\>` （大于）成为 `&gt;`

如果存在下面这种场景，将造成XSS，其实这种XSS属于通用的漏洞场景，在href等属性中，很容易通过伪协议进行XSS绕过：

```php
<?php
$href = "javascript://comment%0aalert(1)";
# 先使用filter_val()过滤url
$href = filter_var($href,FILTER_SANITIZE_URL);
# 然后使用html进行实体编码
$href = htmlspecialchars($href);
echo "<a href='$href'>点我嗷!</a>";
```

此外， `filter_var($href,FILTER_SANITIZE_URL)` 也可能造成SSRF漏洞。一般对于SSRF的fuzz的特殊字符有：`;` ， `/` ， `?` ， `:` ， `@` ， `=` ， `&` 。SSRF绕过可以在这几个方向考虑；






