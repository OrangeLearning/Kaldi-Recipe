# Kaldi 学习笔记

## 0. Kaldi 基础知识
0. 本质问题
   kaldi 只是一个工作包，但是这个工具包的使用比较复杂，因为设计到对于训练的数据格式、运行环境等等进行调整。所以才需要shell 和 perl 进行辅助。

>  我自己正准备撰写一个工具用于简化kaldi的使用。命名为kaldi-Juice

Kaldi 几个重要特征：

* Context-dependent LVCSR system (arbitrary phonetic-context width)

* FST-based training and decoding (we use OpenFst)

* Maximum Likelihood training 

* Working on lattice generation + DT.

* All kinds of linear and affine transforms

* Example scripts demonstrate VTLN, SAT, etc.



Kaldi 所引用的模块：

* Matrix library
* OpenFst and fstext
* Kaldi I/O
* Tree building and clustering code
* HMM and transition modeling
* Decoding-graph creation
* Gaussian Mixture Models
* Linear transform code
* Decoders
* Feature processing
* Command-line tools




1. path.sh 的作用
```bash
export KALDI_ROOT=`pwd`/../../..
[ -f $KALDI_ROOT/tools/env.sh ] && . $KALDI_ROOT/tools/env.sh
export PATH=$PWD/utils/:$KALDI_ROOT/tools/openfst/bin:$PWD:$PATH
[ ! -f $KALDI_ROOT/tools/config/common_path.sh ] && echo >&2 "The standard file $KALDI_ROOT/tools/config/common_path.sh is not present -> Exit!" && exit 1
. $KALDI_ROOT/tools/config/common_path.sh
export LC_ALL=C

# 添加kaldi主目录路径 
# 如果存在`env.sh`文件，则执行该脚本 
# 添加`openfst`执行文件等目录路径 
# 如果不存在`common_path.sh`文件，则打印报错，退出 执行
# 存在，则执行该脚本文件 
# 按照C排序法则
```

2. `tools/`和`src/`文件夹的作用
  `toos/`:
* The most important subdirectory is the one for OpenFst.

3. `src/`:
    储存源码

4. `cmd.sh` 的作用
    用于选择是本机运行还是使用其他运算机器