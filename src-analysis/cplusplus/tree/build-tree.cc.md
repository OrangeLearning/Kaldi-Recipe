# `build-tree.cc`

> build-tree 是一个包含很多的作用函数的文件

## 1. GenRandStats 函数

## 2. BuildTree 函数

## 3. ComputeTreeMapping 函数

## 4. BuildTreeTwoLevel 函数

## 5. ReadSymbolTableAsIntegers 函数

## 6. RemoveDuplicates 函数

## 7. ObtainSetsOfPhones 函数

## 8. AutomaticallyObtainQuestions 函数

> AutomaticallyObtainQuestions 用于自动生成问题集

### 8.1 函数定义

```c++
void AutomaticallyObtainQuestions(BuildTreeStatsType &stats,
                                  const std::vector<std::vector<int32>> &phone_sets_in,
                                  const std::vector<int32> &all_pdf_classes_in,
                                  int32 P,
                                  std::vector<std::vector<int32>> *questions_out);
```

### 8.2 代码流程

1. 统计音素

    这里的IsSortedAndUniq函数请参见/util/stl-utils.h

    这里phones 最后统计了所有的phone_set_in中的音素，并且排序phone_sets中对于所有的phone_sets进行排序

    ```c++
    std::vector<std::vector<int32>> phone_sets(phone_sets_in);
    std::vector<int32> phones;
    for (size_t i = 0; i < phone_sets.size(); i++)
    {
      std::sort(phone_sets[i].begin(), phone_sets[i].end());
      if (phone_sets[i].empty())
        KALDI_ERR << "Empty phone set in AutomaticallyObtainQuestions";
      if (!IsSortedAndUniq(phone_sets[i]))
        KALDI_ERR << "Phone set in AutomaticallyObtainQuestions contains duplicate phones";
      for (size_t j = 0; j < phone_sets[i].size(); j++)
        phones.push_back(phone_sets[i][j]);
    }
    std::sort(phones.begin(), phones.end());
    ```

2. 

## 9. KMeansClusterPhones 函数

## 10. ReadRootsFile 函数

