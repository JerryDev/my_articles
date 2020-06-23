---
title: 怎样在JSON字符串中不转码Unicode字符
date: 2015-06-16 10:34:30
tags: php
---


对于JSON的编码和解码已经足够方便，几乎在各种编程语言之中都已经实现。在PHP之中，有json_encode 和 json_decode 函数实现。

但是有一个问题，json_encode 在编码时，会把中文字符编码为Unicode字符。

```php
$arr = array('test'=>'测试');
$str = json_encode($arr);
echo $str;
```
结果：

```json
{"test":"\u6d4b\u8bd5"}
```

<!--more-->

因此，先用urlencode编码字符串值，再json_encode编码成JSON，最后用urldecode解码。

```php
function string_urlencode(&$value){
    if(is_string($value)){
        $value = urlencode($value);
    }
}
$arr = array('test'=>'测试');
array_walk_recursive($arr, 'string_urlencode');
$str = json_encode($arr);
echo urldecode($str);
```
结果：

```json
{"test":"测试"}
```

那么问题来了，如果key/value 中的值有特殊字符，比如双引号，回车符，换行符等的时候，用上面这种方法转换出的字符串就不能正常解析为JSON字符串。

解决办法一：

```php
function string_urlencode(&$value){
    if(is_string($value)){
        $value = mysql_escape_string($value); // 转义
        $value = urlencode($value);
    }
}
$arr = array('test'=>'测"试'.PHP_EOL.'换行');
array_walk_recursive($arr, 'string_urlencode');
$str = json_encode($arr);
echo urldecode($str);
```

结果：

```json
{"test":"测\"试\r\n换行"}
```

这是我首先想到的解决办法。但是mysql_escape_string是MySQL的专用函数，不可靠，所以又想换成addslashes函数，却没有成功

```php
function string_urlencode(&$value){
    if(is_string($value)){
        $value = addslashes($value); // 错误
        $value = urlencode($value);
    }
}
$arr = array('test'=>'测"试'.PHP_EOL.'换行');
array_walk_recursive($arr, 'string_urlencode');
$str = json_encode($arr);
echo urldecode($str);
```

于是查了下php手册，addslashes只能转义 ' " \ null 四种字符

mysql_escape_string 转义的字符会更多，多了\r \n 和另一个Control-Z 字符，别问我Control-Z是什么，你可以参考[这里](http://www.cnblogs.com/suihui/archive/2012/09/20/2694751.html)

so， 上面的代码改一下

解决办法二：

```php
function string_urlencode(&$value){
    if(is_string($value)){
        $value = addslashes($value);
        $value = str_replace(
                    array(chr(13),chr(10)),
                    array('\r','\n'),
                    $value
                 );
        $value = urlencode($value);
    }
}
$arr = array('test'=>'测"试'.PHP_EOL.'换\行');
array_walk_recursive($arr, 'string_urlencode');
$str = json_encode($arr);
echo urldecode($str);
```

结果：

```json
{"test":"测\"试\r\n换\\行"}
```

当然，因为[JSON](http://json.org/json-zh.html)的规则是确定的，并且对于哪些字符需要转义也是确定的。
so，我们可以更极端一点，干脆不用addslashes函数，自己把所有的特殊字符转义了

解决办法三：

```php
function string_urlencode(&$value){
    if(is_string($value)){
        $value = str_replace(
                    array('\\','"','/',chr(8),chr(12),chr(13),chr(10),chr(9)),
                    array('\\\\','\"','\/','\b','\f','\r','\n','\t'),
                    $value
                 );
        $value = urlencode($value);
    }
}
$arr = array(
        'test'=>'测"试'.PHP_EOL.'换'.chr(8).'行'.chr(12).'制'.chr(9).'表'.'\\ha'
        );
array_walk_recursive($arr, 'string_urlencode');
$str = json_encode($arr);
echo urldecode($str);
```

结果：

```
{"test":"测\"试\r\n换\b行\f制\t表\\ha"}
```


解决办法四：
php5.4版本开始已经提供了原生的解决办法

```php
$arr = array(
        'test'=>'测"试'.PHP_EOL.'换'.chr(8).'行'.chr(12).'制'.chr(9).'表'.'\\ha'
        );
$str = json_encode($arr, JSON_UNESCAPED_UNICODE);
echo $str;
```

```
{"test":"测\"试\r\n换\b行\f制\t表\\ha"}
```


总结分析：
办法一： 方便实用，但是 mysql_escape_string 函数在未来php版本可能被移除（后记：在php7中已经移除）
办法二： 比较简单，够用
办法三： 强迫症+深究型比较喜欢的方式，很够显比格？
办法四： 部署环境必须要PHP的版本在5.4及以上

但是客观的说，如果环境允许，选择第四种，其次二三择其一，最次选择第一种。

End--

