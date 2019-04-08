# `combine_data.sh`的代码分析

## 1. 参数

必须要对于两个参数

```bash
utils/combine_data.sh <dest-data-dir> <src-data-dir1> <src-data-dir2>
```

第一个是目标文件夹，后面可以跟很多数据



## 2. 数据准备要求

* 要求每个`src-data-dir`下面都需要utt2spk文件

