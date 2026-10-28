
> 时间：2026-10-25  
> 版本：CANN 9.2.0 
> 目标：在没有 NPU 的 WSL 环境中，用同一份 Hello World 源码依次尝试 `npu`、`cpu`、`sim`，区分“编译成功”与“Kernel 真正运行成功”，打通第一个 CANN 项目。

# 一、选择最小样例与验证边界

本文使用 `cann-samples` 的 `Samples/0_Introduction/01_simd_cpp_api/00_quickstart/hello_world/hello_world.asc`〔1〕。该文件同时包含 Kernel 和 Host 的启动代码，配套目录已有 `CMakeLists.txt`〔2〕，无需先改源码。

本机环境为 Ubuntu 24.04.3 LTS（WSL、x86_64）、普通用户 `lambertt`、**CANN Toolkit 9.2.0 与 Ascend910B ops 9.2.0**。目标设备是 Ascend910B（A2），因此编译参数选 `dav-2201`。本机没有 `/dev/davinci0`；`npu` 模式只能观察无设备时的失败行为，`cpu` 与 `sim` 可以在本机运行，但都不能提供真实 NPU 性能数据。CPU 调试模式的官方说明见〔3〕。

# 二、准备 WSL 和 CANN

以下命令在 Ubuntu WSL 终端执行。平时使用普通用户；只有安装系统依赖时使用 `sudo`。把源码和构建目录放在 WSL 的 Linux 文件系统中。

```bash
sudo apt-get update
sudo apt-get install -y gcc g++ make cmake git python3 python3-pip gdb
```

本机曾遇到 apt 直连超时，当时安装依赖使用了本地代理 `127.0.0.1:7897`。该端口只是本机配置，不是 CANN 的要求。

## 2.1 下载并校验 CANN 9.2.0 安装包

从官方 9.2.0 下载页〔4〕选择 `linux-x86_64` 的 Toolkit 和 **910B ops**，两者版本保持一致。以下命令由 `lambertt` 普通用户执行，不加 `sudo`：

```bash
mkdir -p "$HOME/Ascend/install-cache"
cd "$HOME/Ascend/install-cache"

wget -c 'https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/CANN%209.2.0/Ascend-cann-toolkit_9.2.0_linux-x86_64.run'
wget -c 'https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/CANN%209.2.0/Ascend-cann-910b-ops_9.2.0_linux-x86_64.run'

bash ./Ascend-cann-toolkit_9.2.0_linux-x86_64.run --check
bash ./Ascend-cann-910b-ops_9.2.0_linux-x86_64.run --check
```

两个包的 `--check` 都返回 `SHA256 checksums are OK`，说明安装器的内部完整性校验通过；本次没有另外记录官网数字签名的独立校验结果。

## 2.2 安装 9.2.0 CANN

```bash
cd "$HOME/Ascend/install-cache"
bash ./Ascend-cann-toolkit_9.2.0_linux-x86_64.run --install
bash ./Ascend-cann-910b-ops_9.2.0_linux-x86_64.run --install
```

如果是**全新环境**而不是从 9.1.0 升级，应按官方非昇腾设备安装说明〔5〕使用对应包的 `--install` 流程。否则应该使用  `--upgrade` 流程，参考说明〔6〕。

## 2.3 安装完后检查

本机升级完成后，查看安装版本，回显 9.2.0：

```bash
cat "$HOME/Ascend/cann/x86_64-linux/ascend_toolkit_install.info"
cat "$HOME/Ascend/cann/x86_64-linux/ascend_ops_install.info"
```

# 三、取得固定版本的样例

本次运行基于已验证的 `cann-samples` 提交 `3a85e824537feaef646bd89eb4d0e799cc2886b5`。固定提交可以避免仓库 `master` 后续变化影响复现。

```bash
mkdir -p "$HOME/projects"
cd "$HOME/projects"
git clone https://gitcode.com/cann/cann-samples.git
cd cann-samples
git checkout 3a85e824537feaef646bd89eb4d0e799cc2886b5
```

如果 `~/projects/cann-samples` 已存在，直接进入该仓库并核对 `git rev-parse HEAD` 即可。

# 四、先读懂 `hello_world.asc`

源码中真正运行的 Kernel 很短：

## 4.1 Kernel 侧
`hello_world` 是由 `__global__ __vector__` 标记的入口，函数体调用 `printf` 输出 `Hello World!!!`。

## 4.2 Host 侧
- `main` 依次初始化、选择设备 0、创建 stream、使用 `hello_world<<<8, 0, stream>>>()` 启动 8 个 block、同步等待、销毁 stream 并释放运行资源。
- `<<<...>>>` 是 Kernel 启动语法；CMake 根据 `CMAKE_ASC_RUN_MODE` 选择对应的编译、链接和运行路径。

这份样例没有输入张量、UB 缓冲、Tiling 或结果精度比对。因此它只负责验证“能编译、能启动 Kernel、能看到输出”，是最简单的一份 Ascend C 学习代码。

# 五、先区分 `npu`、`cpu`、`sim` 三种模式

`CMAKE_ASC_RUN_MODE` 有三个可选值：`npu`（默认）、`cpu`、`sim`。CMake 会据此调整编译选项与链接库〔7〕。三个模式都使用本文的 `dav-2201` 目标架构参数，但“在 x86 CPU 上执行”不等于“运行方式相同”。

| 模式    | Kernel 在哪里运行        | 是否需要真实 NPU | 主要用途                  |
| ----- | ------------------- | ---------- | --------------------- |
| `npu` | Ascend 设备上的 AI Core | 需要         | 上板验证功能，并在真实硬件上测性能     |
| `cpu` | Host CPU 上的调试运行时    | 不需要        | 先检查 Kernel 功能、打印与调试流程 |
| `sim` | Host 上的 NPU 架构仿真器   | 不需要        | 模拟目标架构执行，查看仿真日志和执行行为  |

`cpu` 是 CPU 域孪生调试；`sim` 是 NPU 仿真，两者都在这台 WSL 的 CPU 上运行，但编译产物、运行时和输出不同。`sim` 的模型耗时与 tick 是仿真结果，**不能当成 Ascend910B 上板性能**。这份 Hello World 只有打印，没有数值结果，因此这里只能验证启动与输出，不能声称完成算子精度测试。

# 六、按 `npu` → `cpu` → `sim` 依次运行

本次样例仓库 HEAD 为 `3a85e824537feaef646bd89eb4d0e799cc2886b5`；Hello World 源文件没有本地修改。先在 **WSL 终端**准备公共变量，后面的命令在同一 Shell 依次执行。三个模式各用自己的构建目录，避免复用 CMake 缓存。运行时先进入 WSL 的 `/tmp` 子目录，仿真日志等信息暂时会写入该暂存区。

```bash
source "$HOME/Ascend/cann/set_env.sh"
sample="$HOME/projects/cann-samples/Samples/0_Introduction/01_simd_cpp_api/00_quickstart/hello_world"
work="/tmp/hello-world-3a85-20261025"
git -C "$HOME/projects/cann-samples" rev-parse HEAD
mkdir -p "$work"
```

`-S` 指定源码目录，`-B` 指定构建目录，`cmake --build` 只负责编译；执行 `demo` 才会真正启动 Host 和 Kernel 逻辑。以下输出来自本机 CANN 9.2.0 实测，日志中的 PID、耗时和 Block 打印顺序可能变化。

## 6.1 `npu`：编译成功，但 WSL 没有设备

```bash
# -B 可以任意指定目标构建路径
cmake -S "$sample" -B "$work/build-npu" \
  -DCMAKE_ASC_RUN_MODE=npu \
  -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$work/build-npu" -j 4
mkdir -p "$work/run-npu"
(cd "$work/run-npu" && timeout 30s "$work/build-npu/demo")
echo "exit=$?"
```

CMake 配置与编译完成，末尾出现 `[100%] Built target demo`。运行时**没有**任何 `[Block (x/8)]` 输出，终端出现：

```text
[ERROR] [Bisheng] Get kernel function failure! ret 107002
exit=0
```

运行日志 `~/ascend/log/run/plog/` 还记录了 `aclInitImpl: init SoC version failed`、设备 0 无效（`Expected value: [0, 0)`）、`aclrtSetDevice` 失败，随后创建 stream 和获取 Kernel 函数也失败。本机没有 `/dev/davinci0`。因此 **`npu` 模式在这台 WSL 上未成功执行 Kernel**；“编译成功”只表示生成了可执行文件。

这里的 `exit=0` **不是成功证据**：当前样例的 Host `main` 没有检查 `aclInit`、`aclrtSetDevice` 等调用的返回值，最后无条件 `return 0`。判断这次运行时要同时看错误日志以及 8 个 Block 是否打印。

## 6.2 `cpu`：CPU 孪生运行成功

```bash
cmake -S "$sample" -B "$work/build-cpu" \
  -DCMAKE_ASC_RUN_MODE=cpu \
  -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$work/build-cpu" -j 4
mkdir -p "$work/run-cpu"
(cd "$work/run-cpu" && timeout 30s "$work/build-cpu/demo")
echo "exit=$?"
```

编译成功；实测打印 `[Block (0/8)]` 至 `[Block (7/8)]` 各一次，共 8 行 `Hello World!!!`，随后 `CORE_0` 至 `CORE_7` 均显示 `exit success!`，进程退出码为 0。打印顺序不固定，例如本次先打印的是 Block 2、Block 5、Block 1。这个模式没有 `[AIV Block x/8]` 前缀，说明看到的是 CPU 调试运行时的输出。

```txt
[ascendc_acl_stub.cpp][check_interface:330]
[Block (5/8)]: Hello World!!!
[Block (0/8)]: Hello World!!!
[Block (1/8)]: Hello World!!!
[Block (2/8)]: Hello World!!!
[Block (6/8)]: Hello World!!!
[Block (4/8)]: Hello World!!!
[Block (3/8)]: Hello World!!!
[Block (7/8)]: Hello World!!!
[SUCCESS][CORE_0][pid 38503] exit success!
[SUCCESS][CORE_1][pid 38505] exit success!
[SUCCESS][CORE_2][pid 38507] exit success!
[SUCCESS][CORE_3][pid 38509] exit success!
[SUCCESS][CORE_4][pid 38510] exit success!
[SUCCESS][CORE_5][pid 38513] exit success!
[SUCCESS][CORE_6][pid 38515] exit success!
[SUCCESS][CORE_7][pid 38517] exit success!
```

## 6.3 `sim`：NPU 架构仿真运行成功

```bash
cmake -S "$sample" -B "$work/build-sim" \
  -DCMAKE_ASC_RUN_MODE=sim \
  -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build "$work/build-sim" -j 4
mkdir -p "$work/run-sim"
(cd "$work/run-sim" && timeout 60s "$work/build-sim/demo")
echo "exit=$?"
```

配置阶段显示 `ASC Simulator enabled: .../tools/simulator/dav_2201/lib`，编译成功。运行时仿真模型初始化，8 个 Block 均打印，最后显示模型正常停止，退出码为 0。关键输出如下：

```text
[AIV Block 0/8] [Block (0/8)]: Hello World!!!
...
[AIV Block 7/8] [Block (7/8)]: Hello World!!!
Model RUN TIME: 983.724 ms
[INFO] Model stopped successfully.
exit=0
```

前面的 `[AIV Block x/8]` 是仿真运行时添加的前缀，后面的 `[Block (x/8)]` 来自这版 `hello_world.asc` 内的 `printf`。`Model RUN TIME` 是本次仿真的一个输出值，不是硬件计时，也不宜用来与 `cpu` 模式作性能比较。仿真器可能在**当前工作目录**生成 `core*.dump`、`profile_*.toml`；这里它们留在 `$work/run-sim`。
# 七、总结

在同一份源码、CANN 9.2.0 和 `dav-2201` 配置下：`npu` **可编译但无设备、Kernel 未运行**；`cpu` **在 CPU 调试运行时成功打印 8 个 Block**；`sim` **在 NPU 仿真器中成功打印 8 个 Block**。本机完成了无 NPU 环境下的功能链路验证，真实 Ascend910B 上板运行与性能测量仍需在有 NPU 的环境进行。

# 参考资料

1. [cann-samples 指定提交的 hello_world.asc](https://gitcode.com/cann/cann-samples/blob/3a85e824537feaef646bd89eb4d0e799cc2886b5/Samples/0_Introduction/01_simd_cpp_api/00_quickstart/hello_world/hello_world.asc)
2. [同一提交的 CMakeLists.txt](https://gitcode.com/cann/cann-samples/blob/3a85e824537feaef646bd89eb4d0e799cc2886b5/Samples/0_Introduction/01_simd_cpp_api/00_quickstart/hello_world/CMakeLists.txt)
3. [Ascend C CPU 域孪生调试说明](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/programug/Ascendcopdevg/docs/en/guide/programming_guide/debug_and_tuning/functional_debug/cpu_domain_twin_debugging.md)（机制参考；链接对应 9.1.X 版本文档）
4. [CANN 9.2.0 官方下载页](https://www.hiascend.com/cann/download?versionId=810&ids=d803%2Ch0502%2Ch0602%2Ch0704&currentTab=0)
5. [CANN 9.2.0 在非昇腾设备上安装](https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/920/softwareinst/instg/instg_0054.html)
6. [CANN 9.2.0 升级说明](https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/920/softwareinst/instg/instg_0030.html)
7. [Ascend C 编译参数与三种运行模式](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/programug/Ascendcopdevg/docs/en/guide/programming_guide/compilation_and_execution/operator_compilation/ai_core_operator_compilation_basic_usage.md)（模式定义参考；链接对应 9.1.X 版本文档）

