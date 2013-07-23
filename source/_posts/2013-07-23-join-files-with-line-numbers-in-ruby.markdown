---
layout: post
title: "用Ruby将源代码文件Join起来,并且加上行号"
date: 2013-07-23 16:12
comments: true
categories: [Ruby]
---

今天要弄个专利申请，需要提交相关的代码。写了个script，目的是将文件目录下的java文件全部弄在一起，并且写上行号。
最后看了看生成的txt文件大小，足足有2M多。总数7万多行。

代码如下：

``` ruby

BASE_PATH = "/Users/boboism/Projects/Java/Web/com.gacmotor.eam/"

File.open('/Users/boboism/Desktop/output.txt', 'w') do |output_file|

  Dir["#{BASE_PATH}**/**.java"].each do |file_path|
    File.open(file_path, 'r') do |file|
      output_file.write("/#{'*'*80}\r\n")
      output_file.write(" * File:   #{file_path.gsub(BASE_PATH, '')}\r\n")
      output_file.write(" * Author: Jianbo Su <sujb@gacmotor.com>\r\n")
      output_file.write(" #{'*'*80}/\r\n")
      file.each_with_index do |row, index|
        output_file.write("#{'%04d' % index} #{row}")
      end

      output_file.write("\r\n\r\n")
    end
  end

end


```