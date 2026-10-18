# `cann-samples/vector_add` CPU 孪生测试

本目录的 `CMakeLists.txt` 在本机作为 `~/projects/cann-samples/Samples/0_Introduction/vector_add/cpu_debug/CMakeLists.txt` 使用，目标直接编译上一级的 `main.asc`，与 NPU 目标共用同一份 `add_kernel`。适用环境是 CANN 9.1.0、`dav-2201`、无 NPU 的 WSL。

## 运行

```bash
source "$HOME/Ascend/cann/set_env.sh"
cd "$HOME/projects/cann-samples/Samples/0_Introduction/vector_add/cpu_debug"
cmake -S . -B build -DCMAKE_ASC_RUN_MODE=cpu -DCMAKE_ASC_ARCHITECTURES=dav-2201
cmake --build build -j 4
./build/vector_add_cpu_debug
```

成功时，8 个 `CORE_*` 进程退出成功，最后输出 `Vector add completed successfully!`。

## 与 NPU 入口的区别

`main.asc` 通过 `ASCENDC_CPU_DEBUG` 跳过 NPU Profiling 接口。CPU 入口采用 8 个 Vector 核和 192 KiB UB 预算作为**测试输入**，代入与 NPU 入口相同的分块公式；这些值并非本机检测到的硬件参数，也不是 910B 规格声明。NPU 入口仍通过 `PlatformAscendCManager` 获取平台信息。当前 CPU 用例验证 Kernel 的 CopyIn、Add、CopyOut 和这些给定参数下的 Host 分块、精度，不验证实际设备的资源查询、性能或算子包部署。

原仓库的 NPU 构建仍使用根目录 CMake：

```bash
cd "$HOME/projects/cann-samples"
cmake -S . -B build -DNPU_ARCH=dav-2201
cmake --build build --target vector_add -j 4
```
