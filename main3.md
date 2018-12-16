# Kaldi训练三音素模型

> 主要是kaldi 中的aishell 中的例子。注意aishell 中的`steps/` 和`utils/`主要是wsj例子中的，这里不做区分

## 1. 三音素模型原理

首先介绍一下三音素模型。

### 1.1 三音素模型





## 2. kaldi 实现

在kaldi 中，三音素模型这里的shell 脚本主要是`train_deltas.sh`，这个脚本的调用如下：

```bash
steps/train_deltas.sh --cmd "$train_cmd" \
 2500 20000 data/train data/lang exp/tri1_ali exp/tri2 || exit 1;
```

参数的意义在脚本中可以看到如下：

```bash
numleaves=$1 # 2500
totgauss=$2  # 20000
data=$3      # data/train
lang=$4 	 # data/lang
alidir=$5	 # exp/tri1_ali
dir=$6		 # exp/tri2
```

需要的数据是：

```bash
$alidir/final.mdl $alidir/ali.1.gz $data/feats.scp $lang/phones.txt
```



### 2.0 Decision tree internals 

1. `Event maps`




### 2.1 `acc-tree-stats.cc` 

> Accumulate tree statistics for decision tree training 
>
> 为了构建决策树累积相关的统计量。

在文件中，可以看到一个调用：

`acc-tree-stats 1.mdl scp:train.scp ark:1.ali 1.tacc`

可以看到这几个参数是什么意思:

```c++
model_filename = po.GetArg(1); # 声学模型
feature_rspecifier = po.GetArg(2); # 特征
alignment_rspecifier = po.GetArg(3); # 对齐
accs_out_wxfilename = po.GetOptArg(4);
```

首先看一下设计到的类：

1. AccumulateTreeStatsOptions

2. AccumulateTreeStatsInfo

3. TransitionModel

4. SequentialBaseFloatMatrixReader

5. RandomAccessInt32VectorReader

