---
title: Google-Protobuf序列化反序列化原理探究
date: 2020-5-19 22:48:15
categories: 
- Unity
- ET框架学习笔记
tags :
- ET
- 源码剖析
---
<!-- more -->
# 前言
今天分析一波谷歌的protobuf，序列化反序列化原理，这篇文章主要是以C#语言为例，其他语言生成的代码算法基本是一样的，所以可以作为参考。

# 代码如下
序列化
```csharp
        /// <summary>
        /// Writes a 32 bit value as a varint. The fast route is taken when
        /// there's enough buffer space left to whizz through without checking
        /// for each byte; otherwise, we resort to calling WriteRawByte each time.
        /// </summary>
        internal void WriteRawVarint32(uint value)
        {
            // Optimize for the common case of a single byte value
            if (value < 128 && position < limit)
            {
                buffer[position++] = (byte)value;
                return;
            }

            while (value > 127 && position < limit)
            {
                buffer[position++] = (byte) ((value & 0x7F) | 0x80);
                value >>= 7;
            }
            while (value > 127)
            {
                WriteRawByte((byte) ((value & 0x7F) | 0x80));
                value >>= 7;
            }
            if (position < limit)
            {
                buffer[position++] = (byte) value;
            }
            else
            {
                WriteRawByte((byte) value);
            }
        }
```
反序列化
```csharp
        /// <summary>
        /// Reads a raw Varint from the stream.  If larger than 32 bits, discard the upper bits.
        /// This method is optimised for the case where we've got lots of data in the buffer.
        /// That means we can check the size just once, then just read directly from the buffer
        /// without constant rechecking of the buffer length.
        /// </summary>
        internal uint ReadRawVarint32()
        {
            if (bufferPos + 5 > bufferSize)
            {
                return SlowReadRawVarint32();
            }

            int tmp = buffer[bufferPos++];
            if (tmp < 128)
            {
                return (uint) tmp;
            }
            int result = tmp & 0x7f;
            if ((tmp = buffer[bufferPos++]) < 128)
            {
                result |= tmp << 7;
            }
            else
            {
                result |= (tmp & 0x7f) << 7;
                if ((tmp = buffer[bufferPos++]) < 128)
                {
                    result |= tmp << 14;
                }
                else
                {
                    result |= (tmp & 0x7f) << 14;
                    if ((tmp = buffer[bufferPos++]) < 128)
                    {
                        result |= tmp << 21;
                    }
                    else
                    {
                        result |= (tmp & 0x7f) << 21;
                        result |= (tmp = buffer[bufferPos++]) << 28;
                        if (tmp >= 128)
                        {
                            // Discard upper 32 bits.
                            // Note that this has to use ReadRawByte() as we only ensure we've
                            // got at least 5 bytes at the start of the method. This lets us
                            // use the fast path in more cases, and we rarely hit this section of code.
                            for (int i = 0; i < 5; i++)
                            {
                                if (ReadRawByte() < 128)
                                {
                                    return (uint) result;
                                }
                            }
                            throw InvalidProtocolBufferException.MalformedVarint();
                        }
                    }
                }
            }
            return (uint) result;
        }
```

# 算法解析
我们可以看到他是二进制转换，进行了很多或和与的运算，我们拿int32来举例子，它一般占4个字节，但是为了减小协议体，Google使用动态赋予空间，int32在proto中占1~5个字节，但是其实到不了5个因为最大就是全1，因为他给每个字节增加了一位的标记位所以即使是整型最大值也只需要32+4位就足够了，因为字节以8为单位所以最高是5个

下面是我用一个整性int32来实际演示序列化反序列化位运算的过程
```csharp
序列化:
第一步:
0000 0000 0000 0000 0000 1101 1111 1111 & ~0x7f
1111 1111 1111 1111 1111 1111 1000 0000
--------------------------------------------------
1111 1111 1111 1111 1111 1101 1000 0000 
//这一步判断是否大于128 大于则继续 不大于就直接读入1个字节存入字节数组

第二步:
0000 0000 0000 0000 0000 1101 1111 1111 & 0x7f
0000 0000 0000 0000 0000 0000 0111 1111
--------------------------------------------------
0000 0000 0000 0000 0000 0000 0111 1111 
//这一步是为了取得最后7位 将最高位制0

第三步:
0000 0000 0000 0000 0000 0000 0111 1111 | 0x80
0000 0000 0000 0000 0000 0000 1111 1111
--------------------------------------------------
0000 0000 0000 0000 0000 0000 1111 1111 
//这一步是为了给刚才取得的最后7位 补充一个最高位 使得成为一个字节 并存入字节数组中 最高位的意义是代表高位还有数据

第四步:
0000 0000 0000 0000 0000 1101 1111 1111 >> 7
--------------------------------------------------
//0000 000// 0000 0000 0000 0000 0000 1101 1
//右移7位 意义是将刚才读取的7位移除掉 继续读取高位信息

重复步骤一:
0000 0000 0000 0000 0000 0000 0001 1011 & ~0x7f
1111 1111 1111 1111 1111 1111 1000 0000
--------------------------------------------------
0000 0000 0000 0000 0000 0000 0000 0000
//这一步和第一步一样 判断是否大于128 发现不大于 所以直接读入一个字节

序列化过程完毕 结果为 Byte[2] = {11111111,00011011}

反序列化:(因为标记位的缘故，所以下面运算最大字节可能是5个所以补了8个0)
第一步：判断字节数组下标是否超过5，google.protobuf规定int只能占 1~5个字节 如果超过就执行SlowReadRawVarint32检查缓冲区溢出

第二部：声名tem记录当前读取的字节 tem = buffer[bufferPos++]从下标0开始 tem = 1111 1111
我们用上面得到的字节数组 先判断是否小于128 
小于=>说明这个int型只有不到1个字节所以最高位必然为0 所以说明前面没有数据了 直接将tem转换为uint类型返回
大于=>继续下一步

第三步:
0000 0000 0000 0000 0000 0000 0000 0000 1111 1111 & 0x7f
0000 0000 0000 0000 0000 0000 0000 0000 0111 1111
--------------------------------------------------------------
0000 0000 0000 0000 0000 0000 0000 0000 0111 1111
//这部目的将第一个字节取后7位 这里很容易理解 上面序列化的时候也是7为单位 最高位为一个标记代表前面还有数据
然后定义整性变量result接收这7位二进制 result = 0000 0000 0000 0000 0000 0000 0000 0000 0111 1111

第三步:
取出下一个字节 tem = buffer[bufferPos++] 判断tem是否小于 128 tem = 0001 1011
小于=>说明这个int型只有不到2个字节所以最高位必然为0 所以说明前面没有数据了 所以我们要将读取的第一个字节和第二个字节合并
0000 0000 0000 0000 0000 0000 0000 0000 0001 1011 <<7
--------------------------------------------------------------
0000 0000 0000 0000 0000 0000 0000 1 1011 0000 000
//为了合并要先将高位左移 因为先前读的是7位 所以要让出7位的位置

0000 0000 0000 0000 0000 0000 0000 1101 1000 0000 | result
0000 0000 0000 0000 0000 0000 0000 0000 0111 1111
--------------------------------------------------------------
0000 0000 0000 0000 0000 0000 0000 1101 1111 1111
//这一步通过和刚才的result 或运算 得到结果 并把结果赋值给 result

到了这一步我们已经将原来的数据还原出来

//如果大于继续执行第三步 随着字节变多 左移位数 为7的倍数 最多执行4次 
如果超过 则说明是 整型的最大数值 32位全是1 直接循环读取4个字节并返回result
还有可能循环读取结束 因为第五个字节最大就是0000 1111 如果if判断没生效则出现异常
最后 执行结束 返回result
```

# 结束语
这篇文章简单介绍了protobuf序列化/反序列化原理，但是protobuf的强大不只是这些，后面会有新的文章写protobuf的真正用法