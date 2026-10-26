# 02 Arbitrary Length：非 32 字节对齐尾块

从不均匀切分版修改。Kernel 使用 `DataCopyPad` 完成 GM→UB 与 UB→GM 搬运，`DataCopyExtParams.blockLen` 按**字节**设置；`Add` 的 `currentLength` 仍按**元素**设置。默认总长 `7×2048+1001=15337`，最后 1001 个 `float` 占 4004 字节，不能被 32 整除。完整 Block 与尾块都使用 `DataCopyPad`，此版侧重功能验证，未做性能调优。

`demo` 接受可选的非负十进制长度参数。仅将参数用于 `uint32_t` 范围内的有效数值；这个入门样例没有对非法字符串、负数或溢出做完整校验。长度 0 由 Host 直接返回，不启动 Kernel。

## WSL 中编译和运行

```bash
source "$HOME/Ascend/cann-9.2.0/set_env.sh"
lab="$HOME/github-codes/ascendc-exercise-lab/vector_add/02_arbitrary_length"
work="$(mktemp -d /tmp/ascendc-add-anylen-XXXXXX)"
for mode in cpu sim; do
  cmake -S "$lab" -B "$work/build-$mode" \
    -DCMAKE_ASC_RUN_MODE="$mode" -DCMAKE_ASC_ARCHITECTURES=dav-2201
  cmake --build "$work/build-$mode" -j 4
  for n in 0 1 7 8 9 1001 2048 2049 15337; do
    mkdir -p "$work/run-$mode-$n"
    (cd "$work/run-$mode-$n" && "$work/build-$mode/demo" "$n")
  done
done
```

每个长度均应输出 `test pass!`。可在 `run-$mode-$n` 中找到该次运行的仿真产物；许可证见 [../LICENSE](../LICENSE)。
