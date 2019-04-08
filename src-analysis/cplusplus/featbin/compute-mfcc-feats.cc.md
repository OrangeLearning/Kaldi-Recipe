# `compute-mfcc-feats.cc`代码分析

> featbin/compute-mfcc-feats.cc： 用于计算mfcc特征。
>
> mfcc特征的原理请参见我的其他文件

## 1. 文件结构

是一个直接的可执行文件，不含其他的类和方法的定义。

代码执行流程如下：





## 2. 细节分析

### 2.1 引用类

1. MfccOptions

    参见feat/feature-mfcc.h 的定义

2. SequentialTableReader

3. WaveHolder

4. BaseFloatMatrixWriter

5. TableWriter

6. HtkMatrixHolder

7. RandomAccessBaseFloatReaderMapped

    