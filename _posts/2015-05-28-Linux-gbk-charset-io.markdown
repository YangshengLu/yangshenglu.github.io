---
layout: post
title: Linux挂载U盘中文乱码
date: "2015-01-23 20:42:43"
author: oopsoome
categories: Linux
---
在linux下面挂载U盘，遇到中文文件名总是乱码，从Linux复制
到U盘，有中文文件名的话，在Windows上面看又是乱码，这个
是因为编码引起的，Windows默认中文编码是GBK，而Linux一般是
UTF-8，之后发现这个可以通过自定义挂载参数来解决，如果U盘
对应的设备描述文件是`/dev/sdb`，要挂载到`/mnt`,可以使用命
令  
{% highlight bash %}
sudo mount -o rw,uid=$USER,iocharset=gbk /dev/sdb /mnt
{% endhighlight %}

中文文件名的问题不再是问题...
