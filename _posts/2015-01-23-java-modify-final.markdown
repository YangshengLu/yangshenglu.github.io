---
layout: post
title: java修改final变量
date:   2015-01-23 20:42:43
author: YangshengLu
categories: Java
---
__不推荐在应用层的代码里面这么做,因为这里的实现逻辑是使用到了jdk实现的细节,不确保在所有的jvm上都可以生效__  
  
Java里面在用final修饰的变量的值是不能修改的（这里指的是这个引用的指向，其成员的值不在限制范围内），但是有时候我们的确想修改它的值，这个时候就需要一点小魔法了

#### Java中，有两个东西限制了对final变量的修改

#### 1.java编译器
他会在编译的时候检查，发现对final变量的修改就报错，让编译不通过。关于这一点，可以通过反射去绕开。

#### 2.运行时检查
如果手快的试了上面提到的通过反射修改final的方法，会发现还是不行。运行的时候会抛异常。__这是因为final这个修饰符会作为变量的成员所对应的Field对象的成员存在，可以查看Field类的源代码，有一个modifiers的成员__。但是既然是成员，那么就表示可以通过反射去修改这个成员的值，也就可以在运行时把final变量变成非final变量了。这样就可以修改它的值了

#### 代码栗子如下  
__请注意防止编译器优化引起的反射失败__
{% highlight java %}
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
public class ModifyFinal {
    // 下面这两种写法会被编译器优化掉，无法进行反射修改
    // private static final int constance = 1;
    // private static final String constString = "123"

    // 避免编译器优化
    private static int prevent_optimize() { return 0; }
    private static final int constance = prevent_optimize();
    // 如果你使用""等字符串常量，编译器会优化掉
    private static final String constString = String.valueOf("hello");

    public static void main(String[] args) {
        try {
            Field constanceField = ModifyFinal.class.getDeclaredField("constance");
            Field modifiersField = Field.class.getDeclaredField("modifiers");
            modifiersField.setAccessible(true);
            int curModifier = (Integer) modifiersField.get(constanceField);
            modifiersField.set(constanceField, ~Modifier.FINAL & curModifier);
            constanceField.setAccessible(true);

            constanceField.set(null, 1);
            // constance is 1
            System.out.println("constance is " + constance);

            constanceField = ModifyFinal.class.getDeclaredField("constString");
            curModifier = (Integer) modifiersField.get(constanceField);
            modifiersField.set(constanceField, ~Modifier.FINAL & curModifier);
            constanceField.setAccessible(true);

            constanceField.set(null, "world");
            // constString is world
            System.out.println("constString is " + constString);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
{% endhighlight %}
