---
layout: post
title:  MessagePack规格说明书-中文翻译
date:   2022-03-21 10:50:00 +0800
categories: 网络
tags: 翻译
topping: true
---


MessagePack 像 JSON 一样，是一个对象序列化规范。  

MessagePack 有两个概念：类型系统和格式。  

序列化就是通过 MessagePack 类型系统把应用程序对象转化成 MessagePack 格式的过程。  

反序列化就是通过 MessagePack 类型系统把 MessagePack 格式转化成应用程序对象的过程。  

>序列化：  
>&nbsp;&nbsp;&nbsp;&nbsp;应用程序对象  
>&nbsp;&nbsp;&nbsp;&nbsp;-->　MessagePack 类型系统  
>&nbsp;&nbsp;&nbsp;&nbsp;-->　MessagePack 格式(字节数组)  
>反序列化：  
>&nbsp;&nbsp;&nbsp;&nbsp;MessagePack 格式(字节数组)  
>&nbsp;&nbsp;&nbsp;&nbsp;-->　MessagePack 类型系统  
>&nbsp;&nbsp;&nbsp;&nbsp;-->　应用程序对象  

本文档描述了 MessagePack 类型系统，MessagePack 格式和它们的转换过程。  

### 目录
• MessagePack 序列化  
　　[类型系统](#TypeSystem)  
　　　　[限制](#typeLimit)  
　　　　[拓展类型](#tExtensional)  
　　[格式](#Formats)  
　　　　[概述](#fOverview)  
　　　　[图表中的符号](#fNotation)  
　　　　[nil 格式](#fNil)  
　　　　[bool 格式族](#fBool)  
　　　　[int 格式族](#fInt)  
　　　　[float 格式族](#fFloat)  
　　　　[str 格式族](#fStr)  
　　　　[bin 格式族](#fBin)  
　　　　[array 格式族](#fArray)  
　　　　[map 格式族](#fMap)  
　　　　[ext 格式族](#fExt)  
　　　　[Timestamp 拓展类型](#fTime)  
　　[序列化：类型向格式转换](#Serialization)  
　　[反序列化：格式向类型转换](#Deserialization)  
　　[对未来的探讨](#Future)  
　　　　[Profile](#fProfile)  
　　[实现准则](#Implementation)  
　　　　[升级 MessagePack 规范](#iUpgrade)  
　　　
　　　
### 类型系统    {#TypeSystem}

 - 类型  
     + Integer 表示一个整数  
     + Nil 表示 nil  
     + Boolean 表示 true 或 false  
     + Float 表示 IEEE754 标准的一个双精度浮点型数包括 NaN 和 Infinity  
     + Raw  
        * String 扩展 Raw 类型表示一个 UTF-8 字符串  
        * Binary 扩展 Raw 类型表示一个字节数组  
     + Array 表示一连串对象  
     + Map 表示键值对的对象  
     + Extension 表示一个类型信息和一个字节数组的元组，其中类型信息是一个由应用或 MessagePack 规范定义其含义的整数  
        * Timestamp 表示在世界时间线上独立于时区和日期的瞬间点．最大精度是纳秒  

#### 限制    {#typeLimit}

 - 一个 Integer 对象的值限制在 -(2^63) 到 (2^64)-1 之间。  
 - 一个 Binary 对象的最大长度是 (2^32)-1。  
 - 一个 String 对象的最大字节数是 (2^32)-1。  
 - String 对象可能包含无效字节序列，并且反序列化器的行为取决于它收到无效字节序列时的具体实现。  
     - 反序列化串成必须提供获取原始字节数组的功能以方便应用程序可以决定怎样处理对象。  
 - 一个 Array 对象的最大元素数目是 (2^32)-1。  
 - 一个 Map 对象的最大键值对数目是 (2^32)-1。  

 
#### 拓展类型    {#tExtensional}

MessagePack 允许应用程序使用 Extension 类型定义特定应用类型。Extension 类型包括一个整数和一个字节数组，其中表示一种类型，字节数组表示数据。  
　　
应用程序可以分配 0 到 127 来存储特定应用程序类型信息。比如，应用程序可以定义 type=0 来当做应用程序唯一类型系统，并且在负载中保存类型名称和类型值。  
　　
MessagePack 保留了 -1 到 -128 做为未来拓展来添加预定义类型。这些类型将被添加用来交换更多类型，这样就不用在跨编程环境中使用预共享的静态类型模式了。  

```
[0, 127]: 特定应用程序类型
[-128, -1]: 保留用于预定义类型
```

因为 Extension 类型是计划被添加的，老的应用程序可能并没有实现它们的所有内容。然而它们仍然可以被当做一种 Extension 类型被处理。因此，应用程序可以自行决定它们是否拒绝未知 Extension 类型，当未知数据来接收，或者不管它们的负载直接传输给另一个应用程序。  

这里有一个预定义 Extension 类型的列表。类型的格式被定义在<格式>章节中。  

|Name |Type |
|-----|-----|
|Timestamp |-1 |  


### 格式    {#Formats}
#### 概述    {#fOverview}

|格式名称|第一字节 (二进制)|第一字节 (十六进制)|
|-------|--------------|-----------------|
|positive fixint |0xxxxxxx |0x00 - 0x7f |
|fixmap |1000xxxx |0x80 - 0x8f |
|fixarray |1001xxxx |0x90 - 0x9f |
|fixstr |101xxxxx |0xa0 - 0xbf |
|nil |11000000 |0xc0 |
|(未被使用) |11000001 |0xc1 |
|false |11000010 |0xc2 |
|true |11000011 |0xc3 |
|bin 8 |11000100 |0xc4 |
|bin 16 |11000101 |0xc5 |
|bin 32 |11000110 |0xc6 |
|ext 8 |11000111 |0xc7 |
|ext 16 |11001000 |0xc8 |
|ext 32 |11001001 |0xc9 |
|float 32 |11001010 |0xca |
|float 64 |11001011 |0xcb |
|uint 8 |11001100 |0xcc |
|uint 16 |11001101 |0xcd |
|uint 32 |11001110 |0xce |
|uint 64 |11001111 |0xcf |
|int 8 |11010000 |0xd0 |
|int 16 |11010001 |0xd1 |
|int 32 |11010010 |0xd2 |
|int 64 |11010011 |0xd3 |
|fixext 1 |11010100 |0xd4 |
|fixext 2 |11010101 |0xd5 |
|fixext 4 |11010110 |0xd6 |
|fixext 8 |11010111 |0xd7 |
|fixext 16 |11011000 |0xd8 |
|str 8 |11011001 |0xd9 |
|str 16 |11011010 |0xda |
|str 32 |11011011 |0xdb |
|array 16 |11011100 |0xdc |
|array 32 |11011101 |0xdd |
|map 16 |11011110 |0xde |
|map 32 |11011111 |0xdf |
|negative fixint |111xxxxx |0xe0 - 0xff |  

(译注：带 fix 类型与不带 fix 类型区别在于类型长度，存储方式，见各类型族)

#### 图表中的符号    {#fNotation}

```
一个字节:
+--------+
|        |
+--------+
多个字节:
+========+
|        |
+========+
以MessagePack格式存储的多个对象:
+~~~~~~~~~~~~~~~~~+
|                 |
+~~~~~~~~~~~~~~~~~+
```
X,Y,Z和A 是可以被实际的比特代替的符号。  

#### Nil 格式    {#fNil}
Nil 格式用一个字节来存储 Nil
```
nil:
+--------+
|  0xc0  |
+--------+
```

#### bool 格式族    {#fBool}
Bool 格式族用一个字节存储 false 或 true  
```
false:
+--------+
|  0xc2  |
+--------+
true:
+--------+
|  0xc3  |
+--------+
```

#### int 格式族    {#fInt}
Int 格式族用 1,2,3,5或9 个字节存储一个整数  
```
positive fixint 存储 7 位正整数
+--------+
|0XXXXXXX|
+--------+

negative fixint 存储 5 位负整数
+--------+
|111YYYYY|
+--------+

* 0XXXXXXX 是 8 位无符号整数
* 111YYYYY 是 8 位有符号整数

uint 8 存储一个 8 位无符号整数
+--------+--------+
|  0xcc  |ZZZZZZZZ|
+--------+--------+

uint 16 存储一个 16 位大端无符号整数
+--------+--------+--------+
|  0xcd  |ZZZZZZZZ|ZZZZZZZZ|
+--------+--------+--------+

uint 32 存储一个 32 位大端无符号整数
+--------+--------+--------+--------+--------+
|  0xce  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
+--------+--------+--------+--------+--------+

uint 64 存储一个 64 位大端无符号整数
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|  0xcf  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
+--------+--------+--------+--------+--------+--------+--------+--------+--------+

int 8 存储一个 8 位有符号整数
+--------+--------+
|  0xd0  |ZZZZZZZZ|
+--------+--------+


int 16 存储一个 16 位大端有符号整数
+--------+--------+--------+
|  0xd1  |ZZZZZZZZ|ZZZZZZZZ|
+--------+--------+--------+

int 32 存储一个 32 位大端有符号整数
+--------+--------+--------+--------+--------+
|  0xd2  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
+--------+--------+--------+--------+--------+

int 64 存储一个 64 位大端有符号整数
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|  0xd3  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
```

#### float 格式族    {#fFloat}

Float 格式族用 5 个字节或 9 个字节存储一个浮点型数。  
```
float 32 以 IEEE754 单精度浮点数格式存储了一个浮点型数：
+--------+--------+--------+--------+--------+
|  0xca  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
+--------+--------+--------+--------+--------+

float 64 以 IEEE754 双精度浮点数格式存储了一个浮点型数：
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|  0xcb  |YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|
+--------+--------+--------+--------+--------+--------+--------+--------+--------+

其中
* XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX 是一个大端的 IEEE754 单精度浮点型数。
  从单精度到双精度的精度扩展不会丢失精度。
* YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY 是一个大端的 IEEE754 双精度浮点型数。
```

#### str 格式族    {#fStr}
Str 格式族格式族存储字节数组，除了存储字节数组外,还有 1,2,3或5 字节的额外字节来表示字节数组。  
```
fixstr 存储长度最大 31 个字节的一个字节数组：
+--------+========+
|101XXXXX|  data  |
+--------+========+

str 8 存储长度最大 (2^8)-1 个字节的一个字节数组：
+--------+--------+========+
|  0xd9  |YYYYYYYY|  data  |
+--------+--------+========+

str 16 存储长度最大 (2^16)-1 个字节的一个字节数组：
+--------+--------+--------+========+
|  0xda  |ZZZZZZZZ|ZZZZZZZZ|  data  |
+--------+--------+--------+========+

str 32 存储长度最大 (2^32)-1 个字节的一个字节数组：
+--------+--------+--------+--------+--------+========+
|  0xdb  |AAAAAAAA|AAAAAAAA|AAAAAAAA|AAAAAAAA|  data  |
+--------+--------+--------+--------+--------+========+

其中
* XXXXX 是一个表示 N 的 5 位的无符号整数
* YYYYYYYY 是一个表示 N 的 8 位的无符号整数
* ZZZZZZZZ_ZZZZZZZZ 是一个表示 N 的 16 位大端无符号整数
* AAAAAAAA_AAAAAAAA_AAAAAAAA_AAAAAAAA 是一个表示 N 的 32 位大端无符号整数
* N 是data的长度
```

#### bin 格式族    {#fBin}

Bin 格式族存储字节数组，除了存储字节数组外，还有 2,3或5 字节的额外字节来表示字节数组。  
```
bin 8 存储长度最大 (2^8)-1 个字节的一个字节数组：
+--------+--------+========+
|  0xc4  |XXXXXXXX|  data  |
+--------+--------+========+

bin 16 存储长度最大 (2^16)-1 个字节的一个字节数组：
+--------+--------+--------+========+
|  0xc5  |YYYYYYYY|YYYYYYYY|  data  |
+--------+--------+--------+========+

bin 32 存储长度最大 (2^32)-1 个字节的一个字节数组：
+--------+--------+--------+--------+--------+========+
|  0xc6  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|  data  |
+--------+--------+--------+--------+--------+========+

其中
* XXXXXXXX 是一个表示 N 的 8 位无符号整数
* YYYYYYYY_YYYYYYYY 是一个表示 N 的 16 位大端无符号整数
* ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ 是一个表示 N 的 32 位大端无符号整数
* N 是data的长度
```

#### array 格式族    {#fArray}

Array 格式族存储一系列元素，除了元素外还有 1,3或5 字节的额外数据来表示元素数目。  
```
fixarray 存储最多 15 个元素的一个数组：
+--------+~~~~~~~~~~~~~~~~~+
|1001XXXX|    N objects    |
+--------+~~~~~~~~~~~~~~~~~+

array 16 存储最多 (2^16)-1 个元素的一个数组：
+--------+--------+--------+~~~~~~~~~~~~~~~~~+
|  0xdc  |YYYYYYYY|YYYYYYYY|    N objects    |
+--------+--------+--------+~~~~~~~~~~~~~~~~~+

array 32 存储最多 (2^32)-1 个元素的一个数组：
+--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+
|  0xdd  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|    N objects    |
+--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+

其中
* XXXX 是一个表示 N 的 4 位无符号整数
* YYYYYYYY_YYYYYYYY 是一个表示 N 的 16 位大端无符号整数
* ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ 是一个表示 N 的 32 位大端无符号整数
* N 是数组大小
```

#### map 格式族    {#fMap}
Map 格式族存储键值对，除了键值对外还有 1,3或5 字节额外数据表示键值对数目。  
```
fixmap 存储最多 15 个元素的一个映射：
+--------+~~~~~~~~~~~~~~~~~+
|1000XXXX|   N*2 objects   |
+--------+~~~~~~~~~~~~~~~~~+

map 16 存储最多 (2^16)-1 个元素的一个映射：
+--------+--------+--------+~~~~~~~~~~~~~~~~~+
|  0xde  |YYYYYYYY|YYYYYYYY|   N*2 objects   |
+--------+--------+--------+~~~~~~~~~~~~~~~~~+

map 32 存储最多 (2^32)-1 个元素的一个映射：
+--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+
|  0xdf  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|   N*2 objects   |
+--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+

where
* XXXX 是一个表示 N 的 4 位无符号整数
* YYYYYYYY_YYYYYYYY 是一个表示 N 的 16 位大端无符号整数
* ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ 是一个表示 N 的 32 位大端无符号整数
* N 是映射的大小
* 对象中的奇数元素是映射的键
* 键的下一个元素是它对应的值
```

#### ext 格式族    {#fExt}
Ext 格式族保存由一个整数和一个字节数组组成的一个元组。  
```
fixext 1 存储一个整数和长度为 1 字节的字节数组：
+--------+--------+--------+
|  0xd4  |  type  |  data  |
+--------+--------+--------+

fixext 2 存储一个整数和长度为 2 字节的字节数组：
+--------+--------+--------+--------+
|  0xd5  |  type  |       data      |
+--------+--------+--------+--------+

fixext 4 存储一个整数和长度为 4 字节的字节数组：
+--------+--------+--------+--------+--------+--------+
|  0xd6  |  type  |                data               |
+--------+--------+--------+--------+--------+--------+

fixext 8 存储一个整数和长度为 8 字节的字节数组：
+--------+--------+--------+--------+--------+--------+
|  0xd7  |  type  |                  data
+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+
               data                 |
+--------+--------+--------+--------+


fixext 16 存储一个整数和长度为 16 字节的字节数组：
+--------+--------+--------+--------+--------+--------+
|  0xd8  |  type  |               data                                  
+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+
                    data (cont.)
+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+
                    data (cont.)                      |
+--------+--------+--------+--------+--------+--------+

ext 8 存储一个整数和最大长度可以到 (2^8)-1 字节的字节数组：
+--------+--------+--------+========+
|  0xc7  |XXXXXXXX|  type  |  data  |
+--------+--------+--------+========+

ext 16 存储一个整数和最大长度可以到 (2^16)-1 字节的字节数组：
+--------+--------+--------+--------+========+
|  0xc8  |YYYYYYYY|YYYYYYYY|  type  |  data  |
+--------+--------+--------+--------+========+

ext 32 存储一个整数和最大长度可以到 (2^32)-1 字节的字节数组：
+--------+--------+--------+--------+--------+--------+========+
|  0xc9  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|  type  |  data  |
+--------+--------+--------+--------+--------+--------+========+

其中
* XXXXXXXX 是一个表示 N 的 8 位无符号整数
* YYYYYYYY_YYYYYYYY 是一个表示 N 的 16 位大端无符号整数
* ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ 是一个表示 N 的 32 位大端无符号整数
* N 是 data 的长度
* type 是一个 8 位有符号整数
* type < 0 保留用做未来拓展，包括 2 字节类型信息
```

#### Timestamp extension type    {#fTime}

Timestamp 拓展类型被分配了拓展类型的 -1。它定义了三种格式：32 位格式，64 位格式，和96 位格式。  
```
timestamp 32 存储了以 32 位无符号整数表示的从 1970-01-01 00:00:00 UTC开始过去的秒数：
+--------+--------+--------+--------+--------+--------+
|  0xd6  |   -1   |   seconds in 32-bit unsigned int  |
+--------+--------+--------+--------+--------+--------+

timestamp 64 存储了以 32 位无符号整数表示的从 1970-01-01 00:00:00 UTC开始过去的秒数和纳秒数：
+--------+--------+--------+--------+--------+------|-+--------+--------+--------+--------+
|  0xd7  |   -1   | nanosec. in 30-bit unsigned int |   seconds in 34-bit unsigned int    |
+--------+--------+--------+--------+--------+------^-+--------+--------+--------+--------+
(译注：此处秒数用了 34 位，纳秒用了 30 位)

timestamp 96 存储了分别以 64 位有符号整数和 32 位无符号整数表示的从 1970-01-01 00:00:00 UTC 开始过去的秒数和纳秒数：
+--------+--------+--------+--------+--------+--------+--------+
|  0xc7  |   12   |   -1   |nanoseconds in 32-bit unsigned int |
+--------+--------+--------+--------+--------+--------+--------+
+--------+--------+--------+--------+--------+--------+--------+--------+
                    seconds in 64-bit signed int                        |
+--------+--------+--------+--------+--------+--------+--------+--------+
```
 - Timestamp32 格式可以表示在 [1970-01-01 00:00:00 UTC, 2106-02-07 06:28:16 UTC) 区间的一个时间戳．纳秒部分是 0。  
 - Timestamp64 格式可以表示在 [1970-01-01 00:00:00.000000000 UTC, 2514-05-30 01:53:04.000000000 UTC) 区间的一个时间戳。  
 - Timestamp96 格式可以表示在 [-292277022657-01-27 08:29:52 UTC, 292277026596-12-04 15:30:08.000000000 UTC) 区间的一个时间戳。  
 - 在 timestamp64 和 timestamp96 格式中，纳秒不能大于 999999999。  

序列化的伪代码：  
```
struct timespec {
    long tv_sec;  // seconds
    long tv_nsec; // nanoseconds
} time;
if ((time.tv_sec >> 34) == 0) {
    uint64_t data64 = (time.tv_nsec << 34) | time.tv_sec;
    if (data64 & 0xffffffff00000000L == 0) {
        // timestamp 32
        uint32_t data32 = data64;
        serialize(0xd6, -1, data32)
    }
    else {
        // timestamp 64
        serialize(0xd7, -1, data64)
    }
}
else {
    // timestamp 96
    serialize(0xc7, 12, -1, time.tv_nsec, time.tv_sec)
}
```

反序列化的伪代码：  
```
 ExtensionValue value = deserialize_ext_type();
 struct timespec result;
 switch(value.length) {
 case 4:
     uint32_t data32 = value.payload;
     result.tv_nsec = 0;
     result.tv_sec = data32;
 case 8:
     uint64_t data64 = value.payload;
     result.tv_nsec = data64 >> 34;
     result.tv_sec = data64 & 0x00000003ffffffffL;
 case 12:
     uint32_t data32 = value.payload;
     uint64_t data64 = value.payload + 4;
     result.tv_nsec = data32;
     result.tv_sec = data64;
 default:
     // error
 }
```

### 序列化：类型到格式的转换    {#Serialization}
MessagePack 序列化器按以下方式将 MessagePack 类型转换成格式：

|源类型 |输出格式 |
|------|--------|
|整数 |int格式族 (positive fixint, negative fixint, int 8/16/32/64 or uint 8/16/32/64) |
|Nil |nil |
|布尔值 |bool 格式族(false or true) |
|浮点数 |float 格式族 (float 32/64) |
|字符串 |str 格式族 (fixstr or str 8/16/32) |
|二进制 |bin 格式族 (bin 8/16/32) |
|数组 |array 格式族 (fixarray or array 16/32) |
|映射 |map 格式族 (fixmap or map 16/32) |
|拓展 |ext 格式族 (fixext or ext 8/16/32) |  

如果对象可以被多种输出格式来表示，则序列化器需要使用占用字节数最小的格式来表示数据。  


### 反序列化器：格式到类型的转换    {#Deserialization}
MessagePack 反序列化器按以下方式将 MessagePack 格式转换成类型：  

|源格式 |输出类型 |
|------|--------|
|positive fixint, negative fixint, int 8/16/32/64 and uint 8/16/32/64 |整数 |
|nil |Nil |
|false and true |布尔值 |
|float 32/64 |浮点数 |
|fixstr and str 8/16/32 |字符串 |
|bin 8/16/32 |二进制 |
|fixarray and array 16/32 |数组 |
|fixmap map 16/32 |映射 |
|fixext and ext 8/16/32 |拓展 |


### 对未来的探讨    {#Future}
#### Profile    {#fProfile}
Profile 是这样一种思想：应用程序限制 MessagePack 的语义，同时共享相同的语法以使 MessagePack 适应特定的用例。  

比如，应用程序可以移除二进制类型，限制映射的键都是字符串类型，同时设置一些限制使语义与 JSON 兼容。使用 schema 的应用程序可以字符串和二进制类型，把字节数组当原始类型来处理，使用 hash(digest) 序列化数据的应用程序可以对映射的键值进制排序以使序列化的数据更有确定性。  


### 实现准则    {#Implementation}
#### 升级MessagePack规范    {#iUpgrade}
MessagePack 规范现在已经被修改过了．下面是升级现有 MessagePack 实现的指导准则：  

 - 在少数发行版中，反序列化器支持 bin 格式族和 str8 格式，反序列化器对象的类型可以同 raw16(==str16) 或 raw32(==str32) 类型相同。  
 - 在大多发行版中，序列化器使用 bin 格式族和 str 格式族来区分二进制类型和字符串类型  
     + 同时，序列化器需要提供不能使用 bin 格式族和 str8 格式的兼容模式。  


---
MessagePack 规范说明书  
最后修改于2017-08-09 22:42:07 -0700  
Sadayuki Furuhashi © 2013-04-21 21:52:33 -0700  

<br />
---
**参考**  
原文链接：[MessagePack specification](https://github.com/msgpack/msgpack/blob/master/spec.md)  
翻译工具：  
 - [有道翻译](https://fanyi.youdao.com)  
 - [Goldendict](http://goldendict.org)  


