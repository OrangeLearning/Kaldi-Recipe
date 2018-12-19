# Kaldi训练单音素模型

> 利用`aishell` 进行单音素模型训练
>
> kaldi中训练声学模型，首先是训练单音素模型，即mono-phone过程。

## 1. 单音素模型原理



## 2. Kaldi 代码实现

在实际的过程中`run.sh`中的调用如下：

```bash
steps/train_mono.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/mono || exit 1;

# Monophone decoding
utils/mkgraph.sh data/lang_test exp/mono exp/mono/graph || exit 1;
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/mono/graph data/dev exp/mono/decode_dev
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/mono/graph data/test exp/mono/decode_test

# Get alignments from monophone system.
steps/align_si.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/mono exp/mono_ali || exit 1;
```



### 2.1 `train_mono.sh`

```bash
steps/train_mono.sh data/train data/lang exp/mono || exit 1;
```

注意后面的bash代码结果如下：

```bash
data=$1
lang=$2
dir=$3
```



这个训练流程如下：

1. gmm-init-mono 初始化单因素模型：输入语音特征，求全部特征的均值与方差，以此进行 GMM 参数的 初始化，再根据共享因素的情况建立树结构，存储于 tree 与 0.mdl；
2. 调用compile-train-graph，对于每句话，生成相对应的fst。其实就是将每句话的抄本展成音素级别的 状态转移图; 
3. 调用align-equal-compiled，进行初始对齐，生成对齐状态; 
4. 调用gmm-acc-stats-ali，读取对齐状态，计算更新GMM参数和HMM参数的统计量; 
5. 调用gmm-est，主要是将更新参数统计量去更新HMM的转移模型和GMM的三个参数; 





