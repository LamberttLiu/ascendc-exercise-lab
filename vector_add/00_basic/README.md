# 00 Basic：原始 Vector Add

`add.asc` 与 `CMakeLists.txt` 复制自 cann-samples 提交 `3a85e824537feaef646bd89eb4d0e799cc2886b5` 的 `Samples/0_Introduction/01_simd_cpp_api/01_add/add`。8 个 Block 各处理 2048 个 `float`，总长 16384。Kernel 的三段数据流是 GM→UB、Add、UB→GM；源码中的 `#if 0` 调试输出保持关闭。

## WSL 中编译和运行

```bash
source "$HOME/Ascend/cann-9.2.0/set_env.sh"
lab="$HOME/github-codes/ascendc-exercise-lab/vector_add/00_basic"
work="$(mktemp -d /tmp/ascendc-add-basic-XXXXXX)"
for mode in cpu sim; do
  cmake -S "$lab" -B "$work/build-$mode" \
    -DCMAKE_ASC_RUN_MODE="$mode" -DCMAKE_ASC_ARCHITECTURES=dav-2201
  cmake --build "$work/build-$mode" -j 4
  mkdir -p "$work/run-$mode"
  (cd "$work/run-$mode" && "$work/build-$mode/demo")
done
```

两种模式都应输出 `test pass!`。构建及运行产物位于 Linux 的临时目录，不写进仓库。许可证见 [../LICENSE](../LICENSE)。
