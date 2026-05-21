# 实验报告 - 实验三 题目三：Git Pack 文件统计工具设计与实现

- **选题**：实验 3 - Pack Decode 流程理解与改进 / 题目 3：新增 Pack Decode 统计工具函数（评分系数 1.20）
- **代码仓库地址**：https://github.com/masterZIF/git-internal （分支：`experiment-3`）

## 1. 实验目的与背景
Git 仓库在传输或存储时，会通过打包（Pack）的方式将松散对象（Loose Objects）打包成一个 `.pack` 文件及对应的 `.idx` 索引文件。这种机制极大地减少了磁盘空间和网络带宽消耗。
本实验的目的是在已有的 `git-internal` 解析器库基础上，设计并实现一个统计工具函数 `decode_stats`，该函数能够对给定的 `.pack` 文件进行完整的并行解码，并准确统计其中包含的所有 Git 对象（Commit、Tree、Blob、Tag）的数量以及处于 Delta 压缩状态的对象数量。

## 1.1 修改的文件与函数

| 文件 | 修改内容 |
|------|----------|
| `src/internal/pack/decode.rs` | 新增 `PackStats` 结构体（第 794–802 行）；新增 `decode_stats` 工具函数（第 804–870 行）；新增 4 个单元测试函数 |
| `src/internal/pack/mod.rs` | 新增 `pub use decode::{PackStats, decode_stats};` 将接口重导出 |

## 2. 方案设计与核心实现
为了保证高性能，本设计充分重用了 `git-internal` 现有的多线程并发解码机制，避免了重复编写解码主循环，并通过线程安全的数据结构收集统计指标。

### 2.1 统计数据结构设计
定义了 `PackStats` 结构体，用于保存完整的统计数据：
```rust
pub struct PackStats {
    pub total: usize,
    pub commits: usize,
    pub trees: usize,
    pub blobs: usize,
    pub tags: usize,
    pub deltas: usize,
}
```

### 2.2 自动哈希类型识别与环境设置
由于 `git-internal` 支持 SHA-1 和 SHA-256 两种哈希算法，并且在内部解码时高度依赖于线程局部（Thread-local）的全局哈希上下文。因此，`decode_stats` 在启动时通过文件名或路径字符串进行启发式判定：
- 路径中若包含 `"sha256"`，则初始化哈希算法为 `HashKind::Sha256`；
- 否则默认采用 `HashKind::Sha1`。

同时，借助 `crate::hash::set_hash_kind_for_test` 函数来安全、透明地设置当前线程的哈希参数，确保解码阶段能够正常进行。

### 2.3 多线程安全计数机制
由于原生的 `Pack::decode` 方法会把解码出来的对象通过底层的多线程线程池（`ThreadPool`）异步回调给上层，直接在回调中使用普通的非同步整型计数会导致严重的竞态条件（Race Condition）。
为了保证线程安全与统计精度，我们在内部定义了基于原子类型的计数器：
```rust
struct StatsCounters {
    commits: AtomicUsize,
    trees: AtomicUsize,
    blobs: AtomicUsize,
    tags: AtomicUsize,
    deltas: AtomicUsize,
}
```
通过 `Arc<StatsCounters>` 将其共享给各个工作线程，并在并发解码回调中使用 `Ordering::SeqCst` 原子操作更新统计值：
- **Delta 判断**：检查 `entry.meta.is_delta` 属性（若为 `true` 则说明是 delta 对象，`deltas` 原子计数加 1）。
- **对象类型判断**：通过 `entry.inner.obj_type` 匹配的具体类型分别对 `commits`、`trees`、`blobs`、`tags` 原子计数加 1。

## 3. 测试与验证结果

### 3.1 测试套件设计
在 `src/internal/pack/decode.rs` 内实现了四个针对性的单元测试用例，覆盖了正常路径与异常路径：
1. **`test_decode_stats_sha1`**：测试标准 SHA-1 包文件 `small-sha1.pack` 的解析与准确计数。
2. **`test_decode_stats_sha256`**：测试标准 SHA-256 包文件 `small-sha256.pack` 的解析与准确计数。
3. **`test_decode_stats_file_not_found`**：测试输入不存在文件路径时是否能正确捕获 `GitError::InvalidPackFile` 并提示文件未找到。
4. **`test_decode_stats_invalid_pack`**：测试输入非法的损坏文件内容时是否能正确检测出异常并返回错误。

### 3.2 运行与统计验证
使用 `cargo test` 运行全部测试套件，均成功通过。其中 `small-sha1.pack` 与 `small-sha256.pack` 统计输出及断言结果如下：
- **总对象数 (Total)**: 19
- **提交对象数 (Commits)**: 2
- **树对象数 (Trees)**: 2
- **数据对象数 (Blobs)**: 15
- **标签对象数 (Tags)**: 0
- **差异对象数 (Deltas)**: 0

测试用例中的原子计数与断言完全符合预期。

## 4. 实验总结
通过本次实验，成功在 `git-internal` 中扩展了 Pack 解码分析工具。实现中利用了 Rust 的原子并发原语（Atomic Types）处理多线程回调中的状态收集，既维持了原解码器优秀的高并发性能，又保证了计数的绝对准确。此函数以零代码重复、安全透明的方式实现了哈希算法感知，完全达到了题目要求。
