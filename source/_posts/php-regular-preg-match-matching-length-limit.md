layout: post
title: "PHP 正则preg_match匹配长度限制"
date: 2017-12-19 14:48
comments: true
tags: 
	- PHP
	- 正则
---

什么？有长度限制？我怎么没有碰到过？是的，有限制的，总不能无底洞一直运行下去吧！

## 故事背景
> 有一天，发现以前使用正则表达式写的HTML格式转换代码出问题了！压根没有进行格式转换！但是把部分内容(任意内容)删除之后，就可以匹配成功并且正常转换。于是我想这应该是长度限制，导致无法匹配结果。

<!--more-->
于是，在网上搜索了一下，发现的确是有这个限制：

**搜索关键词：** preg_match 长度限制

**得到结果：**

```
// preg_match有字符串长度限制，果然，发现最大回溯“pcre.backtrack_limit ”的值默认只设了100000。 正则的回溯是什么？这个不是本文重点，有需要可以自行在网上查询

// 解决办法：
ini_set('pcre.backtrack_limit', 999999999);

// 注：这个参数在php 5.2.0版本之后可用。

// 另外说说关于：pcre.recursion_limit

//pcre.recursion_limit是PCRE的递归限制，这个项如果设很大的值，会消耗所有进程的可用堆栈，最后导致PHP崩溃。

// 也可以通过修改配置来限制：
ini_set('pcre.recursion_limit', 99999);
```

**但是**，我都设置还是不行。于是发现这个`preg_last_error`函数：
[preg_last_error官方解释](http://php.net/manual/zh/function.preg-last-error.php) ：返回最后一个PCRE正则执行产生的错误代码

于是在`preg_match_all`之后执行`preg_last_error`得到结果`6`!!

[PORE预定义常量](http://php.net/manual/zh/pcre.constants.php)中，把所有常量打印一遍，发现 是 `PREG_JIT_STACKLIMIT_ERROR`。问题找到了，接下来是解决问题~

```
// 这里应该按照需要设置，尽量不要设置0
ini_set('pcre.jit', 0);
```
就这样解决了!


## 代码(故事主人公)：
```
// ini_set('pcre.jit', 0);

//不要管这坨东西,当时没想那么多。就别问我为什么不用(?P<name>)语法。
$pattern = '/<span([^<\/>]*)>(([^<>]|<br\/>)*<span[^<\/>]*>([^<>]|<br\/>)*<\/span>([^<>]|<br\/>)*)+<\/span>/i';

$content = '<span style="font-size: 24px;">
1、新增功能  
<br/>1）新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>2）新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>3）新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能
<br/>新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>
<br/>2、新增功能新增功能新增功能新增功能新增功能  
<br/>1）新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能
<br/>2）新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能
<br/>
<span style="font-size: 24px; color: rgb(255, 0, 0);">温馨提示：新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能  </span>
<br/>
<br/>3、新增功能新增功能  
<br/>1）新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/> 2）新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>
<br/>4、新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>新增功能新增功能新增功能新增功能新增功能新增功能新增功能   新增功能新增功能新增功能新增功能  
<br/>
<br/>5、新增功能新增功能新增功能新增功能新增功能新增功能新增功能  
<br/>6、新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能新增功能
</span>
';

preg_match_all($pattern, $content, $matches);
// 输出 preg_match_all 的错误码 0 表示没有错误
var_dump(preg_last_error());
$error = [
    'PREG_INTERNAL_ERROR' => PREG_INTERNAL_ERROR,
    'PREG_BACKTRACK_LIMIT_ERROR' => PREG_BACKTRACK_LIMIT_ERROR,
    'PREG_RECURSION_LIMIT_ERROR' => PREG_RECURSION_LIMIT_ERROR,
    'PREG_BAD_UTF8_ERROR' => PREG_BAD_UTF8_ERROR,
    'PREG_BAD_UTF8_OFFSET_ERROR' => PREG_BAD_UTF8_OFFSET_ERROR,
    'PREG_JIT_STACKLIMIT_ERROR' => PREG_JIT_STACKLIMIT_ERROR,
];

// 输出错误对照表
var_dump($error);

// 查看是否有匹配内容
var_dump($matches);
```

上面代码，最终执行失败，`preg_last_error()` 得到结果：6。解决方案：
* 要么删除中间部分内容
* 要么去除上面`ini_set('pcre.jit', 0);`的注释

## 总结：
1. `preg_match`等正则匹配是有各种限制的，最好使用 `preg_last_error` 跟踪一下错误。
2. `preg_match` 和 `preg_match_all` 返回匹配的次数(有可能为0)，但是失败也会返回`false`。因此需要注意返回值
3. 根本原因是：我写的正则效率太低了
    * 没有使用固定分组，导致回溯次数过多
    * 正则过长。正则过长就会匹配更多内容，如果不使用固定分组，那么就会回溯次数过多。
4. 如果可以，将一个复杂的正则简单化，甚至拆分为简单的正则

## 简单优化：
```
// 需要优化 因为回溯次数过多也会导致匹配失败的
$pattern = '/<span([^<\/>]*)>(([^<>]|<br\/>)*<span[^<\/>]*>([^<>]|<br\/>)*<\/span>([^<>]|<br\/>)*)+<\/span>/i';

// 简单优化：使用固定分组。
$pattern = '/<span([^\<\>]*)>(?>(?>[^\<\>]|<br\/>)*<span[^\<\>]*>(?>[^\<\>]|<br\/>)*<\/span>(?>[^\<\>]|<br\/>)*)+<\/span>/i';
```
[php中正则表达式详解](https://www.cnblogs.com/hellohell/p/5718319.html)

使用固定分组之后不仅速度快了，而且避免了正则匹配的限制而导致匹配失败