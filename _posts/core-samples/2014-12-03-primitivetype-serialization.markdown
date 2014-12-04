---
title: 基本类型序列化/反序列化
layout: post-my
guid: urn:uuid:ejsdf5w0dzflu6sg07javuskpqozz2uw
description: "基本类型序列化/反序列化"
tags:
  - 基本类型 序列化
---

在工作中写rpc框架时涉及到了序列化/反序列化的处理，一直计划写序列化/反序列化相关的文章，但一直没有具体整理过，本篇算开山之作吧，序列化和反序列化涉及的东西比较多，需要支持常见数据类型的序列化/反序列化，包括基本类型、对象、集合等。开源项目有很多比如：Gaea Serializer,kryo,fast-serialization,protobuf,thrift等
本系列文章主要介绍一些业界常用的方式，以及一些开源项目的处理方式。
本文主要说明基本类型的一些序列化/反序列化方式，以及优化策略。

**名称解释：**

序列化：把基本类型装换成2进制
反序列化：把2进制反序列化为基本类型

**说明：**

原理尽可能的屏蔽语言特征，demo以java语言编写。

**基础普及：**

| 数据类型  | 大小(位) |   默认值|
| :-------- | --------:| :------:|
| byte      |         8|    0    |
| short     |        16|    0    |
| int       |        32|    0    |
| long      |        64|    0    |
| float     |        32|    0    |
| double    |        64|    0    |
| char      |        16| '\u0000'|
| boolean   |         1| flase  |

**一、定长方式**

根据不同数据类型所占字节数进行序列化。

**1.1、原理**

byte     1字节    1序列化后为：00000001(byte为[1])

short    2字节    1序列化后为：00000000，00000001(byte为[0,1])

int      4字节    1序列化后为：00000000，00000000，00000000，00000001(byte为[0,0,0,1])

long     8字节    1序列化后为：00000000，00000000，00000000，00000000，
                                                00000000，00000000，00000000，00000001(byte为[0,0,0,0,0,0,0,1])

float    4字节    1序列化后为：与int相同

double   4字节    1序列化后为：与long相同

char     2字节    方式同short

boolean  1字节    true用1代替，false用0代替

**1.2、Demo**

**1.2.1、序列化**

**int**

public static byte[] GetBytesFromInt32(int value) {
         
    byte[] buffer = new byte[4];

    for (int i = 0; i < 4; i++) {
        
         buffer[i] = (byte) (value >> (8 * i));
     
    }

    return buffer;

}

     
public static byte[] writeInt(int value) {
         
    byte[] buffer = new byte[4];
    
    buffer[0] = (byte) value;
    
    buffer[1] = (byte) (value >> 8);
          
    buffer[2] = (byte) (value >> 16);
          
    buffer[3] = (byte) (value >> 24);
    
    return buffer;
  
}

**char**

public static byte[] GetBytesFromChar(char ch) {
          
    int temp = (int) ch;
          
    byte[] b = new byte[2];
          
    for (int i = b.length - 1; i > -1; i--) {
              
        // 将最高位保存在最低位
               
        b[i] = new Integer(temp & 0xff).byteValue();
    
        temp = temp >> 8;

    }
    
    return b;

}

    
public static byte[] writeChar(char value) {

    byte[] buffer = new byte[2];
    
    buffer[0] = (byte) (value >>> 8);
    
    buffer[1] = (byte) value;

    return buffer;

}


**1.2.2、反序列化**

**int**


public static int readInt(byte[] buffer) {

    int position = 0;

    return ((buffer[position] & 0xFF)
                    | (buffer[position + 1] & 0xFF) << 8
                    | (buffer[position + 2] & 0xFF) << 16
                    | (buffer[position + 3] & 0xFF) << 24);

 }


**char**

public static char readChar(byte[] buffer) {

    int position = 0;
    
    return (char) (((buffer[position++] & 0xFF) << 8) | (buffer[position++] & 0xFF));

}


别的数据类型方式类似，就不一一举例，有兴趣可以看看下面开源项目具体实现。

**二、变长方式**

**2.1、原理**

有很多情况如果要序列化一个基本类型，但该类型完全可以拿一个更小的字节的基本类型替代，比如：int类型1，如果按照方式1序列化后为[0,0,0,1] 而这个其实完全可以用一个byte替代[1],剩余的3各字节是在浪费空间(只拿int举例，别的基本类型处理方式类似)。这里需要注意一点，序列化后得byte数组要知道当前序列化后的数据是否启动了变量长度处理，更不能因为启动了变量长度而引入额外的处理。

**2.2、demo**

本demo以kryo int处理举例，其他类型原理类似

**2.2.1、序列化**

public static byte[] writeVarInt(int value)  {

    byte buffer[] = new byte[1];
    
    int position = 0;

    if (value >>> 7 == 0) {
        
         buffer[position++] = (byte) value;
        
         return buffer;
    }
          
    if (value >>> 14 == 0) {
               
        buffer = Arrays.copyOf(buffer, 2);
               
        buffer[position++] = (byte) ((value & 0x7F) | 0x80);
               
        buffer[position++] = (byte) (value >>> 7);
               
        return buffer;
          
    }
    
    if (value >>> 21 == 0) {
               
        buffer = Arrays.copyOf(buffer, 3);
               
        buffer[position++] = (byte) ((value & 0x7F) | 0x80);
               
        buffer[position++] = (byte) (value >>> 7 | 0x80);
               
        buffer[position++] = (byte) (value >>> 14);
               
        return buffer;
          
    }
          
    if (value >>> 28 == 0) {
               
        buffer = Arrays.copyOf(buffer, 4);
               
        buffer[position++] = (byte) ((value & 0x7F) | 0x80);
               
        buffer[position++] = (byte) (value >>> 7 | 0x80);
               
        buffer[position++] = (byte) (value >>> 14 | 0x80);
               
        buffer[position++] = (byte) (value >>> 21);
               
        return buffer;
          
    }
          
    buffer = Arrays.copyOf(buffer, 5);
          
    buffer[position++] = (byte) ((value & 0x7F) | 0x80);
          
    buffer[position++] = (byte) (value >>> 7 | 0x80);
          
    buffer[position++] = (byte) (value >>> 14 | 0x80);
          
    buffer[position++] = (byte) (value >>> 21 | 0x80);
          
    buffer[position++] = (byte) (value >>> 28);
          
    return buffer;
     
}
    
**2.2.2、反序列化**


public static int readVarInt(byte[] buff)  {
          
    int position = 0;
          
    int b = buff[position++];
          
    int result = b & 0x7F;
          
    if ((b & 0x80) != 0) {
               
        byte[] buffer = buff;
               
        b = buffer[position++];
               
        result |= (b & 0x7F) << 7;
               
        if ((b & 0x80) != 0) {
                    
            b = buffer[position++];
                    
            result |= (b & 0x7F) << 14;
                    
            if ((b & 0x80) != 0) {
                        
                b = buffer[position++];
                         
                result |= (b & 0x7F) << 21;
                         
                if ((b & 0x80) != 0) {
                              
                    b = buffer[position++];
                              
                    result |= (b & 0x7F) << 28;
                         
                }
                   
            }
        
        }
          
    }
          
    return true ? result : ((result >>> 1) ^ -(result & 1));
    
 }


**说明：**

上面处理方式Int类型最多可能会生成5个字节，为什么会多出一个字节，主要是为了考虑反序列化时判断当前数据类型是否启动了变量长度处理机制，通过1个字节(00000000)的最高位进行的判断，而本身数字对应字节最高位数据放到第2字节处理。可能有人会说最多会生成5个字节比原来还会多出一个字节，没有节省反而增加了，这点需要我们在项目中权衡，如果大部分数据是小于4个字节的，那么这种方式还是比较好的，可以节省点空间，如果项目中数字比较大那使用方式1即可。

fst处理机制跟这个稍微有点区别。

1、会判断传入数据是否一个字节可以表示，如果可以直接写一个字节

2、是否2个字节可以表示，如果可以，先写入-128(10000000)，然后在写入2个字节，第一个字节-128在反序列化时判断是否为2个字节

3、如果不满足1、2情况，先写入-127(10000001),然后在写入4个字节，第一个字节在反序列化时做判断处理

上面的两种处理方式略有差别，但有一点是相同的，如果要变量长度处理，需要有一个标志位标记区分是几个字节。

kryo只提供了int long的处理机制，fst基本类型都提供了，但char、short生成的字节数组会由3位组成，最高位是标志位(标记是否启动变长)，反而增加了长度处理。float、int处理方式一样 long double处理方式一样。byte和boolean用一个字节表示。

**总结：**

在编写序列化通用框架时，建议增加版本号字段，后期如果序列化方式发生变化可以通过版本号做对应的处理，这点很重要，开源项目Gaea这块做得比较好。

后续会介绍别的数据类型序列化/反序列化处理机制，敬请期待。

**参考资料**

1、[Gaea](https://github.com/xinbinhao/Gaea/blob/master/serializer/src/com/bj58/spat/gaea/serializer/component/helper/ByteHelper.java)

2、kryo

[https://github.com/xinbinhao/kryo/blob/master/src/com/esotericsoftware/kryo/io/Output.java](https://github.com/xinbinhao/kryo/blob/master/src/com/esotericsoftware/kryo/io/Output.java)

[https://github.com/xinbinhao/kryo/blob/master/src/com/esotericsoftware/kryo/io/Input.java](https://github.com/xinbinhao/kryo/blob/master/src/com/esotericsoftware/kryo/io/Input.java)

3、fst

[https://github.com/xinbinhao/fast-serialization/blob/master/src/main/java/org/nustaq/serialization/FSTStreamEncoder.java](https://github.com/xinbinhao/fast-serialization/blob/master/src/main/java/org/nustaq/serialization/FSTStreamEncoder.java)

[https://github.com/xinbinhao/fast-serialization/blob/master/src/main/java/org/nustaq/serialization/FSTStreamDecoder.java](https://github.com/xinbinhao/fast-serialization/blob/master/src/main/java/org/nustaq/serialization/FSTStreamDecoder.java)

