# `make_mfcc.sh`代码分析

## 1. 参数

```bash
steps/make_mfcc.sh [options] <data-dir> [<log-dir> [<mfcc-dir>] ]
```

一般使用的参数如下：

* `--mfcc-config`: 这是compute-mfcc-feats 命令的参数文件，具体的我们后面的会分析
* `--nj`: 并行计算个数
* `--cmd`: 这个是并行计算脚本一般是`run.pl`或者`queue.pl`
* `--write-utt2num-frames` : 后面加true 和 false.



## 2. 数据准备要求



## 3. 使用的Kaldi工具

1. extract-segments
2. compute-mfcc-feats
3. copy-feats