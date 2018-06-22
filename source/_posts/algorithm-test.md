layout: post
title: "PHP实现的一些算法例子"
date: 2018-5-27 12:48
comments: true
tags: 
	- PHP
	- 算法
---

本片文章罗列了一些算法例子，后续也会不断增加。
- <a href="#custom_shuffle">随机打乱数组(PHP实现)</a>
- <a href="#get_prime">小于N的所有质数</a>
- <a href="#reverse">反转数组元素(PHP实现)</a>
- <a href="#find_common">两个有序序列相同元素</a>
- <a href="#fibonacci">时间复杂度为O(N)的斐波那契数列</a>
- <a href="#joseph_ring">约瑟夫环问题</a>
- <a href="#binary_search">二分查找</a>
- <a href="#get_min_abs_value">查找有序数组中绝对值最小元素的下标</a>
- <a href="#maxItemSum">最大子序列和</a>
- <a href="#relativePath">两个文件相对路径</a>

<!--more-->

## 1. <a name="custom_shuffle">随机打乱数组(PHP实现)</a>

```php
/**
 * 随机数
 * @param $arr
 * @return mixed
 */
function custom_shuffle($arr)
{
    $n = count($arr) - 1;
    for ($i = 0; $i <= $n; ++$i) {
        $rand_pos = mt_rand(0, $n);
        if ($rand_pos != $i) {
            $temp = $arr[$i];
            $arr[$i] = $arr[$rand_pos];
            $arr[$rand_pos] = $temp;
        }
    }
    return $arr;
}
```

## 2. <a name="get_prime">小于N的所有质数</a>

> 思路：基础情况是2是最小质数。质数是大于等于3的奇数，且不能被所有小于等于自身sqrt开方的质数整除
> 有点绕口，为什么是小于等于sqrt(N)的所有质数？假设能够N不是质数,则N可以表示为：N = x * y。
> 不妨假设x <= y。y = (N/x) >= x；得出 x <= sqrt(N)。

```php
/**
 * 请求小于$n的所有质数
 * 不能被任何根号N小的参数整除就是质数
 */
function get_prime($n)
{
    $prime = [2];
    if ($n < 2) {
        return [];
    }
    for($i = 3;$i<$n;$i+=2){
        $j = intval(sqrt($i));

        foreach($prime as $p){
            if ($p <= $j) {
                if ( $i % $p == 0 ) {
                    break 1;
                }
            } else {
                // 质数
                $prime[] = $i;
                break 1;
            }
        }
    }
    return $prime;
}
```

## 3. <a name="reverse">反转数组元素(PHP实现)</a>
> 思路：首尾交换

```php
function reverse($arr)
{
    $left = 0;
    $right = count($arr) - 1;

    while ($left < $right) {
        $tmp = $arr[$left];
        $arr[$left++] = $arr[$right];
        $arr[$right--] = $tmp;
    }

    return $arr;
}
```
## 4. <a name="find_common">两个有序序列相同元素</a>

```php
/**
 * 两个有序数组，求相同元素
 * @param $arr1
 * @param $arr2
 * @return array
 */
function find_common($arr1, $arr2)
{
    $i = $j = 0;
    $iEnd = count($arr1);
    $jEnd = count($arr2);
    $common = [];
    while (($i < $iEnd) && ($j < $jEnd)) {
        if ($arr1[$i] < $arr2[$j]) {
            ++$i;
        } elseif ($arr1[$i] > $arr2[$j]) {
            ++$j;
        } else {
            ++$i;
            ++$j;
            $common[] = $arr1[$i];
        }
    }
    return array_unique($common);
}
```

## 5. <a name="fibonacci">时间复杂度为O(N)的斐波那契数列</a>

```php
// 最简单的解法。但是复杂度是O(2^n)
function fibonacci1($n)
{
    if ($n < 3) {
        return 1;
    }
    return fibonacci1($n -1) + fibonacci1($n - 2);
}

/**
 * 生成器的斐波那契数列 时间复杂度为线性
 * @param $n
 * @return Generator
 */
function fibonacci($n)
{
    $n1 = $n2 = 1;
    for ($i = 1; $i <= $n; ++$i) {
        if ($i < 3) {
            yield 1;
        } else {
            $tmp = $n1;
            $n1 = $n2;
            $n2 += $tmp;
            yield $n2;
        }
    }
}
```
## 6. <a name="joseph_ring">约瑟夫环问题</a>

> 相关题目：一群猴子排成一圈，按1,2,…,n依次编号。然后从第1只开始数，数到第m只,把它踢出圈，从它后面再开始数，
> 再数到第m只，在把它踢出去…，如此不停的进行下去， 直到最后只剩下一只猴子为止，
> 那只猴子就叫做大王。要求编程模拟此过程，输入m、n, 输出最后那个大王的编号。

```php
// 这个是把整个过程都进行演变了 O(n*m)
function get_king_mokey($n, $m)
{
    $monkeys = range(1, $n);

    $i = 0;
    while (count($monkeys) > 1) {
        ++$i;
        $v = array_shift($monkeys);
        if ($i % $m) {
            array_push($monkeys, $v);
        }
    }

    return $monkeys[0];
}

// 参考：https://blog.csdn.net/k346k346/article/details/50992397
/**
 * 把下标演变的规律找出来即可 O(n)
 * 返回下标 从0开始
 * @param $n
 * @param $m
 * @return int
 */
function joseph_ring($n, $m)
{
    if ($n < 1 || $m < 1) {
        return -1;
    }
    $last = 0; // $n = 1时为0下标
    for ($i = 2; $i <= $n; ++$i) {
        $last = ($last + $m) % $i;
    }
    return $last;

}
```

## 7. <a name="binary_search">二分查找</a>
> 一个算法用常数时间把问题消减为原来的一部分(比如1/2)那么，算法就是O(logN)；
> 如果仅仅消减为常数数量，那么就是O(N)。

```php
function binary_search($array, $value)
{
    $left = 0;
    $right = count($array) -1;

    while ($left <= $right) {
        $mid = intval(($left + $right) / 2);
        if ($array[$mid] == $value) {
            return $mid;
        } elseif ($array[$mid] < $value) {
            $left = $mid + 1;
        } else {
            $right = $mid - 1;
        }
    }

    return -1;
}
```

## 8. <a name="get_min_abs_value">查找有序数组中绝对值最小元素的下标</a>
> 有序序列查找，使用二分查找？

```php
/**
 * 查找有序数组中绝对值最小的下标
 * 有序数组的查找--使用二分查找
 * @param $array
 */
function get_min_abs_value($array)
{
    $left = 0;
    $right = count($array) - 1;
    // 如果是同号
    if ($array[0] * $array[$right] >= 0) {
        return $array[0] >= 0 ? 0 : $right;
    }

    while ($left <= $right) {

        if ($left + 1 == $right) {
            return (abs($array[$left]) > abs($array[$right])) ? $right : $left;
        }

        // $mid 不会等于 $left，若相等 则 $left + 1 == $right 成立 早就退出了
        $mid = intval(($left + $right) / 2);
        if ($array[$mid] > 0) {
            $right = $mid;
        } elseif ($array[$mid] < 0) {
            $left = $mid;
        } else {
            return $mid;
        }
    }

    // 不会到这里
}
```

## 9. <a name="maxItemSum">最大子序列和</a>

```php
/**
 * 最大子序列和
 * @param $data
 * @return array
 */
function maxItemSum($data)
{
    $maxSum = $curSum = $curIndex = 0;
    $length = count($data);

    for ($curIndex = 0; $curIndex < $length; ++$curIndex) {
        $curSum += $data[$curIndex];
        if ($curSum < 0) {
            $curSum = 0;
        } elseif ($curSum > $maxSum) {
            $maxSum = $curSum;
        }
    }
    return $maxSum;
}
```

## 10. <a name="relativePath">两个文件相对路径</a>

```php
// 计算两个文件的相对路径
function relative_dir($path1, $path2)
{
  $path1 = explode('/', dirname($path1));
  $path2 = explode('/', dirname($path2));

  // 去掉相同的
  foreach ($path1 as $k => $dir) {
    if (!isset($path2[$k]) || ($dir != $path2[$k])) {
      break;
    }
    unset($path1[$k], $path2[$k]);
  }

  $str =  str_repeat('../', count($path2));

  return ($str . implode('/', $path1)) ?: './';

}

$path1 = '/a/b/d/c/name.pdf';
$path2 = '/a/b/c/df.pdf';

echo relative_dir($path1, $path2), "\n";
```