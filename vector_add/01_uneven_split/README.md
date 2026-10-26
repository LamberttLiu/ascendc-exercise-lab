# 01 Uneven Split：尾核工作量不同

从基础版修改：`main` 的输入总长变为 `7×2048+1000=15336`；Host 计算 `numBlocks=ceil(totalLength/blockLength)=8` 并把总长传给 Kernel。Kernel 按 `block_idx` 计算 `offset`，再用 `currentLength=min(2048,totalLength-offset)` 限定 GM 片段和三次数据操作。

前 7 个 Block 各处理 2048 个 `float`；最后一个处理 1000 个，即 4000 字节，仍满足连续 `DataCopy` 的 32 字节对齐要求。UB 中三个 `LocalTensor` 依然各保留 2048 元素容量。

## WSL 中编译和运行

```bash
source "$HOME/Ascend/cann-9.2.0/set_env.sh"
lab="$HOME/github-codes/ascendc-exercise-lab/vector_add/01_uneven_split"
work="$(mktemp -d /tmp/ascendc-add-uneven-XXXXXX)"
for mode in cpu sim; do
  cmake -S "$lab" -B "$work/build-$mode" \
    -DCMAKE_ASC_RUN_MODE="$mode" -DCMAKE_ASC_ARCHITECTURES=dav-2201
  cmake --build "$work/build-$mode" -j 4
  mkdir -p "$work/run-$mode"
  (cd "$work/run-$mode" && "$work/build-$mode/demo")
done
```

两种模式都应输出 `test pass!`；Host 的 `std::equal` 比较全部 15336 个结果。许可证见 [../LICENSE](../LICENSE)。
