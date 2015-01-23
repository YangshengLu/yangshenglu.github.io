---
layout: post
title:  "Android Test Not Found"
date:   2015-01-23 20:42:43
categories: Android
---
最近学习了一下单元测试，拿Android来试手，按照官方给的栗子有模有样的敲了一遍，跑测试的时候老是给我报一个Test not found的错误，Google了好久都没找到对的解决方法，最后才知道，所有的TestCase类都需要一个没有参数的构造器，而我继承的是ActivityInstrumentationTestCase2，一时手快按到了自动补全，他给我补全了一个这样的构造器  

    public MainActivityTest(Class<MainActivity> activityClass) {
      super(activityClass);
    }

被这个问题坑了一天也是醉了，看来要好好看看书  
修改成下面这个样子就好了

    public MainActivityTest() {
      super(MainActivity.class);
    }
