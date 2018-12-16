#  利用`aishell`  进行特征提取

## 1. `run.sh`代码

> 首先cmd.sh 主要是为了给出两个参数，这一点不重要我们下面继续
> path.sh 主要是为了加载kaldi 的命令
>
> 关键在于run.sh:

```bash
#!/bin/bash
# 首先定义两个data的变量
data=/export/a05/xna/data
data_url=www.openslr.org/resources/33

# 运行cmd.sh shell
# 这样就加载了三个变量：
# export train_cmd="queue.pl --mem 2G"
# export decode_cmd="queue.pl --mem 4G"
# export mkgraph_cmd="queue.pl --mem 8G"
. ./cmd.sh

# 下载数据
## 解释一下 download_and_untar.sh 这个shell
## 参数必须是三个
## 第一个是data，第二个是url，第三个是part
## data所指向的目录必须存在否则报错
## 这个东西一般是用不到得，除非是自己得算法要使用网络上得数据进行自己分析。
local/download_and_untar.sh $data $data_url data_aishell || exit 1;
local/download_and_untar.sh $data $data_url resource_aishell || exit 1;

# Lexicon Preparation,
## 准备字典
## 显然在上面的步骤中，创建了resource_aishell 的文件夹
local/aishell_prepare_dict.sh $data/resource_aishell || exit 1;

# Data Preparation,
## 
local/aishell_data_prep.sh $data/data_aishell/wav $data/data_aishell/transcript || exit 1;

# Phone Sets, questions, L compilation
## 准备音素集
utils/prepare_lang.sh --position-dependent-phones false data/local/dict \
    "<SPOKEN_NOISE>" data/local/lang data/lang || exit 1;

# LM training
## language model
## 准备语言模型
local/aishell_train_lms.sh || exit 1;

# G compilation, check LG composition
utils/format_lm.sh data/lang data/local/lm/3gram-mincount/lm_unpruned.gz \
    data/local/dict/lexicon.txt data/lang_test || exit 1;

# Now make MFCC plus pitch features.
# mfccdir should be some place with a largish disk where you want to store MFCC features.
## 训练mfcc模型
mfccdir=mfcc
for x in train dev test; do
  steps/make_mfcc_pitch.sh --cmd "$train_cmd" --nj 10 data/$x exp/make_mfcc/$x $mfccdir || exit 1;
  steps/compute_cmvn_stats.sh data/$x exp/make_mfcc/$x $mfccdir || exit 1;
  utils/fix_data_dir.sh data/$x || exit 1;
done

# 训练单音素模型
steps/train_mono.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/mono || exit 1;

# Monophone decoding
# 单音素解码
utils/mkgraph.sh data/lang_test exp/mono exp/mono/graph || exit 1;
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/mono/graph data/dev exp/mono/decode_dev
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/mono/graph data/test exp/mono/decode_test

# Get alignments from monophone system.
steps/align_si.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/mono exp/mono_ali || exit 1;

# train tri1 [first triphone pass]
steps/train_deltas.sh --cmd "$train_cmd" \
 2500 20000 data/train data/lang exp/mono_ali exp/tri1 || exit 1;

# decode tri1
utils/mkgraph.sh data/lang_test exp/tri1 exp/tri1/graph || exit 1;
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/tri1/graph data/dev exp/tri1/decode_dev
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/tri1/graph data/test exp/tri1/decode_test

# align tri1
steps/align_si.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/tri1 exp/tri1_ali || exit 1;

# train tri2 [delta+delta-deltas]
steps/train_deltas.sh --cmd "$train_cmd" \
 2500 20000 data/train data/lang exp/tri1_ali exp/tri2 || exit 1;

# decode tri2
utils/mkgraph.sh data/lang_test exp/tri2 exp/tri2/graph
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/tri2/graph data/dev exp/tri2/decode_dev
steps/decode.sh --cmd "$decode_cmd" --config conf/decode.config --nj 10 \
  exp/tri2/graph data/test exp/tri2/decode_test

# train and decode tri2b [LDA+MLLT]
steps/align_si.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/tri2 exp/tri2_ali || exit 1;

# Train tri3a, which is LDA+MLLT,
steps/train_lda_mllt.sh --cmd "$train_cmd" \
 2500 20000 data/train data/lang exp/tri2_ali exp/tri3a || exit 1;

utils/mkgraph.sh data/lang_test exp/tri3a exp/tri3a/graph || exit 1;
steps/decode.sh --cmd "$decode_cmd" --nj 10 --config conf/decode.config \
  exp/tri3a/graph data/dev exp/tri3a/decode_dev
steps/decode.sh --cmd "$decode_cmd" --nj 10 --config conf/decode.config \
  exp/tri3a/graph data/test exp/tri3a/decode_test

# From now, we start building a more serious system (with SAT), and we'll
# do the alignment with fMLLR.

steps/align_fmllr.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/tri3a exp/tri3a_ali || exit 1;

steps/train_sat.sh --cmd "$train_cmd" \
  2500 20000 data/train data/lang exp/tri3a_ali exp/tri4a || exit 1;

utils/mkgraph.sh data/lang_test exp/tri4a exp/tri4a/graph
steps/decode_fmllr.sh --cmd "$decode_cmd" --nj 10 --config conf/decode.config \
  exp/tri4a/graph data/dev exp/tri4a/decode_dev
steps/decode_fmllr.sh --cmd "$decode_cmd" --nj 10 --config conf/decode.config \
  exp/tri4a/graph data/test exp/tri4a/decode_test

steps/align_fmllr.sh  --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/tri4a exp/tri4a_ali

# Building a larger SAT system.

steps/train_sat.sh --cmd "$train_cmd" \
  3500 100000 data/train data/lang exp/tri4a_ali exp/tri5a || exit 1;

utils/mkgraph.sh data/lang_test exp/tri5a exp/tri5a/graph || exit 1;
steps/decode_fmllr.sh --cmd "$decode_cmd" --nj 10 --config conf/decode.config \
   exp/tri5a/graph data/dev exp/tri5a/decode_dev || exit 1;
steps/decode_fmllr.sh --cmd "$decode_cmd" --nj 10 --config conf/decode.config \
   exp/tri5a/graph data/test exp/tri5a/decode_test || exit 1;

steps/align_fmllr.sh --cmd "$train_cmd" --nj 10 \
  data/train data/lang exp/tri5a exp/tri5a_ali || exit 1;

# nnet3
local/nnet3/run_tdnn.sh

# chain
local/chain/run_tdnn.sh

# getting results (see RESULTS file)
for x in exp/*/decode_test; do [ -d $x ] && grep WER $x/cer_* | utils/best_wer.sh; done 2>/dev/null

exit 0;
```

## 2. 数据准备

#### 2.1 数据下载`download_and_untar.sh` 
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

#### 2.2 准备字典`aishell_prepare_dict.sh`

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



#### 2.3 准备数据`aishell_data_prep.sh`

传入两个参数，第一个参数是audio_shell ，第二个参数是transcript文件，这里应该是超本。

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



这里需要查看一下这个`filter_scp.pl`文件

##### 2.3.1 `filter_scp.pl`文件

注意调用的方式是：

```bash
utils/filter_scp.pl -f 1 $dir/utt.list $aishell_text
utils/filter_scp.pl -f 1 $dir/utt.list $dir/utt2spk_all
utils/filter_scp.pl -f 1 $dir/utt.list $dir/wav.scp_all
# 一般是4个参数 -f 1 dir1 dir2
```

下面我们来看filter_scp

```perl
#!/usr/bin/env perl
# Copyright 2010-2012 Microsoft Corporation
#                     Johns Hopkins University (author: Daniel Povey)

# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#  http://www.apache.org/licenses/LICENSE-2.0
#
# THIS CODE IS PROVIDED *AS IS* BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, EITHER EXPRESS OR IMPLIED, INCLUDING WITHOUT LIMITATION ANY IMPLIED
# WARRANTIES OR CONDITIONS OF TITLE, FITNESS FOR A PARTICULAR PURPOSE,
# MERCHANTABLITY OR NON-INFRINGEMENT.
# See the Apache 2 License for the specific language governing permissions and
# limitations under the License.


# This script takes a list of utterance-ids or any file whose first field
# of each line is an utterance-id, and filters an scp
# file (or any file whose "n-th" field is an utterance id), printing
# out only those lines whose "n-th" field is in id_list. The index of
# the "n-th" field is 1, by default, but can be changed by using
# the -f <n> switch

$exclude = 0;
$field = 1;
$shifted = 0;

do {
  $shifted=0;
  # 第一个参数是否是--exclude
  if ($ARGV[0] eq "--exclude") {
    $exclude = 1;
    shift @ARGV;
    $shifted=1;
  }
  if ($ARGV[0] eq "-f") {
    $field = $ARGV[1];
    shift @ARGV; shift @ARGV;
    $shifted=1
  }
} while ($shifted);

if(@ARGV < 1 || @ARGV > 2) {
  die "Usage: filter_scp.pl [--exclude] [-f <field-to-filter-on>] id_list [in.scp] > out.scp \n" .
      "Prints only the input lines whose f'th field (default: first) is in 'id_list'.\n" .
      "Note: only the first field of each line in id_list matters.  With --exclude, prints\n" .
      "only the lines that were *not* in id_list.\n" .
      "Caution: previously, the -f option was interpreted as a zero-based field index.\n" .
      "If your older scripts (written before Oct 2014) stopped working and you used the\n" .
      "-f option, add 1 to the argument.\n" .
      "See also: utils/filter_scp.pl .\n";
}

# filter_scp.pl -f 1 $dir/utt.list $aishell_text

$idlist = shift @ARGV; # 注意是$ 只有一个
open(F, "<$idlist") || die "Could not open id-list file $idlist";
while(<F>) {
  @A = split;
  @A>=1 || die "Invalid id-list file line $_";
  $seen{$A[0]} = 1;
}

if ($field == 1) { # Treat this as special case, since it is common.
  while(<>) {
    $_ =~ m/\s*(\S+)\s*/ || die "Bad line $_, could not get first field.";
    # $1 is what we filter on.
    if ((!$exclude && $seen{$1}) || ($exclude && !defined $seen{$1})) {
      print $_;
    }
  }
} else {
  while(<>) {
    @A = split;
    @A > 0 || die "Invalid scp file line $_";
    @A >= $field || die "Invalid scp file line $_";
    if ((!$exclude && $seen{$A[$field-1]}) || ($exclude && !defined $seen{$A[$field-1]})) {
      print $_;
    }
  }
}

# tests:
# the following should print "foo 1"
# ( echo foo 1; echo bar 2 ) | utils/filter_scp.pl <(echo foo)
# the following should print "bar 2".
# ( echo foo 1; echo bar 2 ) | utils/filter_scp.pl -f 2 <(echo 2)

```



####  2.4 准备语言

`prepare_lang.sh`准备这个模型的时候还是使用了python



####  2.5 准备语言模型

`aishell_train_lms.sh`



## 3. 特征提取

`make_mfcc_pitch.sh`

```bash
#!/bin/bash

# Copyright 2013 The Shenzhen Key Laboratory of Intelligent Media and Speech,
#                PKU-HKUST Shenzhen Hong Kong Institution (Author: Wei Shi)
#           2016  Johns Hopkins University (Author: Daniel Povey)
# Apache 2.0
# Combine MFCC and pitch features together
# Note: This file is based on make_mfcc.sh and make_pitch_kaldi.sh

# Begin configuration section.

## Job 数
nj=4
## 运行脚本
cmd=run.pl
## mfcc 的设置
mfcc_config=conf/mfcc.conf
## pitch 的设置
pitch_config=conf/pitch.conf
pitch_postprocess_config=
paste_length_tolerance=2
compress=true
write_utt2num_frames=false  # if true writes utt2num_frames
# End configuration section.

echo "$0 $@"  # Print the command line for logging

## 运行path.sh
if [ -f path.sh ]; then . ./path.sh; fi
. parse_options.sh || exit 1;


# 注意参数必须在 [1,3] 之间
if [ $# -lt 1 ] || [ $# -gt 3 ]; then
   echo "Usage: $0 [options] <data-dir> [<log-dir> [<mfcc-dir>] ]";
   echo "e.g.: $0 data/train exp/make_mfcc/train mfcc"
   echo "Note: <log-dir> defaults to <data-dir>/log, and <mfcc-dir> defaults to <data-dir>/data"
   echo "Options: "
   echo "  --mfcc-config              <mfcc-config-file>        # config passed to compute-mfcc-feats "
   echo "  --pitch-config             <pitch-config-file>       # config passed to compute-kaldi-pitch-feats "
   echo "  --pitch-postprocess-config <postprocess-config-file>  # config passed to process-kaldi-pitch-feats "
   echo "  --paste-length-tolerance   <tolerance>               # length tolerance passed to paste-feats"
   echo "  --nj                       <nj>                      # number of parallel jobs"
   echo "  --cmd (utils/run.pl|utils/queue.pl <queue opts>)     # how to run jobs."
   echo "  --write-utt2num-frames <true|false>     # If true, write utt2num_frames file."
   exit 1;
fi

data=$1

## 参数 >=2 
if [ $# -ge 2 ]; then
  logdir=$2
else
  logdir=$data/log
fi

## 参数 >= 3
if [ $# -ge 3 ]; then
  mfcc_pitch_dir=$3
else
  mfcc_pitch_dir=$data/data
fi

# make $mfcc_pitch_dir an absolute pathname.
# @ARGV 表示从pl 后面的参数列表，从0开始索引
mfcc_pitch_dir=`perl -e '($dir,$pwd)= @ARGV; if($dir!~m:^/:) { $dir = "$pwd/$dir"; } print $dir; ' $mfcc_pitch_dir ${PWD}`

# use "name" as part of name of the archive.
name=`basename $data`

mkdir -p $mfcc_pitch_dir || exit 1;
mkdir -p $logdir || exit 1;

# 如果存在feats.scp 移动到备份文件夹中
if [ -f $data/feats.scp ]; then
  mkdir -p $data/.backup
  echo "$0: moving $data/feats.scp to $data/.backup"
  mv $data/feats.scp $data/.backup
fi

scp=$data/wav.scp

# 需要的三个配置文件
required="$scp $mfcc_config $pitch_config"

# 查看三个需要的文件是否存在
for f in $required; do
  if [ ! -f $f ]; then
    echo "make_mfcc_pitch.sh: no such file $f"
    exit 1;
  fi
done

# 这个shell的主要作用就是检验所有的data准备是否是正确的、合法的
utils/validate_data_dir.sh --no-text --no-feats $data || exit 1;

if [ ! -z "$pitch_postprocess_config" ]; then
  postprocess_config_opt="--config=$pitch_postprocess_config";
else
  postprocess_config_opt=
fi

if [ -f $data/spk2warp ]; then
  echo "$0 [info]: using VTLN warp factors from $data/spk2warp"
  vtln_opts="--vtln-map=ark:$data/spk2warp --utt2spk=ark:$data/utt2spk"
elif [ -f $data/utt2warp ]; then
  echo "$0 [info]: using VTLN warp factors from $data/utt2warp"
  vtln_opts="--vtln-map=ark:$data/utt2warp"
fi

# 开启多个作业
for n in $(seq $nj); do
  # the next command does nothing unless $mfcc_pitch_dir/storage/ exists, see
  # utils/create_data_link.pl for more info.
  utils/create_data_link.pl $mfcc_pitch_dir/raw_mfcc_pitch_$name.$n.ark
done

# 一般用不到这个
if $write_utt2num_frames; then
  write_num_frames_opt="--write-num-frames=ark,t:$logdir/utt2num_frames.JOB"
else
  write_num_frames_opt=
fi

# scp 的作用就是为了方便nj的操作
if [ -f $data/segments ]; then
  echo "$0 [info]: segments file exists: using that."
  split_segments=""
  for n in $(seq $nj); do
    split_segments="$split_segments $logdir/segments.$n"
  done

  utils/split_scp.pl $data/segments $split_segments || exit 1;
  rm $logdir/.error 2>/dev/null

  mfcc_feats="ark:extract-segments scp,p:$scp $logdir/segments.JOB ark:- | compute-mfcc-feats $vtln_opts --verbose=2 --config=$mfcc_config ark:- ark:- |"
  pitch_feats="ark,s,cs:extract-segments scp,p:$scp $logdir/segments.JOB ark:- | compute-kaldi-pitch-feats --verbose=2 --config=$pitch_config ark:- ark:- | process-kaldi-pitch-feats $postprocess_config_opt ark:- ark:- |"

  $cmd JOB=1:$nj $logdir/make_mfcc_pitch_${name}.JOB.log \
    paste-feats --length-tolerance=$paste_length_tolerance "$mfcc_feats" "$pitch_feats" ark:- \| \
    copy-feats --compress=$compress $write_num_frames_opt ark:- \
      ark,scp:$mfcc_pitch_dir/raw_mfcc_pitch_$name.JOB.ark,$mfcc_pitch_dir/raw_mfcc_pitch_$name.JOB.scp \
     || exit 1;

else
  echo "$0: [info]: no segments file exists: assuming wav.scp indexed by utterance."
  split_scps=""
  for n in $(seq $nj); do
    split_scps="$split_scps $logdir/wav_${name}.$n.scp"
  done

  utils/split_scp.pl $scp $split_scps || exit 1;

	# 这里的命令进行 Mfcc feature 的提取
  mfcc_feats="ark:compute-mfcc-feats $vtln_opts --verbose=2 --config=$mfcc_config scp,p:$logdir/wav_${name}.JOB.scp ark:- |"
  pitch_feats="ark,s,cs:compute-kaldi-pitch-feats --verbose=2 --config=$pitch_config scp,p:$logdir/wav_${name}.JOB.scp ark:- | process-kaldi-pitch-feats $postprocess_config_opt ark:- ark:- |"

	# 注意cmd 是一个pl 运行指令
  $cmd JOB=1:$nj $logdir/make_mfcc_pitch_${name}.JOB.log \
    paste-feats --length-tolerance=$paste_length_tolerance "$mfcc_feats" "$pitch_feats" ark:- \| \
    copy-feats --compress=$compress $write_num_frames_opt ark:- \
      ark,scp:$mfcc_pitch_dir/raw_mfcc_pitch_$name.JOB.ark,$mfcc_pitch_dir/raw_mfcc_pitch_$name.JOB.scp \
      || exit 1;

fi


# 下面就是一些后续处理
if [ -f $logdir/.error.$name ]; then
  echo "Error producing mfcc & pitch features for $name:"
  tail $logdir/make_mfcc_pitch_${name}.1.log
  exit 1;
fi

# concatenate the .scp files together.
for n in $(seq $nj); do
  cat $mfcc_pitch_dir/raw_mfcc_pitch_$name.$n.scp || exit 1;
done > $data/feats.scp

if $write_utt2num_frames; then
  for n in $(seq $nj); do
    cat $logdir/utt2num_frames.$n || exit 1;
  done > $data/utt2num_frames || exit 1
  rm $logdir/utt2num_frames.*
fi

rm $logdir/wav_${name}.*.scp  $logdir/segments.* 2>/dev/null

nf=`cat $data/feats.scp | wc -l`
nu=`cat $data/utt2spk | wc -l`
if [ $nf -ne $nu ]; then
  echo "It seems not all of the feature files were successfully processed ($nf != $nu);"
  echo "consider using utils/fix_data_dir.sh $data"
fi

if [ $nf -lt $[$nu - ($nu/20)] ]; then
  echo "Less than 95% the features were successfully generated.  Probably a serious error."
  exit 1;
fi

echo "Succeeded creating MFCC & Pitch features for $name"

```

`compute_cmvn_stats.sh`Compute cepstral mean and variance statistics per speaker.



`fix_data_dir.sh`fix_data_dir.sh





> kaldi 的训练分为如下几个：
>
> 1. train_mono.sh 用来训练单音子隐马尔科夫模型，一共进行40次迭代，每两次迭代进行一次对齐操作
> 2. train_deltas.sh 用来训练与上下文相关的三音子模型
> 3. train_lda_mllt.sh 用来进行线性判别分析和最大似然线性转换
> 4. train_sat.sh 用来训练发音人自适应，基于特征空间最大似然线性回归
> 5. nnet3/run_dnn.sh 用nnet3来训练DNN，包括xent和MPE
> 6. 用chain训练DNN

下面我们一点一点看

