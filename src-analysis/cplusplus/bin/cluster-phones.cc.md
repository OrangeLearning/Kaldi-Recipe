# `cluster-phones.cc`代码

## 1.引用库

```c++
#include "base/kaldi-common.h"
#include "util/common-utils.h"
#include "tree/context-dep.h"
#include "tree/build-tree.h"
#include "tree/build-tree-utils.h"
#include "tree/context-dep.h"
#include "tree/clusterable-classes.h"
#include "util/text-utils.h"
```

## 2.作用

> Cluster phones (or sets of phones) into sets for various purposes
>
> 对于（三）音素进行聚类
>
> 用法如下：
>
> ```bash
> cluster-phones [options] <tree-stats-in> <phone-sets-in> <clustered-phones-out>
> cluster-phones 1.tacc phonesets.txt questions.txt
> ```

## 3.代码流程

1. 读入的参数

    ```c++
    std::string stats_rxfilename = po.GetArg(1),
            phone_sets_rxfilename = po.GetArg(2),
            phone_sets_wxfilename = po.GetArg(3);
    ```

2. 定义**BuildTreeStatsType** 并且 读入该数据

    ```c++
    BuildTreeStatsType stats;
    {  // Read tree stats.
      bool binary_in;
      GaussClusterable gc;  // dummy needed to provide type.
      Input ki(stats_rxfilename, &binary_in);
      ReadBuildTreeStats(ki.Stream(), binary_in, gc, &stats);
    }
    ```

3. 读取**pdf_class_list**和**phone_sets**

    ```c++
    std::vector<int32> pdf_class_list;
    if (!SplitStringToIntegers(pdf_class_list_str, ":", false, &pdf_class_list)
        || pdf_class_list.empty()) {
      KALDI_ERR << "Invalid pdf-class-list string [expecting colon-separated list of integers]: " 
                  << pdf_class_list_str;
    }
    
    std::vector<std::vector< int32> > phone_sets;
    if (!ReadIntegerVectorVectorSimple(phone_sets_rxfilename, &phone_sets))
      KALDI_ERR << "Could not read phone sets from "
                  << PrintableRxfilename(phone_sets_rxfilename);
    ```

4. 核心部分，生成问题集

    mode表示得到问题集的方式的不同，mode的默认值是questions

    关键的函数`AutomaticallyObtainQuestions`和`KMeansClusterPhones`：

    * AutomaticallyObtainQuestions 函数请参见 /tree/build-tree.cc
    * KMeansClusterPhones 函数请参见 /tree/build-tree.cc

    ```c++
    if (mode == "questions")
    {
      AutomaticallyObtainQuestions(stats,phone_sets,pdf_class_list,P,&phone_sets_out);
    }
    else if (mode == "k-means")
    {
      KMeansClusterPhones(stats,phone_sets,pdf_class_list,P,num_classes,&phone_sets_out);
    }
    
    ```