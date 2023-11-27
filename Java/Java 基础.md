# Java 基础



## equals 与 == 的区别



- 值类型（`int`,`char`,`long`,`boolean` 等）的话
  - 都是用 == 判断相等性。
- 对象引用的话
  - == 判断引用所指的对象是否是同一个。
  - equals 方法，是 Object 的成员函数，有些类会覆盖(`override`) 这个方法，用于判断对象的等价性。



## 什么是自动拆装箱



自动装箱和拆箱，就是基本类型和引用类型之间的转换。



- `int` 是基本数据类型。
- Integer 是其包装类，注意是一个类。



1. 当数值范围为-128~127时：如果两个new出来Integer对象，即使值相同，通过“==”比较结果为false，但两个对象直接赋值，则通过“==”比较结果为“true，这一点与String非常相似。
2. 当数值不在-128~127时，无论通过哪种方式，即使两个对象的值相等，通过“==”比较，其结果为false；
3. 当一个Integer对象直接与一个int基本数据类型通过“==”比较，其结果与第一点相同；
4. Integer对象的hash值为数值本身；



**JVM会自动维护八种基本类型的常量池，int常量池中初始化-128~127的范围，所以当为Integer i=127时，在自动装箱过程中是取自常量池中的数值，而当Integer i=128时，128不在常量池范围内，所以在自动装箱过程中需new 128，所以地址不一样。**

查看Integer类源码，发现里面有一个私有的静态内部类IntegerCache，而如果直接将一个基本数据类型的值赋给Integer对象，则会发生自动装箱，其原理就是通过调用Integer类的public static Integer valueOf(将int类型的值包装到一个对象中 ，其部分源码如下：


```java

    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];
 
        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            }
            high = h;
 
            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);
        }
 
        private IntegerCache() {}
    }
  ..................
  ...................
  ...................
 

    public static Integer valueOf(int i) {
        assert IntegerCache.high >= 127;
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
 
```

