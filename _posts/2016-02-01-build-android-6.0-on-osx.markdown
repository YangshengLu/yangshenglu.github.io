---
layout: post
title:  "在OSX上下载和编译Android6.0源代码"
date:   2016-02-01
categories: Android
---
最近事情想对少, 也是想提升一下自己对Android的认识, 打算认真阅读一下Android系统的源代码. 在下载代码和编译的过程中, 遇到了一些问题, 这里记录下来, 以备不时之需.  
这里大部分的知识来源于[谷歌android官网](http://source.android.com/)  

## 准备工作

### 创建大小写敏感的文件系统镜像  
其实觉得OSX挺奇葩的, 作为一个unix类系统, 默认的文件系统居然不区分文件名大小写. 不知道是历史包袱还是故意而为之. Android并不支持在大小写不敏感的文件系统上编译, 因此我们需要创建一个大小写敏感的文件系统的镜像文件  
{% highlight bash %}
hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 60g ~/android.dmg
{% endhighlight %}  
如果在使用的过程中, 发现分配给镜像文件的空间不够用了, 还以使用下面的命令修改镜像文件的容量  
{% highlight bash %}
hdiutil resize -size <new-size-you-want>g ~/android.dmg.sparseimage
{% endhighlight %}  
执行了`hdiutil create`命令之后, 应该就可以在home目录中发现一个android.img(或者android.img.sparse), 直接双击就可以挂载这个这个镜像了, 默认的挂载路径应该是`/Volumes/untitled`

### 安装JDK
JDK的安装应该是安卓开发者非常熟悉的了, 如果你不熟悉, 那么你还是先不要直接看Android源码吧. 饭要一口一口吃(´・ω・`)  
需要注意的是, Android 6.0系统需要JDK 1.8.x, Android 5.0系统需要JDK 1.7.x

### 安装Homebrew和其他依赖
其实homebrew不是编译Android必须安装的, 但是这个包管理工具可以使安装其他的库和工具变得极其简单  

* 安装homebrew  
{% highlight bash %}
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
{% endhighlight %}
 

* 安装git, openssl, gnupg
{% highlight bash %}
brew install git openssl gnupg
{% endhighlight %}

* 安装curl  
由于内置的curl和homebrew安装的curl都是基于SecureTransport的, Android 6.0的编译需要使用到基于openssl的curl, 因此需要从源代码编译安装, 如果你对此不太熟悉, 可以直接尝试下面的脚本, 老司机请忽略.  
{% highlight bash %}
#!/bin/bash
curl http://www.execve.net/curl/curl-7.47.0.zip -o curl-7.47.0.zip
unzip curl-7.47.0.zip
OPEN_SSL_DIR=`ls -1 -d /usr/local/Cellar/openssl/* | tail -n 1`
./configure --prefix=/usr/local --with-ssl=$OPEN_SSL_DIR
make -j 4 && make install
{% endhighlight %}

* 设置PATH环境变量
{% highlight bash %}
export JAVA_HOME=`/usr/libexec/java_home -v '1.8.*'`
export PATH=$JAVA_HOME/bin:/usr/local/bin:$PATH
ulimit -S -n 1024
{% endhighlight %}

## 下载, 编译, 运行

* 下载repo
{% highlight bash %}
curl https://storage.googleapis.com/git-repo-downloads/repo > /usr/local/bin/repo
chmod a+x /usr/local/bin/repo
{% endhighlight %}

* 同步代码到本地
{% highlight bash %}
cd /Volumes/untitled
mkdir AndroidSources
cd AndroidSources
git config --global user.name "Your Name" 
git config --global user.email "you@example.com" # 这里必须是注册google账号的邮箱
repo init -u https://android.googlesource.com/platform/manifest
repo sync
{% endhighlight %}

* 编译源代码
{% highlight bash %}
source build/envsetup.sh
lunch aosp_arm-eng
make -j4 # 可以根据自己的CPU核心数量增加, 推荐进程数量为cpu数量+1
{% endhighlight %}
如果你之前的所有步骤都没有错, 那么你编译结束的时候, 应该看到了Success类似的关键字

* 使用模拟器运行编译的android系统
{% highlight bash %}
emulator
{% endhighlight %}