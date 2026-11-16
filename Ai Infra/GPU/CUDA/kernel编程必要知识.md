明白一个kernel从头到尾都在干什么，并判断他为什么快或者慢

## Thread Block Grid索引
grid：整个任务有多少个 block。
block：一个 block 里有多少个 thread。
threadIdx：线程在 block 内的编号。
blockIdx：block 在 grid 内的编号。
blockDim：每个 block 的线程数。

常用的索引计算：
int tid = blockIdx.x * blockDim.x + threadIdx.x;

一个kernel启动的时候长成这样子：
```cuda
kernel<<<grid_size, block_size>>>(...);
```
如：
```cuda
kernel<<<32, 32>>>
```
grid 有 32 个 block,每个 block 有 32 个线程
总共1024线程
线程 0 处理 `data[0]`,线程 1 处理 `data[1]`,线程 32 处理` data[32]`

#### 常见映射方式
一个block处理一个样本,一行向量,一个attention head
一个grid覆盖整个batch或者整个序列
一个thread仅仅处理几个元

永远先看: 谁负责哪块元素

## Native Softmax
假设输入是`[batch,dim]`,每一行独立做softmax

常见写法是：每个 block 处理一行，block 内线程协作完成归约

```Cpp
__global__ void softmax_kernel(float* input, float* output, int dim) {
    int row = blockIdx.x;
    float* row_in  = input  + row * dim;
    float* row_out = output + row * dim;

    // 1. 求行内最大值
    float max_val = -INFINITY;
    for (int i = threadIdx.x; i < dim; i += blockDim.x)
        max_val = fmaxf(max_val, row_in[i]);

    // block 内归约 max
    __shared__ float s_max;
    max_val = blockReduceMax(max_val);
    if (threadIdx.x == 0) s_max = max_val;
    __syncthreads();

    // 2. 求 exp 的和
    float sum = 0.0f;
    for (int i = threadIdx.x; i < dim; i += blockDim.x)
        sum += expf(row_in[i] - s_max);

    sum = blockReduceSum(sum);
    __shared__ float s_sum;
    if (threadIdx.x == 0) s_sum = sum;
    __syncthreads();

    // 3. 写输出
    for (int i = threadIdx.x; i < dim; i += blockDim.x)
        row_out[i] = expf(row_in[i] - s_max) / s_sum;
}
```

Softmax本身计算并不难，难的是如何减少数据搬运
访存上，naive 多 kernel 实现会反复读写 global memory：读 input、写 max、读 max、写 exp、读 exp、写 sum，最后再读一遍算除法。这样同一个数据会被搬运很多次
优化后的做法是把三步放在同一个 kernel 里，尽量让数据留在寄存器或 shared memory 中，减少 HBM 访问次数。
#### 生产级别实现细节
Wrap级别的规约： 先用` __shfl_xor_sync` 在 32 线程的 warp 内归约，比 shared memory 更快。

分块 + 向量化读取： 用float一次读取4个float，提高带宽利用率

Online SoftMax： 不用先读一遍求 max，再一次求 sum。它把 max 和 sum 合并成一次遍历