---
layout: post
title: "如何使用Anemone来爬视频地址"
date: 2013-05-23 20:06
comments: true
categories: [Ruby, Anemone, Crawler, Spider]
---
刚刚翻回之前写的一些爬虫脚本，想分享一下其中一个比较有意思的爬虫。
这个爬虫脚本使用的是[Chris Kite](http://www.chriskite.com)写的[Anemone](http://anemone.rubyforge.org/)（一个Ruby的爬虫库）。它提供了非常简单的DSL用来爬取每一个页面以及其URLs，官方说会自动计算出其需要的最短路径。而且，这个爬虫库是多线程。由于Ruby正则表达式的写法简洁，因此，看上去非常简短。

主要用到的gem有anemone，digest，还有用来保存链接的mongo

``` ruby
gem install anemone
gem install mongo
```

首先，会定义一些全局变量,其中ENTRY_PATTERN是入口，PAGE_PATTERN是要爬的页面，ANY_PATTERN还会包含另外一些需要爬的链接：

``` ruby
ENTRY_PATTERN = "http://www.oabt.org/?cid=5"
PAGE_PATTERN  = %r[cid=(?:5|25|6|7|8|11)(?:&page=\d+)?$]
ANY_PATTERN   = PAGE_PATTERN
```
接着新建一个MONGODB以及表，因为是要爬[http://www.oabt.org](http://www.oabt.org)，所以就直接命名了：

``` ruby
db = Mongo::Connection.new.db("oabt_org")
movies = db["movie"]
```

接着，是定义Anemone的一些options：

``` ruby
options = {
  :threads              => 1,                # 线程数
  :verbose              => true,             # 详细显示 
  :discard_page_bodies  => true,             
  :user_agent           => "Mozilla...",   
  :delay                => 0,
  :obey_robots_txt      => true,
  :depth_limit          => 1,
  :redirect_limit       => 5,
  :storage              => nil,
  :cookies              => nil,
  :accept_cookies       => true,
  :skip_query_strings   => false,
  :proxy_host           => nil,
  :proxy_port           => false,
  :read_timeout         => 20
}
```



``` ruby 官方代码中的default options https://github.com/chriskite/anemone/blob/master/lib/anemone/core.rb
    DEFAULT_OPTS = {
      # run 4 Tentacle threads to fetch pages
      :threads => 4,
      # disable verbose output
      :verbose => false,
      # don't throw away the page response body after scanning it for links
      :discard_page_bodies => false,
      # identify self as Anemone/VERSION
      :user_agent => "Anemone/#{Anemone::VERSION}",
      # no delay between requests
      :delay => 0,
      # don't obey the robots exclusion protocol
      :obey_robots_txt => false,
      # by default, don't limit the depth of the crawl
      :depth_limit => false,
      # number of times HTTP redirects will be followed
      :redirect_limit => 5,
      # storage engine defaults to Hash in +process_options+ if none specified
      :storage => nil,
      # Hash of cookie name => value to send with HTTP requests
      :cookies => nil,
      # accept cookies from the server and send them back?
      :accept_cookies => false,
      # skip any link with a query string? e.g. http://foo.com/?u=user
      :skip_query_strings => false,
      # proxy server hostname 
      :proxy_host => nil,
      # proxy server port number
      :proxy_port => false,
      # HTTP read timeout in seconds
      :read_timeout => nil
    }
```

好了，接着就是开始爬了,#focus_crawl中会定义需要爬的URL，只会保留ANY_PATTERN的URL，接着，如果符合PAGE_PATTERN的，会在其页面中找他的title，id，reference url，还有就是他的下载链接：magnet／thunder／ed2k（这里会使用到NOKIGIRI，因为我熟悉css selector，所以代码中直接使用css selecor来抓），接着就会检查他的引用页的hash值是否已经存在，如果不存在就直接插入mongo：

``` ruby
Anemone.crawl(ENTRY_PATTERN, options) do |anemone|

  anemone.focus_crawl do |page|
    p "focus #{page.url}"
    page.links.keep_if{|link| link.to_s =~ ANY_PATTERN}
  end

  anemone.on_pages_like(PAGE_PATTERN) do |page|
    if page.doc
      p "crawl #{page.url}"
      p "crawl header:#{page.headers}"
      p "crawl code:#{page.code}"
      p "crawl body:#{page.body}"
      p "crawl links:#{(page.links||[]).collect(&:to_s).join('\n')}"
      page.doc.css('tr').each do |tr|
        p "crawl tr"
        title, id, ref_url = tr.css('td.name.magTitle a').collect{|a| [a.text, a['rel'], a['href']]}.first
        if id && (md5 = Digest::MD5.hexdigest(id)) && (movies.find({"md5" => md5}).first.nil?)
          cat = tr.css("a.sbule").collect{|a| [a['href'][/\d+/], a.text]}.first.join('|')
          ed2k_url, mag_url, thunder_url = (tr.css('td.dow a.ed2kDown').first||{})['ed2k'], (tr.css('td.dow a.magDown').first||{})['href'], (tr.css('td.dow a.thunder').first||{})['thunderhref']
          movie = {:md5 => md5, :cat => cat, :ref_url => ref_url, :title => title, :ed2k_url => ed2k_url, :mag_url => mag_url, :thunder_url => thunder_url}
          p "Inserting #{movie.inspect}"
          movies.insert movie
        end
      end
    end
  end

end
```


全部的代码如下，总共不到60行的代码：

``` ruby
require 'anemone'
require 'digest/md5'
require 'mongo'

# Patterns
ENTRY_PATTERN = "http://www.oabt.org/?cid=5"
PAGE_PATTERN  = %r[cid=(?:5|25|6|7|8|11)(?:&page=\d+)?$]
ANY_PATTERN   = PAGE_PATTERN

db = Mongo::Connection.new.db("oabt_org")
movies = db["movie"]

options = {
  :threads              => 1,
  :verbose              => true,
  :discard_page_bodies  => true,
  :user_agent           => "Mozilla...",
  :delay                => 0,
  :obey_robots_txt      => true,
  :depth_limit          => 1,
  :redirect_limit       => 5,
  :storage              => nil,
  :cookies              => nil,
  :accept_cookies       => true,
  :skip_query_strings   => false,
  :proxy_host           => nil,
  :proxy_port           => false,
  :read_timeout         => 20
}
p "begin"
Anemone.crawl(ENTRY_PATTERN, options) do |anemone|

  anemone.focus_crawl do |page|
    p "focus #{page.url}"
    page.links.keep_if{|link| link.to_s =~ ANY_PATTERN}
  end

  anemone.on_pages_like(PAGE_PATTERN) do |page|
    if page.doc
      p "crawl #{page.url}"
      p "crawl header:#{page.headers}"
      p "crawl code:#{page.code}"
      p "crawl body:#{page.body}"
      p "crawl links:#{(page.links||[]).collect(&:to_s).join('\n')}"
      page.doc.css('tr').each do |tr|
        p "crawl tr"
        title, id, ref_url = tr.css('td.name.magTitle a').collect{|a| [a.text, a['rel'], a['href']]}.first
        if id && (md5 = Digest::MD5.hexdigest(id)) && (movies.find({"md5" => md5}).first.nil?)
          cat = tr.css("a.sbule").collect{|a| [a['href'][/\d+/], a.text]}.first.join('|')
          ed2k_url, mag_url, thunder_url = (tr.css('td.dow a.ed2kDown').first||{})['ed2k'], (tr.css('td.dow a.magDown').first||{})['href'], (tr.css('td.dow a.thunder').first||{})['thunderhref']
          movie = {:md5 => md5, :cat => cat, :ref_url => ref_url, :title => title, :ed2k_url => ed2k_url, :mag_url => mag_url, :thunder_url => thunder_url}
          p "Inserting #{movie.inspect}"
          movies.insert movie
        end
      end
    end
  end

end
```