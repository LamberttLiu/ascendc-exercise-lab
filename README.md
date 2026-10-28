# Ascend C 练习仓库

基于 CANN 9.2.0、Ascend 910B（A2，`dav-2201`）的 WSL CPU 孪生和 SIM 仿真练习。

| 工程 | 内容 | 默认输入长度 |
| --- | --- | ---: |
| [00_basic](vector_add/00_basic/README.md) | 上游原始 Vector Add；8 个 Block 均分 | 16384 |
| [01_uneven_split](vector_add/01_uneven_split/README.md) | 前 7 个 Block 各 2048，尾块 1000 | 15336 |
| [02_arbitrary_length](vector_add/02_arbitrary_length/README.md) | 用 DataCopyPad 处理任意对齐状态的尾块 | 15337 |

每个目录均有完整的 `add.asc`、独立的 `CMakeLists.txt` 和运行说明。历史练习 `vector_add_cpu_debug` 保留在仓库中。

基础版取自 [cann-samples 固定提交](https://gitcode.com/cann/cann-samples/blob/3a85e824537feaef646bd89eb4d0e799cc2886b5/Samples/0_Introduction/01_simd_cpp_api/01_add/add/add.asc)，进阶版在其上修改。上游许可见 [vector_add/LICENSE](vector_add/LICENSE)。

## 配套教程

- [01-第一个 Hello World 的 AscendC 程序](docs/01-%E7%AC%AC%E4%B8%80%E4%B8%AA%20Hello%20World%20%E7%9A%84%20AscendC%20%E7%A8%8B%E5%BA%8F.md)
- [02-从 Vector Add 的代码样例编译到 Kernel 实现分析](docs/02-%E4%BB%8E%20Vector%20Add%20%E7%9A%84%E4%BB%A3%E7%A0%81%E6%A0%B7%E4%BE%8B%E7%BC%96%E8%AF%91%E5%88%B0%20Kernel%20%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90.md)
