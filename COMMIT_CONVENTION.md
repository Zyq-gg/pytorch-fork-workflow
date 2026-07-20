# Commit Convention

提交记录规范（commit log）
要求 commit 标题前仍旧统一带 [das] 标记，以区别官方提交。因此后续新增提交的实际写法统一为：
[das] type(scope): subject
其中 scope 仍使用模块名或主题，例如 aten、inductor、build、perf、runtime、test、merge。同时，body 仍建议保留 Why / What / Impact 结构。
推荐示例：
- [das] feat(aten): add rmsnorm kernel
- [das] fix(inductor): handle missing rocm combo kernel grid metadata
- [das] perf(aten): optimize simd memory access in swap2d
- [das] build(runtime): enable aicc via USE_AICC
- [das] fix(merge): resolve merge-time compile failure
因此，下面各类 feat / fix / perf / refactor / docs / test / build / ci / chore 的说明，仍然保留其语义分类用途；但实际标题格式，以 [das] type(scope): subject 为准。
必须满足的 commit 需求
- commit 标题统一使用 [das] type(scope): subject
- type 仅建议使用 feat / fix / perf / refactor / docs / test / build / ci / chore
- scope 使用模块名或整改主题，如 aten、torch、inductor、runtime、build、readme、license、notice
- subject 必须小写、动词开头，尽量不超过 72 字符
- 重要提交应补 Why / What / Impact body
- fix 提交必须写原因、影响范围和修复点
- perf 提交必须写性能指标，如 throughput、latency、benchmark 前后对比
- 如有破坏性变更，必须写 BREAKING CHANGE:
- 历史提交暂不重写，但从当前开源整改分支开始，后续新增提交都按这套规范执行
type 逐项定义与真实示例
feat：表示新增功能、新支持、新能力。适用于新算子、新 kernel、新后端支持、新 API 或新能力开关。判断标准是：原来仓库做不到，现在可以做到。
- add rmsnorm 建议改写为 [das] feat(aten): add rmsnorm kernel
- support fuse group norm and fuse celu gn 建议改写为 [das] feat(aten): add fused group norm and celu group norm kernels
- [das-scaled] Add FP8 scaled_mm GEMM support using hipblaslt backend. 建议改写为 [das] feat(aten): add fp8 scaled_mm support via hipblaslt backend
fix：表示修 bug、修错误行为。适用于编译失败、运行时报错、结果错误、fallback 错误、metadata 错误或 crash。判断标准是：原本应该能工作，但现在是坏的。
- [das-inductor] Handle missing ROCm combo kernel grid metadata 建议改写为 [das] fix(inductor): handle missing rocm combo kernel grid metadata
- [das-merge]fix compile bug 建议改写为 [das] fix(merge): resolve merge-time compile failure
- fix lightop import error 建议改写为 [das] fix(aten): resolve lightop import error
perf：表示性能优化。适用于吞吐提升、延迟下降、显存优化、kernel launch 开销降低或 autotune 选择优化。判断标准是：这次提交的核心价值是更快或更省，而不是修正确性。
- [das-perf] add simd mem access optimize and swap2d function 建议改写为 [das] perf(aten): optimize simd memory access in swap2d
- [das-inductor] Use FMA for HIP add alpha 建议改写为 [das] perf(inductor): use fma in hip add alpha lowering
- [das-jit] Fallback TensorExpr on mismatched device or oversized launch 如以性能回退策略为主，可改写为 [das] perf(jit): fall back from tensorexpr on oversized launches
refactor：表示重构、实现整理、非功能性调整。适用于代码结构调整、实现替换但不改行为、公开文案对齐、逻辑清理。判断标准是：功能和结果不变，主要变化在实现或结构。
- 当前整改提交 34f0618e73a 的原始标题为 refactor(runtime): align HCU and ROCm wording；按当前口径应写为 [das] refactor(runtime): align HCU and ROCm wording
- [das-bilinear] refactor upsample_bilinear2d backward CUDA kernel ... 建议改写为 [das] refactor(aten): restructure upsample bilinear backward kernel mapping
- [das-jit]Fallback to eager when grid *block >uint32.max() 如不属于 bug 修复而是路径整理，可改写为 [das] refactor(jit): route oversized launches to eager fallback
docs：表示文档修改。适用于 README、docs/source、说明文档、对外解释性内容。判断标准是：只改变文档理解，不改变代码行为。
- 当前整改提交 99b90ef8902 的原始标题为 docs(readme): add upstream attribution for hygon pytorch；按当前口径应写为 [das] docs(readme): add upstream attribution for hygon pytorch
- [das] docs(open-source): add bilingual compliance docs 是符合当前口径的写法
- [das] docs(open-source): pin upstream references to PyTorch 2.12.0 也是合规示例
test：表示测试相关修改。适用于新增测试、修测试、skip/xfail、更新 golden 或扩覆盖率。判断标准是：主要改动发生在测试文件和测试逻辑。
- [das-hipblasltgg] add some tests for hipblasltgg. 建议改写为 [das] test(inductor): add hipblasltgg coverage
- [das-test] fix test_get_item_3d NameError 建议改写为 [das] test(torch): fix test_get_item_3d nameerror
- [das-inductor] Skip PDL subprocess tests on CPU 建议改写为 [das] test(inductor): skip pdl subprocess tests on cpu
build：表示构建、编译、打包、依赖装配。适用于 setup.py、CMakeLists.txt、编译选项、toolchain、dependency pin、submodule bump。判断标准是：主要影响“怎么编出来、怎么打包出来”。
- [das-build] Enable aicc compiler via USE_AICC=1 environment variable 建议改写为 [das] build(runtime): enable aicc via USE_AICC
- Bump gloo submodule to include native CMake HIP support 建议改写为 [das] build(gloo): bump submodule for native cmake hip support
- [das-inductor] Prefer aillvm clang for ROCm multiarch 建议改写为 [das] build(inductor): prefer aillvm clang for rocm multiarch
ci：表示持续集成和流水线修改。适用于 GitLab CI、GitHub Actions、runner、workflow、shard、job pin。判断标准是：只影响 CI 任务编排和环境，不影响仓库产品代码。
- [Windows CI] Pin openssl=3.5.6 to fix Windows cert-store ASN.1 failure 建议改写为 [das] ci(windows): pin openssl to fix cert-store asn.1 failure
- [ROCm][CI] Decrease ROCm shards of trunk.yml 建议改写为 [das] ci(rocm): reduce trunk.yml shard count
- Publish docs preview from a standalone docs-build workflow 可写为 [das] ci(docs): publish docs preview from standalone workflow
chore：表示仓库维护、开源合规、元材料整理。适用于 LICENSE、NOTICE、SPDX、attribution、仓库级元信息。判断标准是：不是功能、不是 bug、不是性能、不是正文文档，而是维护和合规动作。
- 当前整改提交 c1aa20c4104 的原始标题为 chore(license): add hygon bsd headers to new source files；按当前口径应写为 [das] chore(license): add hygon bsd headers to new source files
- 当前整改提交 cf506f1f0b9 的原始标题为 chore(notice): preserve third-party attribution materials；按当前口径应写为 [das] chore(notice): preserve third-party attribution materials
- [das] chore(license): record Hygon DCU attribution 是这一类的典型合规示例
后续判断顺序可固定为：先判断是不是新功能，再判断是不是 bug 修复，再判断是不是性能优化；如果都不是，再看是不是文档、测试、构建、CI 或合规材料。只有在功能行为不变、主要是结构整理时，才使用 refactor。
