# BloomFilter



布隆过滤器介绍：https://zhuanlan.zhihu.com/p/43263751



布隆过滤器的几个重要属性：误判率，数据个数，bit个数，哈希函数的个数

无论哪种实现，这四种属性都会表现出来。其中误判率，数据个数，bit个数三者最多只能指定两个，剩下的会自动计算出来。举个例子：不可以要求过滤器只使用1个bit在100个数据量下做到误判率为任意值，实际上bit位和数据总量确定了。误判率可以根据公式计算出一个定值。 可以操作一下链接中的过滤器感受一下。

布隆过滤器生成器：https://hur.st/bloomfilter/?n=1000000000&p=0.03&m=&k=









## Guava BloomFilter

`com.google.common.hash.BloomFilter<T>` 不同版本的Guava实现有很大差异，因此类上被打上了@Beta的标记表示此API没被固定住后续版本可能改变甚至删除。

构建BloomFilter由一些重载的静态方法BloomFilter.create()完成。

完整的参数列表是：

​	`com.google.common.hash.Funnel` :accept:  用来将入参转化为字节数组，内部利用 java.nio.ByteBuffer使用起来很简单，通常会以一下形式实现它。

```java
enum PersonFunnel implements Funnel<Person> { 
    
    INS;    
    
    @Override    
    public void funnel(Person from, PrimitiveSink into) {     
        into.putInt(from.getAge());  
    }
}
```

​	`int expectedInsertions` 一个int型的数字，表示期望插入的item的数量，暗示不可超过21亿。

​	`double fpp` 误判率  [0,1]

​	`com.google.common.hash.BloomFilter.Strategy` 插入策略，定义了BloomFilter内部的BitMap的存储格式以及一个对象被转化为一个Hash值之后，具体会被以怎么样的策略放入BitMap。此参数用户无法指定，有个默认值：

com.google.common.hash.BloomFilterStrategies #MURMUR128_MITZ_64

由于`MURMUR128_MITZ_64` 内部涉及了从hash值到最终设置到过滤器的全部过程，所以有必要说一下这个过程。

定义的BitMap格式如下：

```java
// Note: We use this instead of java.util.BitSet because we need access to the long[] data field
static final class BitArray {  
    final long[] data;  
    long bitCount;  
    
    BitArray(long bits) { //过滤器的总bit
        //一个long有64bit，总数据量除以64进1就能知道long数组的长度了
        this(new long[Ints.checkedCast(LongMath.divide(bits, 64,RoundingMode.CEILING))]); 
    }  // Used by serialization 
    
    BitArray(long[] data) {//构造器
        checkArgument(data.length > 0, "data length is zero!"); 
        this.data = data;  
        long bitCount = 0;  
        for (long value : data) {  
            bitCount += Long.bitCount(value); 
        }  
        this.bitCount = bitCount; 
    }
    
    //省略很多操作数据的方法
}
```



1.利用com.google.common.hash.HashFunction#hashObject计算Object的Hash值结合Funnel得到byte[16]

2.byte[16]拆分成两个8字节的用long刚好装下，得到两个long：  hash1   hash2
4.操作Hash值，代码如下：

```java
long combinedHash = hash1;
for (int i = 0; i < numHashFunctions; i++) {  
    // Make the combined hash positive and indexable 
    bitsChanged |= bits.set((combinedHash & Long.MAX_VALUE) % bitSize);  
    combinedHash += hash2;
}
```

可以看出，这里的Hash函数的个数，并没有对Object求n次Hash而是利用这个for模拟求了n次Hash。这样做也行吧。

5.(combinedHash & Long.MAX_VALUE) 截取hash值低64位。然后将Hash值转化为一个比总bit位数小的数字以便放得下。

6.将上一步得到的数字设置在BitMap上：

```java
boolean set(long index) { 
    if (!get(index)) {  
        data[(int) (index >>> 6)] |= (1L << index);    
        bitCount++;   
        return true; 
    }  
    return false;
}
```

get(index)找到BitMap上是否存在这条数据，如果不存在进入if设置上。

index>>>6：去掉index右边6bit   ---> 转为int去掉左边32-6 bit 得到一个数组元素long。

1L << index 可以设置一个bit在上一步得到的long上。



这些操作看上去很复杂，本质上还是在做散列操作。把Hash值散列成bitmap上的某个bit然后放上去。这些操作看上去很繁琐，只是为了散列的更随机而已。







## Redis BloomFilter

前置知识，[Redis Module](https://redis.io/topics/modules-intro#redis-modules-an-introduction-to-the-api)：Redis通过暴露自己核心API让用户可以自定义命令，函数甚至新的数据类型。开源社区利用此功能贡献了很多Modules，Bloom就是其中一个。

装载完成后使用BD.ADD newFilter foo 向过滤器中添加一项，如果不存在过滤器会创建一个。

使用BF.EXISTS newFilter foo 使用过滤器判断值是否存在。







完结！撒花！🌸 





