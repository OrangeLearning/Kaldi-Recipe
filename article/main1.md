# Kaldi 中的 mfcc 提取

## 1. 脚本

我们以wsj脚本为例，首先在`run.sh`脚本中调用`make_mfcc.sh`：

```shell
steps/make_mfcc.sh --mfcc-config conf/mfcc.conf --nj 40 --cmd "$train_cmd" \
  data/train exp/make_mfcc $mfccdir
```



根据`make_mfcc.sh`的脚本中，可以看到，数据取自data/train，而log在exp/make_mfcc中，而最终的mfcc特征的结果在mfccdir中。

随后，检查了一下是否已有了最终的结果feats.scp,是否拥有wav.scp和mfcc_config这几个核心文件。

下面调用了validate_data_dir.sh 用于查看数据的合法性。





## 2. 代码

> 根据上面的提示，一共有三个kaldi源码工具需要查看：`extract-segment` , `compute-mfcc-feats`和`copy-feats`三个脚本进行计算

