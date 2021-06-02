# js按照指定长度为数字前面补零

# 背景

在工作中，如果我们输出的数字长度是固定的，假设为4，如果数字为3，则输出0003，不够位数就在之前补足0，这里总结提供了三种不同的方式实现JS代码给数字补0 的操作;

# 方法1

```js
function pad(num, n) {
    var i = (num + "").length;
    while(i++ < n) num = "0" + num;
    return num;
}
```

# 方法2

```js
function PrefixInteger(num, length) {
    return (num/Math.pow(10,length)).toFixed(length).substr(2);
}
```

# 方法3（更为高效，推荐）

```js
function PrefixInteger(num, length) {
    return ( "0000000000000000" + num ).substr( -length );
}
```

# 方法4（更加高效，简洁）

```js
function PrefixInteger(num, length) {
   return (Array(length).join('0') + num).slice(-length);
}
```

# 方法5（神奇递归法）

```js
function pad2(num, n) {
    if ((num + "").length >= n)
        return num;
    return pad2("0" + num, n);
}
```

# 参考文献

JQuery按照指定长度为数字前面补零 [https://blog.csdn.net/fengqingtao2008/article/details/49796725?utm_source=blogxgw](https://blog.csdn.net/fengqingtao2008/article/details/49796725?utm_source=blogxgwz4)

