
> 日期：2026-10-26  
> 环境：Ubuntu 24.04.3 LTS（WSL、x86_64），CANN Toolkit 与 Ascend910B ops 均为 9.2.0；目标架构 `dav-2201`。
>  前置阅读：[[Ascend C 算子开发交付实践/01-第一个 Hello World 的 AscendC 程序]]。  

# 一、选定原始样例

本文直接使用 `cann-samples` 提交 `3a85e824537feaef646bd89eb4d0e799cc2886b5` 中的 `Samples/0_Introduction/01_simd_cpp_api/01_add/add` 工程〔1—3〕。

目录内已有 `add.asc` 和 `CMakeLists.txt`：前者同时包含 Kernel 与 Host，后者生成可执行文件 `demo`。整个实验不改样例源码，也不引入额外的 `cpu_debug_launch.h`、独立 CPU 调试入口或 `vector_add_cpu_debug` 补丁。CPU 模式使用 CANN 9.2.0 自带的编译与运行支持。

本例计算 `z[i] = x[i] + y[i]`。Host 构造 16384 个 `float` 元素，启动 8 个 Block，每个 Block 处理 2048 个元素。下文先从干净构建目录跑通原始工程，再沿 Kernel 的执行顺序分析 GM、UB 和向量计算之间的数据流。

| 本机验证项    | 结果                                   |
| -------- | ------------------------------------ |
| 样例源码     | 固定提交的 `add.asc`，没有本地修改               |
| CPU 孪生   | 直接构建并运行，8 个模拟核正常退出，`test pass!`      |
| NPU 架构仿真 | 直接构建并运行，仿真模型正常停止，`test pass!`        |
| 真实 NPU   | WSL 没有 `/dev/davinci0`，本篇未做上板运行或性能测量 |

`cpu` 在 Host CPU 上运行调试版 Kernel；`sim` 使用 CANN 的 `dav-2201` NPU 仿真器。两者都不要求这台 WSL 有真实 NPU，但生成和链接的程序不同，应使用**独立构建目录**〔3、6〕。

# 二、从零配置、编译与运行

以下命令在**同一个 WSL Shell** 中执行。当前环境已有 `$HOME/projects/cann-samples`。首次获取源码时，先执行：

```bash
mkdir -p "$HOME/projects"
git clone https://gitcode.com/cann/cann-samples.git "$HOME/projects/cann-samples"
git -C "$HOME/projects/cann-samples" switch --detach 3a85e824537feaef646bd89eb4d0e799cc2886b5
```

已有仓库时跳过 `git clone`。核对本篇分析的源码版本并加载 CANN 环境：

```bash
repo="$HOME/projects/cann-samples"
git -C "$repo" rev-parse HEAD
# 预期：3a85e824537feaef646bd89eb4d0e799cc2886b5
source "$HOME/Ascend/cann-9.2.0/set_env.sh"
sample="$repo/Samples/0_Introduction/01_simd_cpp_api/01_add/add"
cd ${sample}
work=${sample} # 可任意指定
```

若本地提交不同，以下命令仍可尝试，但应以实际 `add.asc` 为准核对代码和输出。
## 2.1 CPU 孪生模式

```bash
cd ${sample}
cmake -S "$sample" -B "$work/build-cpu" \
  -DCMAKE_ASC_RUN_MODE=cpu \
  -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$work/build-cpu" -j 4
mkdir -p "$work/run-cpu"
(cd "$work/run-cpu" && "$work/build-cpu/demo")
```

`-S` 指向原始样例；`-B` 指向全新构建目录。第一次 `cmake` 读取自带的 `CMakeLists.txt` 并配置 ASC 编译器、架构和 CPU 运行模式；`cmake --build` 才真正编译 `add.asc`、链接 `demo`。

本机全新目录编译得到 `[100%] Built target demo`，运行退出码为 0。8 个模拟核均显示 `exit success!`，输出与参考值的前 20 项相同，末尾为：

```text
Output: 0 0.3 0.6 0.9 1.2 1.5 1.8 2.1 2.4 2.7 ... 
Golden: 0 0.3 0.6 0.9 1.2 1.5 1.8 2.1 2.4 2.7 ...
test pass!
```

CPU 输出中也出现了 `[TmSim]: Run in serial mode.`；这条内部日志**不表示当前构建采用了 `sim` 模式**。运行模式由 CMake 配置的 `CMAKE_ASC_RUN_MODE=cpu` 决定。

## 2.2 NPU 架构仿真模式

```bash
cmake -S "$sample" -B "$work/build-sim" \
  -DCMAKE_ASC_RUN_MODE=sim \
  -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$work/build-sim" -j 4
mkdir -p "$work/run-sim"
(cd "$work/run-sim" && "$work/build-sim/demo")
```

SIM 使用另一套构建目录，因为它与 CPU 模式生成和链接的程序不同。本机全新目录编译同样得到 `[100%] Built target demo`，运行时仿真模型正常停止，Host 输出 `Output` 与 `Golden` 一致，末尾为 `test pass!`，退出码为 0。SIM 模型时间**不是 Ascend910B 实卡耗时**，也不能与 CPU 孪生耗时直接比较。

# 三、从 Host 调用看 Kernel 的工作量

`main()` 创建各含 `8 × 2048 = 16384` 个 `float` 元素的输入 `x`、`y`，分别按 `x[i] = i × 0.1f`、`y[i] = i × 0.2f` 填值，再调用 `kernel_add(x, y)`。Host 设置：

```cpp
constexpr uint32_t numBlocks = 8;
constexpr uint32_t blockLength = 2048;
add_custom<blockLength><<<numBlocks, 0, stream>>>(xDevice, yDevice, zDevice);
aclrtSynchronizeStream(stream);
```

注意这里的 kernel 函数调用使用的是 C++ 模板，`<blockLength>` 是编译期模板实参，让 Kernel 知道每个 Block 处理 2048 个 `float`。`<<<numBlocks, 0, stream>>>` 启动 8 个逻辑 Block；中间的 `0` 表示本次启动不额外指定动态 UB 空间，**不代表 Kernel 不使用 UB**。三个函数实参是 GM 中两个输入和一个输出的起始地址，Host 通过 `aclrtMalloc` 为其分配空间。

Kernel 启动是异步的，所以 Host 用 `aclrtSynchronizeStream` 等待完成，然后将 `zDevice` 拷回 `zHost`，与逐元素计算的 `golden` 比较。末尾 `test pass!` 来自 `VerifyResult` 中的 `std::equal`，不是仅凭程序退出码判断。

# 四、逐段读懂 `add_custom` Kernel

下面是原始 `add.asc` 的计算路径。原文件还有一个 `#if 0` 包裹的 `printf`/`DumpTensor` 示例；它默认不参与编译，也不是外加的 CPU 调试代码〔1〕。

```cpp
template <uint32_t blockLength>
__vector__ __global__ void add_custom(__gm__ float* x, __gm__ float* y, __gm__ float* z)
{
    AscendC::InitSocState();

    AscendC::GlobalTensor<float> xGm, yGm, zGm;
    xGm.SetGlobalBuffer(x + block_idx * blockLength, blockLength);
    yGm.SetGlobalBuffer(y + block_idx * blockLength, blockLength);
    zGm.SetGlobalBuffer(z + block_idx * blockLength, blockLength);

    AscendC::LocalMemAllocator<AscendC::Hardware::UB> ubAllocator;
    AscendC::LocalTensor<float> xLocal = ubAllocator.Alloc<float, blockLength>();
    AscendC::LocalTensor<float> yLocal = ubAllocator.Alloc<float, blockLength>();
    AscendC::LocalTensor<float> zLocal = ubAllocator.Alloc<float, blockLength>();

    AscendC::DataCopy(xLocal, xGm, blockLength);
    AscendC::DataCopy(yLocal, yGm, blockLength);
    AscendC::PipeBarrier<PIPE_ALL>();

    AscendC::Add(zLocal, xLocal, yLocal, blockLength);
    AscendC::PipeBarrier<PIPE_ALL>();

    AscendC::DataCopy(zGm, zLocal, blockLength);
    AscendC::PipeBarrier<PIPE_ALL>();
}
```

## 4.1 函数签名和初始化

- `__global__` 表示这是由 Host 启动的 Kernel；`__vector__` 指定向量计算类型；`__gm__ float*` 表示传入的是 GM 地址，尚未把数据搬进 UB。
- `InitSocState()` 初始化静态 Tensor 编程所需的核内状态，位于每个 Block 的执行入口〔4〕。
- `blockLength` 是**一个 Block 负责的总元素数**。本例一次处理完 2048 个元素，没有 `tileSize` 变量使用块内循环。

## 4.2 用 `block_idx` 划分 GM 区间

`GlobalTensor<float>` 是 GM 数据的访问描述。
`SetGlobalBuffer(起始指针, 长度)` 将当前 Block 的三个描述对象绑定到各自负责的片段；它本身不是 GM→UB 搬运。`x + block_idx * blockLength` 按 `float` 元素移动指针，不是按字节移动。

| `block_idx` | 元素区间 | 起始偏移 | 本 Block 的输入与输出 |
| ---: | --- | ---: | --- |
| 0 | `[0, 2048)` | 0 个元素 | `x[0:2048]`、`y[0:2048]` → `z[0:2048]` |
| 1 | `[2048, 4096)` | 2048 个元素 | `x[2048:4096]`、`y[2048:4096]` → `z[2048:4096]` |
| 7 | `[14336, 16384)` | 14336 个元素 | 最后一段 2048 个元素 |

8 个 Block 的输出区间互不重叠，因此不需要跨核合并结果。样例固定的输入总长恰好为 `numBlocks × blockLength`；若长度改变，仅修改一个常量会使覆盖范围不匹配，这段 Kernel 没有尾块处理。

## 4.3 在 UB 中创建三个 `LocalTensor`

`LocalMemAllocator<Hardware::UB>` 管理当前 Block 的 UB 空间，连续三次 `Alloc<float, blockLength>()` 分别为两个输入和一个输出提供本地 Tensor〔5〕。每个 Tensor 容纳 `2048 × 4 = 8192` 字节，即 8 KiB；三个共 24 KiB，未计其他运行开销。三个 `LocalTensor` 是不同的缓冲区，`Add` 不会直接改写 `xGm` 或 `yGm`。

这里使用静态 Tensor 编程，没有 `TPipe`、`TQue`、双缓冲或 Host 侧 tiling 结构。启动参数中的 `0` 与 UB 分配不矛盾：三块缓冲区由 Kernel 内的分配接口创建。

## 4.4 搬入、计算、搬出与三个屏障

| 语句                                         | 数据去向    | 作用                                                    |
| ------------------------------------------ | ------- | ----------------------------------------------------- |
| `DataCopy(xLocal, xGm, blockLength)`       | GM → UB | 搬入本 Block 的 2048 个 `x` 元素。                            |
| `DataCopy(yLocal, yGm, blockLength)`       | GM → UB | 搬入对应的 `y` 元素。                                         |
| 第一个 `PipeBarrier<PIPE_ALL>()`              | 核内同步    | 等待搬入完成，让 `Add` 读取到有效的 UB 数据。                          |
| `Add(zLocal, xLocal, yLocal, blockLength)` | UB → UB | 对 2048 对元素执行向量加法，将结果写入 `zLocal`。源码没有 C++ 标量 `for` 循环。 |
| 第二个 `PipeBarrier<PIPE_ALL>()`              | 核内同步    | 等待向量计算完成，再允许搬出读取 `zLocal`。                            |
| `DataCopy(zGm, zLocal, blockLength)`       | UB → GM | 将结果写回本 Block 的 GM 输出片段。                               |
| 第三个 `PipeBarrier<PIPE_ALL>()`              | 核内同步    | 确保写回阶段完成后结束 Kernel。                                   |

`DataCopy` 的**第一个参数是目的地，第二个参数是来源**：判断搬入或搬出要看 `LocalTensor` 与 `GlobalTensor` 的位置。`PipeBarrier<PIPE_ALL>()` 是核内流水线之间的顺序约束；它不负责同步 8 个 Block，也不同于 Host 的 `aclrtSynchronizeStream`。本例各阶段按顺序执行，尚未通过双缓冲重叠搬入、计算与搬出。

```text
xGm ──DataCopy──> xLocal ──┐
                            Add ──> zLocal ──DataCopy──> zGm
yGm ──DataCopy──> yLocal ──┘
 ↓                  ↓                 ↓                    ↓
 GM                 UB                UB                  GM
```

# 五、验证范围与下一步

在指定提交和 CANN 9.2.0 下，**未经修改的** `add.asc` 使用自带 CMake 工程分别完成 CPU 孪生和 SIM 仿真，编译成功且 Host 比对都输出 `test pass!`。理解这段 Kernel 的主线是 **Block 分段 → GM 绑定 → UB 分配 → 搬入 → 向量加法 → 写回**。没有外加 CPU debug 代码；CPU/SIM 所需的工具链和运行库仍由 CANN 提供。

这仍是固定形状的**Kernel 直调样例**：它没有动态 Host Tiling、尾块处理、算子注册或算子包交付。CPU/SIM 功能通过也不代表测得 Ascend910B 实卡性能。所有的 kernel 实现都是基于静态 Tensor 编程，暂时不涉及学习 `TPipe`/`TQue` 的多 tile 数据流。

# 六、接口速查

## 6.1 `GlobalTensor` 与 `LocalTensor`

Ascend C 用 Tensor 对象描述 Kernel 访问的数据及其存储位置。`GlobalTensor<T>` 对应 GM（全局内存），`LocalTensor<T>` 对应 AI Core 的本地内存；本例的本地内存是 UB。`T` 是元素类型，这里均为 `float`〔9、10〕。

```cpp
AscendC::GlobalTensor<float> xGm, yGm, zGm;
AscendC::LocalMemAllocator<AscendC::Hardware::UB> ubAllocator;
AscendC::LocalTensor<float> xLocal = ubAllocator.Alloc<float, blockLength>();
```

`xGm` 等对象描述当前 Block 在 GM 中访问的输入、输出片段；`xLocal` 等对象描述当前 Block 在 UB 中使用的计算缓冲区。

注意：**声明 Tensor 对象与搬运数据是两件事**。

`GlobalTensor` 要先用 `SetGlobalBuffer` 绑定 GM 地址；本例的 `LocalTensor` 由 `LocalMemAllocator::Alloc` 返回；GM 与 UB 之间的数据移动再由 `DataCopy` 完成。`Add` 的三个操作数都是 `LocalTensor`，因此加法在 UB 数据上进行。

## 6.2 `GlobalTensor::SetGlobalBuffer`

本例调用的是设置起始地址和长度的重载〔7〕：

```cpp
void SetGlobalBuffer(__gm__ PrimType* buffer, uint64_t bufferSize);

xGm.SetGlobalBuffer(x + block_idx * blockLength, blockLength);
```

`buffer` 是当前 `GlobalTensor` 对应的 GM 起始地址。
- 这里 `x` 是整个输入的首地址，`block_idx * blockLength` 将指针移动到当前 Block 的片段。
- `bufferSize` 是**元素个数**，不是字节数：`xGm` 的类型为 `GlobalTensor<float>`，所以 `2048` 表示 2048 个 `float`，即 `2048 × 4 = 8192` 字节。
- `yGm`、`zGm` 以同样方式分别绑定输入 `y` 和输出 `z` 的片段；实际内存必须至少覆盖所指定的范围。

该接口返回 `void`，作用是初始化 GM Tensor 的地址与长度；它不分配 GM 内存，也不把数据搬到 UB。真正的搬入由后面的 `DataCopy(xLocal, xGm, blockLength)` 完成。

## 6.3 `LocalMemAllocator`

`LocalMemAllocator<Hardware::UB>` 是本例使用的本地内存分配器，模板参数 `Hardware::UB` 指定从 UB 中为 `LocalTensor` 安排空间〔11〕：

```cpp
AscendC::LocalMemAllocator<AscendC::Hardware::UB> ubAllocator;
```

该分配器按调用 `Alloc` 的顺序线性安排地址。

本例先给 `xLocal`，再给 `yLocal`，最后给 `zLocal` 安排 UB 空间；每个 Block 都运行同一段 Kernel，并管理自己使用的本地缓冲区。它属于静态 Tensor 编程方式，不与 `TPipe`、`TQue` 的内存管理接口混用。`LocalMemAllocator` 负责安排本地空间，**不负责从 GM 读取数据**。

## 6.4 `LocalMemAllocator::Alloc`

本例使用 `LocalMemAllocator<Hardware::UB>` 的编译期长度重载〔8〕：

```cpp
template <class DataType, uint32_t tileSize>
LocalTensor<DataType> Alloc();

AscendC::LocalTensor<float> xLocal = ubAllocator.Alloc<float, blockLength>();
```

- `DataType=float` 指定元素类型，`tileSize=blockLength=2048` 指定 **LocalTensor 的元素个数**。
- `blockLength` 是 `constexpr`，可以作为模板实参；调用返回一个 `LocalTensor<float>`，对应当前 Block 在 UB 中使用的 2048 个 `float` 空间（8 KiB）。样例连续调用三次，得到 `xLocal`、`yLocal`、`zLocal`，合计申请 24 KiB；申请量必须在当前 UB 的剩余空间内。

接口参数名虽然叫 `tileSize`，但本例的 `blockLength` 是一个 Block 的全部工作量，并没有块内多轮 tile 循环。`Alloc` 只取得本地 Tensor 空间，不会自动将 GM 数据填进去；后续 `DataCopy` 才负责搬入。若元素数只能在运行时确定，接口另有 `Alloc<float>(tileSize)` 重载；本例没有使用它。

## 6.5 `DataCopy`

本例使用 GM 与 UB 之间的**连续搬运**重载〔12〕。两种方向的原型分别是：

```cpp
// GM → UB
template <typename T>
__aicore__ inline void DataCopy(const LocalTensor<T>& dst, const GlobalTensor<T>& src, const uint32_t count);

// UB → GM
template <typename T>
__aicore__ inline void DataCopy(const GlobalTensor<T>& dst, const LocalTensor<T>& src, const uint32_t count);
```

第一个参数 `dst` 是目的地，第二个参数 `src` 是来源；`count` 是**元素个数**，并非字节数。本例的三次调用分别搬入 `x`、搬入 `y`、写回 `z`：

```cpp
AscendC::DataCopy(xLocal, xGm, blockLength); // GM → UB
AscendC::DataCopy(yLocal, yGm, blockLength); // GM → UB
AscendC::DataCopy(zGm, zLocal, blockLength); // UB → GM
```

每次 `count=2048`，类型为 `float`，实际传输 `2048 × 4 = 8192` 字节。连续搬运接口要求传输字节数按 32 字节对齐；8192 满足这一条件。`DataCopy` 只复制数据，不执行加法：中间的 `Add(zLocal, xLocal, yLocal, blockLength)` 才产生结果。样例还用 `PipeBarrier<PIPE_ALL>()` 约束“搬入完成 → 计算 → 搬出”的先后关系。

# 七、进阶一：让最后一个 Block 处理较短的尾块

前面分析的是**未修改的原始样例**：8 个 Block 各处理 2048 个元素，总长 16384。现在单独制作一个练习副本，使前 `N-1` 个 Block 的工作量相同，第 `N` 个 Block 只处理剩余元素。不要直接改仓库里的原始 `add.asc`，以便保留前面章节的基准结果。

## 7.1 先确定切分规则

设每个完整 Block 处理 `B=2048` 个 `float`，总长 `L=7×2048+1000=15336`，则 `N=⌈L/B⌉=8`。各 Block 的起点为 `offset = block_idx × B`，本 Block 的实际长度为 `currentLength = min(B, L-offset)`：

| `block_idx` | 起点 `offset` | 实际长度 `currentLength` | GM 元素区间 |
| ---: | ---: | ---: | --- |
| 0～6 | `block_idx × 2048` | 2048 | 各自不重叠的完整片段 |
| 7 | 14336 | 1000 | `[14336, 15336)` |

尾块先选 **1000 个 `float`**：`1000×4=4000` 字节，恰好是 32 字节的整数倍，满足本例连续 `DataCopy` 的搬运对齐要求〔12〕。这里暂不处理尾长为 1001 等不满足该条件的情况；要覆盖任意元素数，需要另行设计非对齐尾块搬运。

## 7.2 修改临时副本中的 Host

在 WSL 中复制现有样例工程；以下路径沿用第二节定义的 `sample` 变量：

```bash
tailWork="$(mktemp -d /tmp/ascend-add-tail-XXXXXX)"
cp -a "$sample" "$tailWork/sample"
```

只编辑 `"$tailWork/sample/add.asc"`。先将 `main` 中的总长改为：

```cpp
constexpr uint32_t totalLength = 7 * 2048 + 1000;
```

然后在 `kernel_add` 中保留 `blockLength=2048`，根据实际总长计算 Block 数，并把总长作为第四个 Kernel 实参传入：

```cpp
constexpr uint32_t blockLength = 2048;
uint32_t totalLength = x.size();
uint32_t numBlocks = (totalLength + blockLength - 1) / blockLength;
// 原有 aclrtMalloc、输入拷贝等代码保持原样。
add_custom<blockLength><<<numBlocks, 0, stream>>>(
    xDevice, yDevice, zDevice, totalLength);
```

此处 `numBlocks` 是逻辑 Block 数。对本次 `L=15336`，向上取整得到 8；输入长度应为正数，且 `x`、`y` 长度一致。本练习的 `main` 已构造满足条件的两个向量。

## 7.3 修改 Kernel 的实际处理长度

将 Kernel 函数签名增加 `uint32_t totalLength`，在 `InitSocState()` 后计算当前 Block 的 `offset` 和 `currentLength`。用 `currentLength` 绑定 GM 片段，并作为 `DataCopy`、`Add` 的本核实际运算元素数：

```cpp
template <uint32_t blockLength>
__vector__ __global__ void add_custom(
    __gm__ float* x, __gm__ float* y, __gm__ float* z, uint32_t totalLength)
{
    AscendC::InitSocState();
    const uint32_t offset = block_idx * blockLength;
    // 本核的实际运算元素数目
    uint32_t currentLength = (totalLength - offset) <= blockLength ? (totalLength - offset) : blockLength;

    AscendC::GlobalTensor<float> xGm, yGm, zGm;
    xGm.SetGlobalBuffer(x + offset, currentLength);
    yGm.SetGlobalBuffer(y + offset, currentLength);
    zGm.SetGlobalBuffer(z + offset, currentLength);

    AscendC::LocalMemAllocator<AscendC::Hardware::UB> ubAllocator;
    AscendC::LocalTensor<float> xLocal = ubAllocator.Alloc<float, blockLength>();
    AscendC::LocalTensor<float> yLocal = ubAllocator.Alloc<float, blockLength>();
    AscendC::LocalTensor<float> zLocal = ubAllocator.Alloc<float, blockLength>();

    AscendC::DataCopy(xLocal, xGm, currentLength);
    AscendC::DataCopy(yLocal, yGm, currentLength);
    AscendC::PipeBarrier<PIPE_ALL>();
    AscendC::Add(zLocal, xLocal, yLocal, currentLength);
    AscendC::PipeBarrier<PIPE_ALL>();
    AscendC::DataCopy(zGm, zLocal, currentLength);
    AscendC::PipeBarrier<PIPE_ALL>();
}
```

这一行三元表达式等价于取 `min(totalLength - offset, blockLength)`；等于 `blockLength` 时选哪一支，结果都相同。它成立的前提是 `blockLength > 0`、`totalLength > 0`，且 Host 按 `numBlocks = ⌈totalLength / blockLength⌉` 启动，保证每个已启动 Block 的 `offset < totalLength`。若在其他场景中额外启动了 `offset >= totalLength` 的 Block，`uint32_t` 减法会下溢，不能直接使用此写法。

最后一个 Block **仍按 `blockLength` 为三个 `LocalTensor` 各准备 2048 元素的 UB 容量，但只搬运和计算 `currentLength=1000` 个元素**。

容量与实际工作量不同；把 `SetGlobalBuffer`、`DataCopy`、`Add` 都改为实际长度，才不会让尾核读取或写入输入范围之外的数据。前 7 个 Block 的 `currentLength` 仍为 2048。

## 7.4 分别编译 CPU 与 SIM 并核对结果

完成上述编辑后，用两个独立构建目录运行临时副本：

```bash
cmake -S "$tailWork/sample" -B "$tailWork/build-cpu" \
  -DCMAKE_ASC_RUN_MODE=cpu -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$tailWork/build-cpu" -j 4
mkdir -p "$tailWork/run-cpu"
(cd "$tailWork/run-cpu" && "$tailWork/build-cpu/demo")

cmake -S "$tailWork/sample" -B "$tailWork/build-sim" \
  -DCMAKE_ASC_RUN_MODE=sim -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$tailWork/build-sim" -j 4
mkdir -p "$tailWork/run-sim"
(cd "$tailWork/run-sim" && "$tailWork/build-sim/demo")
```

本机使用 CANN 9.2.0 在新的 `/tmp` 副本上验证：两种模式都编译到 `[100%] Built target demo`，运行均输出 `test pass!`。`VerifyResult` 用 `std::equal` 比较完整的 15336 个结果，虽然终端只展示前 20 项，尾块也包含在比较范围内。此练习解决的是**长度对齐但工作量不均匀**的尾核；它尚未解决任意长度的非对齐尾块。

# 八、进阶二：处理非 32 字节对齐的任意尾长

进阶一的尾长 1000 个 `float`，即 4000 字节，满足连续 `DataCopy` 的 32 字节对齐要求。若改为 `L=7×2048+1001=15337`，最后一个 Block 的 1001 个 `float` 只占 **4004 字节**，无法被 32 整除。继续使用第七节的 `DataCopy(..., currentLength)`，就不能保证尾部完整搬运；本节改用 A2 支持的非对齐搬运接口 `DataCopyPad`〔14、15〕。

## 8.1 在进阶一的副本上继续修改

先完成第七节，再复制一份已经实现不均匀切分的样例。以下命令假定 `tailWork` 仍是第七节创建的变量：

```bash
anyWork="$(mktemp -d /tmp/ascend-add-anylen-XXXXXX)"
cp -a "$tailWork/sample" "$anyWork/sample"
```

只编辑 `"$anyWork/sample/add.asc"`。保留第七节的 `offset`、`currentLength`、GM 片段绑定和 Host 传入 `totalLength` 的逻辑；把 Kernel 中原来的三次 `DataCopy` 替换为下列调用，`Add` 及三个 `PipeBarrier` 保持原有顺序：

```cpp
// blockCount=1；blockLen 按字节指定；两个 stride 和保留字段均为 0。
AscendC::DataCopyExtParams copyParams{
    1, static_cast<uint32_t>(currentLength * sizeof(float)), 0, 0, 0};
AscendC::DataCopyPadExtParams<float> padParams{false, 0, 0, 0.0f};

AscendC::DataCopyPad(xLocal, xGm, copyParams, padParams); // GM → UB
AscendC::DataCopyPad(yLocal, yGm, copyParams, padParams); // GM → UB
AscendC::PipeBarrier<PIPE_ALL>();
AscendC::Add(zLocal, xLocal, yLocal, currentLength);
AscendC::PipeBarrier<PIPE_ALL>();
AscendC::DataCopyPad(zGm, zLocal, copyParams);             // UB → GM
AscendC::PipeBarrier<PIPE_ALL>();
```

`DataCopyExtParams` 的 `blockLen` 单位是**字节**，与连续 `DataCopy` 的 `count` 单位（元素）不同。本例 `currentLength=1001` 时，`blockLen=4004`。`sizeof(float)` 的结果类型是 `size_t`；`blockLen` 字段为 `uint32_t`，这里显式转换是为了避免列表初始化的窄化编译错误。`padParams` 关闭自定义填充值，左右额外填充数设为 0；GM→UB 的搬运会在 UB 侧将不足 32 字节的部分补成完整数据块，**加法只使用前 `currentLength` 个有效元素**。UB→GM 的 `DataCopyPad` 只写回指定的 4004 字节，不把用于对齐的 dummy 数据写出。

尾块需要在 UB 中容纳对齐后的 `⌈4004/32⌉×32=4032` 字节，即 1008 个 `float` 位置；第七节保留的 `Alloc<float, blockLength>()` 提供 2048 个位置，因此足够。本练习为了先验证功能，让完整块和尾块都走 `DataCopyPad`；暂不比较它与完整块使用 `DataCopy` 的性能。

## 8.2 让同一个程序接收不同长度

在 `add.asc` 的头文件区增加 `#include <cstdlib>`。将 `main` 里的固定总长改为可从命令行读取的非负十进制数；不传参数时默认测试 15337：

```cpp
uint32_t totalLength = argc > 1
    ? static_cast<uint32_t>(std::strtoul(argv[1], nullptr, 10))
    : 7 * 2048 + 1001;
```

`main` 其余构造 `x`、`y`、`golden` 的代码保持原样。在 `kernel_add` 的函数体开头加入：

```cpp
if (x.empty()) {
    return {};
}
```

空向量直接由 Host 返回，不启动 Kernel，也不调用零字节的 ACL 分配。非空向量沿用第七节的 `numBlocks = (totalLength + blockLength - 1) / blockLength`。这个命令行入口只用于本练习，测试参数需为 `uint32_t` 范围内的非负十进制数；生产代码应另外检查解析错误及两个输入的长度是否相同。

## 8.3 重新编译并跑不同边界

```bash
source "$HOME/Ascend/cann-9.2.0/set_env.sh"
for mode in cpu sim; do
  cmake -S "$anyWork/sample" -B "$anyWork/build-$mode" \
    -DCMAKE_ASC_RUN_MODE="$mode" -DCMAKE_ASC_ARCHITECTURES=dav-2201
  cmake --build "$anyWork/build-$mode" -j 4
  for n in 0 1 7 8 9 1001 2048 2049 15337; do
    mkdir -p "$anyWork/run-$mode-$n"
    (cd "$anyWork/run-$mode-$n" && "$anyWork/build-$mode/demo" "$n")
  done
done
```

本机 CANN 9.2.0 的临时副本验证结果如下；每个模式均编译得到 `[100%] Built target demo`：

| 总长 `L` | Block 数 | 最后一个 Block 的有效元素 | 尾部字节数 | CPU | SIM |
| ---: | ---: | ---: | ---: | --- | --- |
| 0 | 0 | 无，Host 直接返回 | 0 | `test pass!` | `test pass!` |
| 1、7、8、9 | 1 | 分别为 1、7、8、9 | 4、28、32、36 | 均 `test pass!` | 均 `test pass!` |
| 1001 | 1 | 1001 | 4004 | `test pass!` | `test pass!` |
| 2048 | 1 | 2048 | 8192 | `test pass!` | `test pass!` |
| 2049 | 2 | 1 | 4 | `test pass!` | `test pass!` |
| 15337 | 8 | 1001 | 4004 | `test pass!` | `test pass!` |

`VerifyResult` 比较整个输出向量，终端只打印前 20 项。因此，`15337` 的 `test pass!` 也覆盖了第 8 个 Block 的 1001 个结果。此处的“任意长度”是指**在样例可分配内存、`uint32_t` 计数和可启动 Block 范围内，不再要求尾长满足 32 字节对齐**；没有据此验证无限大的向量或实卡性能。

# 参考资料

1. [指定提交的 add.asc](https://gitcode.com/cann/cann-samples/blob/3a85e824537feaef646bd89eb4d0e799cc2886b5/Samples/0_Introduction/01_simd_cpp_api/01_add/add/add.asc)
2. [同一提交的 CMakeLists.txt](https://gitcode.com/cann/cann-samples/blob/3a85e824537feaef646bd89eb4d0e799cc2886b5/Samples/0_Introduction/01_simd_cpp_api/01_add/add/CMakeLists.txt)
3. [同一提交的 Add 样例 README](https://gitcode.com/cann/cann-samples/blob/3a85e824537feaef646bd89eb4d0e799cc2886b5/Samples/0_Introduction/01_simd_cpp_api/01_add/add/README.md)
4. [昇腾官方：`InitSocState`](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/tool_interface/system_init/InitSocState.md)
5. [昇腾官方：`LocalMemAllocator`](https://www.hiascend.com/doc_center/source/en/CANNCommunityEdition/900/API/ascendcopapi/atlasascendc_api_07_00096.html)
6. [昇腾官方：CMAKE_ASC_RUN_MODE 的三种模式](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/programug/Ascendcopdevg/docs/en/guide/programming_guide/compilation_and_execution/operator_compilation/ai_core_operator_compilation_basic_usage.md)（模式机制参考）
7. [昇腾官方：`SetGlobalBuffer` 接口](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/data_structures/GlobalTensor/SetGlobalBuffer.md)
8. [昇腾官方：`Alloc` 接口](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/resource_management/LocalMemAllocator/Alloc.md)
9. [昇腾官方：`GlobalTensor` 简介](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/data_structures/GlobalTensor/globaltensor_introduction.md)
10. [昇腾官方：`LocalTensor` 简介](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/data_structures/LocalTensor/localtensor_introduction.md)
11. [昇腾官方：`LocalMemAllocator` 简介](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/resource_management/LocalMemAllocator/localmemallocator_introduction.md)
12. [昇腾官方：GM 与 UB 连续搬运 `DataCopy`](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/memory_vector_compute/data_move/DataCopy_GMAndUB_continuous.md)
13. [昇腾官方：向量加法 `Add` 接口](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/basic_api/memory_vector_compute/basic_arithmetic/Add.md)
14. [Ascend C 官方：GM→UB 非对齐搬运 `DataCopyPad`](https://asc.gitcode.com/api/SIMD-API/basic_api/memory_vector_compute/data_move/DataCopyPad_GMToUB.html)
15. [Ascend C 官方：UB→GM 非对齐搬运 `DataCopyPad`](https://asc.gitcode.com/api/SIMD-API/basic_api/memory_vector_compute/data_move/DataCopyPad_UBToGM.html)
