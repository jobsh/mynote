# 如何在Redis中使用lua

在redis里面使用lua脚本的主要命令

- eval + 要执行的lua语句：用来直接执行lua脚本
- script load + lua语句：将lua语句缓存到redis，返回一个id（返回脚本内容的SHA1校验和）
- evalsha ：执行脚本，脚本中如果有参数可以加参数
- script exist + 返回的lua脚本id : 判断 该lua脚本是否存在：存在返回1；不存在返回0
- script flush：清空redis缓存的脚本

## eval 命令

```lua
-- EVAL script numkeys key [key ...] arg [arg ...]

> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

numkeys：这里是2，代表我要传入两个key到lua脚本中

lua脚本中可以通过KEY[index], 从后面的传入的key1 key2取，KEY[1] 则代表取的是后面的第一个key，index从1开始

lua脚本中可以通过ARGV[index], 从后面的传入的参数中取，ARGV[1] 则代表取的是后面的第一个参数即first，同样index从1开始

## SCRIPT LOAD 命令

在用eval命令的时候，可以注意到每次都要把执行的脚本发送过去，这样势必会有一定的网络开销，所以redis对lua脚本做了缓存，通过script load 和 evalsha实现 script load命令会在redis服务器缓存你的lua脚本，并且返回脚本内容的SHA1校验和，然后通过evalsha 传递SHA1校验和来找到服务器缓存的脚本进行调用，这两个命令的格式以及使用方式如下

```lua
SCRIPT LOAD script
EVALSHA sha1 numkeys key [key ...] arg [arg ...]

redis> SCRIPT LOAD "return 'hello moto'"
"232fd51614574cf0867b83d384a5e898cfd24e5a"
```

> SHA1有如下特性：不可以从消息摘要中复原信息；两个不同的消息不会产生同样的消息摘要,(但会有1x10 ^ 48分之一的机率出现相同的消息摘要,一般使用时忽略)。

## 通过EVALSHA调用缓存中的lua脚本

```lua
redis> EVALSHA 232fd51614574cf0867b83d384a5e898cfd24e5a 0
"hello moto"
```

