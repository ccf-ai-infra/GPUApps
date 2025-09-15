# `Flash Attention`

`Flash Attention`，是一种高效的**自注意力机制实现**，在加速注意力计算的同时也减少了内存占用。其核心原理是将输入分块，在每个块上分别进行注意力计算，从而减少对 HBM 的读写次数以提高计算效率。

标准 Attention 的计算为：
1. 计算 $QK^T$，得到初步的注意力分数 attention_score；
2. 添加位置编码以区别不同位置的元素，并乘以缩放系数；
3. 根据注意力掩码屏蔽对应位置的元素，得到 masked_attention_score；
4. 最后经过 softmax 计算，并与 $V$ 相乘，得到最终的注意力分数 attention_out。

从中可以发现，若 SRAM 无法存储完整的计算数据，则计算过程中需要频繁访问 HBM，I/O 需求为 $O(Nd+N^2)$。此前，由于 softmax 的计算需要遍历整个序列以得到整体最大值，所以优化的方式往往是通过近似的方式来降低数据占用的空间。

而 FlashAttention 给出了 softmax 的分块计算方式，以此代替原公式中的 softmax 计算，过程如下：
1. 提前对输入的 QKV 进行分块（分别按照 QKV 的 sequence length 方向切分）；
2. 增量计算分块 softmax ，并维护两个全局变量
    - $m$：已处理块的最大值
    - $\ell$：已处理块的指数和
    
    以计算 attention_out 中的第 j 块为例，需遍历 $Q$ 切分后的每一块
    1. $Q_0$ 参与计算时，正常计算，并将计算结果直接保存在 attention_out[j] 中，并更新 $m$ 与 $\ell$；
    2. 此后， $Q_i$ 参与计算时，先统计当前块 masked_attention_score 的最大值 $m_i$，并计算其指数和 $\ell_i$；
        ```
        P = e^{masked_attention_score - m_i}
        l_i = rowsum(P)
        ```
    3. 更新 $m_{new}$ 为 $\max(m_i,m)$ ，并根据新的最大值计算 $\ell_{new}$，再由此计算新的 attention_out[j]；
        ```
        l_new = e^{m-m_new}*l+e^{m_i-m_new}
        attention_out = (attention_out * l * e^{m-m_new} + e^{m_i-m_new} * P * V_j) / l_new 
        ```
    4. 分别用 $m_{new}$ 与 $\ell_{max}$ 更新 $m$ 与 $\ell$，开启新一块的计算;
    5. 遍历完 $Q$ 之后，即可得到完整的 attention_out[j]

FlashAttention-2 则在 FlashAttention 的基础上进行了两点改进：

1. 在计算局部 Attention 时，先不考虑分母的指数和，而是 $\mathbf{O}_{i}^{(j)}=\mathrm{diag}(e^{m_{i}^{(j-1)}-m_{i}^{(j)}})\mathbf{O}_{i}^{(j-1)}+\tilde{\mathbf{P}}_{i}^{(j)}\mathbf{V}_{j}$
2. 在最后一步再带入计算，得到正确的结果 $\mathbf{O}_{i}=\mathrm{diag}(\ell_{i}^{(T_{c})})^{-1}\mathbf{O}_{i}^{(T_{c})}$

同时，FlashAttention-2 将 $Q$ 移到了外循环，而 $K$ 移到了内循环。由于改进了算法，使得 warps 之间不需要相互通信，所以外循环可以放在不同的 thread block 上。
    

## 接口

### 计算

```c
infiniStatus_t infiniopFlashAttention(
    infiniopFlashAttentionDescriptor_t desc,
    void *workspace,
    size_t workspace_size,
    void *out,
    void *l,
    const void *q,
    const void *k,
    const void *v,
    const void *mask,
    void *stream
);
```

<div style="background-color: lightblue; padding: 1px;"> 参数： </div>

- `desc`:
  已使用 `infiniopCreateFlashAttentionDescriptor()` 初始化的算子描述符；
- `workspace`:
  指向算子计算所需的额外工作空间；
- `workspace_size`:
  `workspace` 的大小，单位：字节；
- `out`:
  注意力计算结果地址。张量限制见[创建算子描述](#创建算子描述)部分。
- `l`:
  logsumexp，用于反向传播计算的暂存结果。张量限制见[创建算子描述](#创建算子描述)部分。
- `q`:
  查询（Query）张量数据指针。张量限制见[创建算子描述](#创建算子描述)部分。
- `k`:
  键（Key）张量数据指针。张量限制见[创建算子描述](#创建算子描述)部分。
- `v`:
  值（Value）张量数据指针。张量限制见[创建算子描述](#创建算子描述)部分。
- `mask`:
  注意力掩码的数据指针，可选参数。取值为 `0` 时表示保留对应位置的元素（参与计算）；取值为 `-inf` （负无穷）时表示屏蔽对应位置的元素（即跳过，不参与计算）。张量限制见[创建算子描述](#创建算子描述)部分。
- `stream`:
  计算流/队列。

<div style="background-color: lightblue; padding: 1px;">  返回值：</div>

- [`INFINI_STATUS_SUCCESS`], [`INFINI_STATUS_NULL_POINTER`], [`INFINI_STATUS_INSUFFICIENT_WORKSPACE`], [`INFINI_STATUS_DEVICE_TYPE_NOT_SUPPORTED`], [`INFINI_STATUS_INTERNAL_ERROR`].

### 创建算子描述

```c
infiniStatus_t infiniopCreateFlashAttentionDescriptor(
    infiniopHandle_t handle,
    infiniopFlashAttentionDescriptor_t *desc_ptr,
    infiniopTensorDescriptor_t out_desc,
    infiniopTensorDescriptor_t l_desc,
    infiniopTensorDescriptor_t q_desc,
    infiniopTensorDescriptor_t k_desc,
    infiniopTensorDescriptor_t v_desc,
    infiniopTensorDescriptor_t mask_desc,
    infiniopAttentionMaskType_t mask_type
); 
```

<div style="background-color: lightblue; padding: 1px;"> 参数：</div>

- `handle`:
  `infiniopHandle_t`类型的硬件控柄。详见 [`InfiniopHandle_t`]。
- `desc_ptr`:
  `infiniopFlashAttentionDescriptor_t` 指针，指向将被初始化的算子描述符地址。
- `out_desc` - { dT | ((batch_size,) seq_len_q, num_heads_q, head_dim) | ($\ldots, 1$)}:
  算子计算参数 `out` 的张量描述，四维或者三维，最后一维连续。
- `l_desc` - { dT | ((batch_szie,) seq_len_q, num_heads_q) | ($\ldots, 1$)};
  算子计算参数 `l` 的张量描述，三维或者二维。
- `q_desc` - { dT | ((batch_size,) seq_len_q, num_heads_q, head_dim) | ($\ldots, 1$)}:
  算子计算参数 `q` 的张量描述，形状与 `out_desc` 一致，最后一维连续。
- `k_desc` - { dT | ((batch_size,) seq_len_kv, num_heads_kv, head_dim) | ($\ldots, 1$)}:
  算子计算参数 `k` 的张量描述，最后一维连续。
- `v_desc` - { dT | ((batch_size,) seq_len_kv, num_heads_kv, head_dim) | ($\ldots, 1$)}:
  算子计算参数 `v` 的张量描述，形状与 `k_desc` 一致，最后一维连续。
- `mask_desc` - { dM | (seq_len_q, seq_len_kv) | (~)}:
  算子计算参数 `mask` 的张量描述，当 `mask_type=INFINIOP_ATTENTION_MASK_TYPE_FULL` 时，`mask` 不可为空，其余情况 `mask` 可为`nullptr`。
- `mask_type` - `infiniopAttentionMaskType_t`:
  注意力类型参数，有三种类型可选，详细见参数限制。

参数限制：

- `dT`: `Float16`, `Float32` 或 `BFloat16`。
- `dM`: `Flaot32`。
- `seq_len_q` 与 `seq_len_kv` 可以不同。
- `num_heads_q` 与 `num_heads_kv` 可以不同，但需满足前者是后者的整数倍（非0整数）。
  - 当 $N_q/N_{kv}=1$ 时，即为 MQA (multi-query attention)
  - 当 $N_q/N_{kv}>1$ 时，即为 GQA (grouped-query attention)
- `mask_type` 的三种类型：
  - `INFINIOP_ATTENTION_MASK_TYPE_NONE=0`: 不使用注意力掩码，忽略 `mask` 取值；
  - `INFINIOP_ATTENTION_MASK_TYPE_FULL=1`: 使用完整 mask 矩阵，此时 `mask` 不可为空；
  - `INFINIOP_ATTENTION_MASK_TYPE_CAUSAL=2`: 使用标准因果掩码，对应以左上顶点划分的下三角场景，忽略 `mask` 取值；

<div style="background-color: lightblue; padding: 1px;"> 返回值：</div>

- [`INFINI_STATUS_SUCCESS`], [`INFINI_STATUS_BAD_PARAM`],  [`INFINI_STATUS_BAD_TENSOR_SHAPE`], [`INFINI_STATUS_BAD_TENSOR_DTYPE`], [`INFINI_STATUS_BAD_TENSOR_STRIDES`], [`INFINI_STATUS_DEVICE_TYPE_NOT_SUPPORTED`].

### 计算额外工作空间

```c
infiniStatus_t infiniopGetFlashAttentionWorkspaceSize(
    infiniopFlashAttentionDescriptor_t desc, 
    size_t *size
);
```

<div style="background-color: lightblue; padding: 1px;"> 参数：</div>

- `desc`:
  已使用 `infiniopCreateFlashAttentionDescriptor()` 初始化的算子描述符；
- `size`:
  额外空间大小的计算结果的写入地址；

<div style="background-color: lightblue; padding: 1px;"> 返回值：</div>

- [`INFINI_STATUS_SUCCESS`], [`INFINI_STATUS_BAD_PARAM`], [`INFINI_STATUS_DEVICE_TYPE_NOT_SUPPORTED`].

### 销毁算子描述符

```c
infiniStatus_t infiniopDestoryFlashAttentionDescriptor(
    infiniopFlashAttentionDescriptor_t desc
);
```

<div style="background-color: lightblue; padding: 1px;"> 参数： </div>

- `desc`:
  输入。 待销毁的算子描述符；

<div style="background-color: lightblue; padding: 1px;"> 返回值： </div>

- [`INFINI_STATUS_SUCCESS`], [`INFINI_STATUS_DEVICE_TYPE_NOT_SUPPORTED`].

<!-- 链接 -->
[`InfiniopHandle_t`]: /infiniop/handle/README.md

[`INFINI_STATUS_SUCCESS`]: /common/status/README.md#INFINI_STATUS_SUCCESS
[`INFINI_STATUS_BAD_PARAM`]: /common/status/README.md#INFINI_STATUS_BAD_PARAM
[`INFINI_STATUS_DEVICE_TYPE_NOT_SUPPORTED`]: /common/status/README.md#INFINI_STATUS_DEVICE_TYPE_NOT_SUPPORTED
[`INFINI_STATUS_BAD_TENSOR_SHAPE`]: /common/status/README.md#INFINI_STATUS_BAD_TENSOR_SHAPE
[`INFINI_STATUS_BAD_TENSOR_DTYPE`]: /common/status/README.md#INFINI_STATUS_BAD_TENSOR_DTYPE
[`INFINI_STATUS_BAD_TENSOR_STRIDES`]: /common/status/README.md#INFINI_STATUS_BAD_TENSOR_STRIDES
[`INFINI_STATUS_NULL_POINTER`]:/common/status/README.md#INFINI_STATUS_NULL_POINTER
[`INFINI_STATUS_INSUFFICIENT_WORKSPACE`]:/common/status/README.md#INFINI_STATUS_INSUFFICIENT_WORKSPACE
[`INFINI_STATUS_INTERNAL_ERROR`]:/common/status/README.md#INFINI_STATUS_INTERNAL_ERROR
