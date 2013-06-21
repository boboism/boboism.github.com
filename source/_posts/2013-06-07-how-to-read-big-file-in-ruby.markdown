---
layout: post
title: "如何使用Ruby来读大文件（日志分析）"
date: 2013-06-07 20:08
comments: true
categories: [Ruby]
---

最近需要读取应用的日志分析数据。打开其中一个节点的Log，发现已经差不多有2G了。我的妈呀～都长这么大了～
分析文件是件麻烦事，特别是这么大的日志。而且，如果不用程序员的方法还真不知道要看到猴年马月。这是，我想起了Ruby的[IO](http://ruby-doc.org/core-1.9.3/IO.html)类以及其子类[File](http://ruby-doc.org/core-1.9.3/File.html)。其中有几个方法：

1. [#read](http://ruby-doc.org/core-1.9.3/IO.html#method-i-read)
2. [#each](http://ruby-doc.org/core-1.9.3/IO.html#method-i-each)
3. [#readlines](http://ruby-doc.org/core-1.9.3/IO.html#method-i-readlines)

现在逐个介绍一下。

## read([length [, buffer]]) → string, buffer, or nil

这个函数官方文档上面是这样写的：
``` 
Reads length bytes from the I/O stream.
```
那也就是说，此斯比较适合用来读取二进制文件。OK，不太适合我。下一位。

## each(sep=$/) {|line| block } → ios
文档上说：
``` 
Executes the block for every line in ios, where lines are separated by sep. ios must be opened for reading or an IOError will be raised.
```
嘿～还可以将每行读进IO，那就非常的好，正好哥的文件比较大，这个很适合。再看看还有没有更好的。

## readlines(sep=$/) → array
文档上说：
``` 
Reads all of the lines in ios, and returns them in anArray. 
```
看来这个是可以按照数组的方式去操作文件行的，不错。但是，还有一个关键的东东，那就是把全部的文件都读到IO中，那就是说，假如我的文件有2G，但是我的系统中只有1G内存，那就有可能再读到一半的时候就挂掉了。

选来选去，还是用```#each```吧。

以下是分析：假设我现在有个案例，需要找到日志中所有请求IP的前10名，而我的文件每行是以IP打头的，因此，就会写成以下的示例：

``` ruby 案例
# 新建Hash，每个key的value初始化为0
ip_counter = Hash.new{|hash, key| hash[key] = 0}
# 打开文件
File.open("/Users/boboism/Downloads/access.log") do |f|
  # 读取每一行到内存中
  f.each("\n") do |line|
    # 获得IP，并且将IP的字符串转化成Symbol后作为key值，并且在计数器上＋1 
    line.scan(/^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}/).each{|ip| ip_counter[ip.to_sym] += 1}
  end
end
# 按照IP出现的先后，由多到少排序，并且获取前10名
top_ten = Hash[*ip_counter.sort_by{|key,value| value*-1}.take(10).flatten]
```

Total 12行就可以写一个简单的日志分析啦。回想起如果用Java的话，那是什么状况。不过题外话，这种还是可以用Linux下的Shell＋awk来完成。
