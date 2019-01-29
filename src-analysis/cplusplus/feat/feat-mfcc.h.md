#  `feat-mfcc.h`代码分析

> 在namespae = kaldi中定义了一些类和结构

## 1. 代码结构

### 1.0 文件结构

定义了两个类

### 1.1 `MfccOptions:struct`

所包含的变量如下：

```cpp
FrameExtractionOptions frame_opts;
MelBanksOptions mel_opts;
int32 num_ceps;  // e.g. 13: num cepstral coeffs, counting zero.
bool use_energy;  // use energy; else C0
BaseFloat energy_floor;  // 0 by default; set to a value like 1.0 or 0.1 if
                          // you disable dithering.
bool raw_energy;  // If true, compute energy before preemphasis and windowing
BaseFloat cepstral_lifter;  // Scaling factor on cepstra for HTK compatibility.
                            // if 0.0, no liftering is done.
bool htk_compat;  // if true, put energy/C0 last and introduce a factor of
                  // sqrt(2) on C0 to be the same as HTK.
```



### 1.2 `MfccComputer:class`

所包含的属性如下：

```cpp
// disallow assignment.
MfccComputer &operator = (const MfccComputer &in);

const MelBanks *GetMelBanks(BaseFloat vtln_warp);

MfccOptions opts_;
Vector<BaseFloat> lifter_coeffs_;
Matrix<BaseFloat> dct_matrix_;  // matrix we left-multiply by to perform DCT.
BaseFloat log_energy_floor_;
std::map<BaseFloat, MelBanks*> mel_banks_;  // BaseFloat is VTLN coefficient.
SplitRadixRealFft<BaseFloat> *srfft_;

// note: mel_energies_ is specific to the frame we're processing, it's
// just a temporary workspace.
Vector<BaseFloat> mel_energies_;
```





## 2. 实现细节

### 2.1 引用的外部类

1. FrameExtractionOptions
2. MelBanksOptions
3. MelBanks
4. SplitRadixRealFft

