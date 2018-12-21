#  数据准备 和 特征提取

> 利用`aishell`进行数据准备和特征提取

## 1. 数据准备

#### 1.1 数据下载`download_and_untar.sh` 
作用：下载解压数据
输入三个参数：
```bash
data=$1
url=$2
part=$3
```

```bash
#!/bin/bash

remove_archive=false
if [ "$1" == --remove-archive ]; then
  remove_archive=true
  shift
fi

# 判断参数个数
if [ $# -ne 3 ]; then
  echo "Usage: $0 [--remove-archive] <data-base> <url-base> <corpus-part>"
  echo "e.g.: $0 /export/a05/xna/data www.openslr.org/resources/33 data_aishell"
  echo "With --remove-archive it will remove the archive after successfully un-tarring it."
  echo "<corpus-part> can be one of: data_aishell, resource_aishell."
fi

data=$1
url=$2
part=$3

# 判断$data是否是目录
# 目录不存在直接
if [ ! -d "$data" ]; then
  echo "$0: no such directory $data"
  exit 1;
fi

# part_ok 判断这个part 是否正确
# 如果$part in (data_aishell resource_aishell) part_ok=true
# part_ok=false
part_ok=false
list="data_aishell resource_aishell"
for x in $list; do
  if [ "$part" == $x ]; then part_ok=true; fi
done

# 必须是(data_aishell resource_aishell)的其中一个
if ! $part_ok; then
  echo "$0: expected <corpus-part> to be one of $list, but got '$part'"
  exit 1;
fi


# url 不能为空
if [ -z "$url" ]; then
  echo "$0: empty URL base."
  exit 1;
fi

# 文件是否可读
if [ -f $data/$part/.complete ]; then
  echo "$0: data part $part was already successfully extracted, nothing to do."
  exit 0;
fi

# 后面基本上就是查看size以判断是否下载了
# sizes of the archive files in bytes.
sizes="15582913665 1246920"

if [ -f $data/$part.tgz ]; then
# 利用ls -l 这样第5个是 size,使用awk 进行输出
  size=$(/bin/ls -l $data/$part.tgz | awk '{print $5}')
  size_ok=false
  for s in $sizes; do if [ $s == $size ]; then size_ok=true; fi; done
  if ! $size_ok; then
    echo "$0: removing existing file $data/$part.tgz because its size in bytes $size"
    echo "does not equal the size of one of the archives."
    rm $data/$part.gz
  else
    echo "$data/$part.tgz exists and appears to be complete."
  fi
fi

if [ ! -f $data/$part.tgz ]; then
  if ! which wget >/dev/null; then
    echo "$0: wget is not installed."
    exit 1;
  fi
  full_url=$url/$part.tgz
  echo "$0: downloading data from $full_url.  This may take some time, please be patient."

# 利用wget 进行数据下载
  cd $data
  if ! wget --no-check-certificate $full_url; then
    echo "$0: error executing wget $full_url"
    exit 1;
  fi
fi

cd $data

# 解压缩
if ! tar -xvzf $part.tgz; then
  echo "$0: error un-tarring archive $data/$part.tgz"
  exit 1;
fi

# touch命令用于修改文件或者目录的时间属性，包括存取时间和更改时间。
# 若文件不存在，系统会建立一个新的文件。 
touch $data/$part/.complete

if [ $part == "data_aishell" ]; then
  cd $data/$part/wav
  for wav in ./*.tar.gz; do
    echo "Extracting wav from $wav"
    tar -zxf $wav && rm $wav
  done
fi

echo "$0: Successfully downloaded and un-tarred $data/$part.tgz"

if $remove_archive; then
  echo "$0: removing $data/$part.tgz file since --remove-archive option was supplied."
  rm $data/$part.tgz
fi

exit 0;

```

#### 1.2 准备字典`aishell_prepare_dict.sh`

> 这里可以看到，resource_aishell 中存放这lexicon



```bash
#!/bin/bash

# Copyright 2017 Xingyu Na
# Apache 2.0

# prepare dict resources

. ./path.sh

# 判断参数是否只有一个
[ $# != 1 ] && echo "Usage: $0 <resource-path>" && exit 1;


res_dir=$1
# 注意这个dict 原本存储的位置在 resource_aishell/ 文件夹之下的，直接从resource_aishell/
# 现在将这个lexicon.txt下载到 data/local/dict 
dict_dir=data/local/dict
mkdir -p $dict_dir
cp $res_dir/lexicon.txt $dict_dir

# 下面这段代码需要查看awk 和 perl 再进行操作
cat $dict_dir/lexicon.txt | awk '{ for(n=2;n<=NF;n++){ phones[$n] = 1; }} END{for (p in phones) print p;}'| \
  perl -e 'while(<>){ chomp($_); $phone = $_; next if ($phone eq "sil");
    m:^([^\d]+)(\d*)$: || die "Bad phone $_"; $q{$1} .= "$phone "; }
    foreach $l (values %q) {print "$l\n";}
  ' | sort -k1 > $dict_dir/nonsilence_phones.txt  || exit 1;

echo sil > $dict_dir/silence_phones.txt

echo sil > $dict_dir/optional_silence.txt

# No "extra questions" in the input to this setup, as we don't
# have stress or tone

cat $dict_dir/silence_phones.txt| awk '{printf("%s ", $1);} END{printf "\n";}' > $dict_dir/extra_questions.txt || exit 1;
cat $dict_dir/nonsilence_phones.txt | perl -e 'while(<>){ foreach $p (split(" ", $_)) {
  $p =~ m:^([^\d]+)(\d*)$: || die "Bad phone $_"; $q{$2} .= "$p "; } } foreach $l (values %q) {print "$l\n";}' \
 >> $dict_dir/extra_questions.txt || exit 1;

echo "$0: AISHELL dict preparation succeeded"
exit 0;
```



#### 1.3 准备数据`aishell_data_prep.sh`

传入两个参数，第一个参数是audio_shell ，第二个参数是transcript文件，这里应该是抄本。

```bash
#!/bin/bash

. ./path.sh || exit 1;

if [ $# != 2 ]; then
  echo "Usage: $0 <audio-path> <text-path>"
  echo " $0 /export/a05/xna/data/data_aishell/wav /export/a05/xna/data/data_aishell/transcript"
  exit 1;
fi

aishell_audio_dir=$1
aishell_text=$2/aishell_transcript_v0.8.txt

# 分dir,将数据存储到data/local/ 的情况
train_dir=data/local/train
dev_dir=data/local/dev
test_dir=data/local/test
tmp_dir=data/local/tmp

mkdir -p $train_dir
mkdir -p $dev_dir
mkdir -p $test_dir
mkdir -p $tmp_dir

# data directory check
if [ ! -d $aishell_audio_dir ] || [ ! -f $aishell_text ]; then
  echo "Error: $0 requires two directory arguments"
  exit 1;
fi

# find wav audio file for train, dev and test resp.
# 找到数据
find $aishell_audio_dir -iname "*.wav" > $tmp_dir/wav.flist
# 查看行数
n=`cat $tmp_dir/wav.flist | wc -l`
[ $n -ne 141925 ] && \
  echo Warning: expected 141925 data data files, found $n

grep -i "wav/train" $tmp_dir/wav.flist > $train_dir/wav.flist || exit 1;
grep -i "wav/dev" $tmp_dir/wav.flist > $dev_dir/wav.flist || exit 1;
grep -i "wav/test" $tmp_dir/wav.flist > $test_dir/wav.flist || exit 1;

rm -r $tmp_dir

# Transcriptions preparation
# 下面这些核心操作，注意到wav.flist是所有的wav
# 这里使用了filter_scp.pl
# 实现复杂度和写一个python 文件应该没有本质区别
# 这当中的utt2spk_to_spk2utt.pl等等操作其实本质没有变化
for dir in $train_dir $dev_dir $test_dir; do
  echo Preparing $dir transcriptions
  sed -e 's/\.wav//' $dir/wav.flist | awk -F '/' '{print $NF}' > $dir/utt.list
  sed -e 's/\.wav//' $dir/wav.flist | awk -F '/' '{i=NF-1;printf("%s %s\n",$NF,$i)}' > $dir/utt2spk_all
  paste -d' ' $dir/utt.list $dir/wav.flist > $dir/wav.scp_all
  utils/filter_scp.pl -f 1 $dir/utt.list $aishell_text > $dir/transcripts.txt
  awk '{print $1}' $dir/transcripts.txt > $dir/utt.list
  utils/filter_scp.pl -f 1 $dir/utt.list $dir/utt2spk_all | sort -u > $dir/utt2spk
  utils/filter_scp.pl -f 1 $dir/utt.list $dir/wav.scp_all | sort -u > $dir/wav.scp
  sort -u $dir/transcripts.txt > $dir/text
  utils/utt2spk_to_spk2utt.pl $dir/utt2spk > $dir/spk2utt
done

# 创建文件夹
mkdir -p data/train data/dev data/test

# 拷贝数据
for f in spk2utt utt2spk wav.scp text; do
  cp $train_dir/$f data/train/$f || exit 1;
  cp $dev_dir/$f data/dev/$f || exit 1;
  cp $test_dir/$f data/test/$f || exit 1;
done

echo "$0: AISHELL data preparation succeeded"
exit 0;
```

####  1.4 准备语言

`prepare_lang.sh`准备这个模型的时候还是使用了python



####  1.5 准备语言模型

`aishell_train_lms.sh`



## 2. 特征提取

在`run.sh`中脚本如下：

```bash
mfccdir=mfcc
for x in train dev test; do
  steps/make_mfcc_pitch.sh --cmd "$train_cmd" --nj 10 data/$x exp/make_mfcc/$x $mfccdir || exit 1;
  steps/compute_cmvn_stats.sh data/$x exp/make_mfcc/$x $mfccdir || exit 1;
  utils/fix_data_dir.sh data/$x || exit 1;
done
```



下面我们逐一来看这三个.sh文件的作用：

### 2.1`make_mfcc_pitch.sh`



### 2.2`compute_cmvn_stats.sh`

Compute cepstral mean and variance statistics per speaker.



###  2.3 `fix_data_dir.sh`

fix_data_dir.sh





> kaldi 的训练分为如下几个：
>
> 1. train_mono.sh 用来训练单音子隐马尔科夫模型，一共进行40次迭代，每两次迭代进行一次对齐操作
> 2. train_deltas.sh 用来训练与上下文相关的三音子模型
> 3. train_lda_mllt.sh 用来进行线性判别分析和最大似然线性转换
> 4. train_sat.sh 用来训练发音人自适应，基于特征空间最大似然线性回归
> 5. nnet3/run_dnn.sh 用nnet3来训练DNN，包括xent和MPE
> 6. 用chain训练DNN

下面我们一点一点看

