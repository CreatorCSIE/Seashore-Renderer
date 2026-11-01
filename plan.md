# Seashore-Renderer 开发计划（渲染器 Fork 线）

> 2026-11-01 自立工作位。本仓 = fork **Krypton Wrapper（NG-GL4ES 0.4.5，MIT）**，产出与 Krypton 并列的"Seashore 渲染器"条目；修复从 Krypton 代码出发，不追自研。
> 上游 = [BZLZHH/NG-GL4ES](https://github.com/BZLZHH/NG-GL4ES)（gl4es + Pojav gl4es-114-extra 双 fork）；origin = [CreatorCSIE/Seashore-Renderer](https://github.com/CreatorCSIE/Seashore-Renderer)，HEAD `6e153a1`（0.4.5），submodule 齐（glslang 11.1.0 / SPIRV-Cross vulkan-sdk-1.4.328.0）。
> 主仓 = `Seashore-Launcher`，取证卷宗 = 其 `research/qcl-android-shell.md` §31-§35（本文根因结论全部引自该处，不重复论证）。

## 三面账（根因已定罪，本仓职责边界清晰）

1. **天空/classic 色块面 = 本仓唯一必修面**。根因 = 上游 gl4es `glInterleavedArrays` 的 `GL_C3F_V3F` 步长 typo：`vert=4` 应为 3（规范 C3F+V3F=6 float=24B，vert=4 使 stride 误算 28、vertex size=4、偏移 +12 → 逐顶点错位 4 字节 → 垃圾颜色 + 巨型三角）。本仓位置 = `src/gl/gl4es.c` L220-222（与 1.1.4 逐字一致，已复核）。修复 = 一行 `vert=4→3`，主仓已在 1.1.4 基座 + AVD + QCL/Holy-GL4ES 终验通过（坏窗口 0.0.14a_08 天空与生成进度色污染全消）。
2. **阴影三角面 = 本仓无责，明确不修**。真凶 = 游戏（0611~b1.7.3）预三角化复制循环不搬法线槽 → 零法线重复顶点；本仓 FPE 的 `normalize(gl_NormalMatrix * gl_Normal)` 只是把桌面驱动容错掉的未定义行为显性化。出货归主仓/启动器侧映射表（`-Dlf.gl4esquad` 语义），与渲染器无关（卷宗 §35）。可选增强见工单④。
3. **崩溃面 = 白捡**。上游 listdraw.c EBO `sizeof(GL_UNSIGNED_SHORT)`→`sizeof(GLushort)` 的越界 bug，Krypton 已随上游修好，本仓无需动作（卷宗 §31）。

## 工单序

- [x] **① 基座构建跑通**：NDK r30 + CMake 交叉编 **arm64-v8a + x86_64**（AVD 验证优先）。构建面事实：`CMakeLists.txt` 自带 `if(ANDROID)` 一等分支（`-DNOX11`、排除 `src/glx/glx.c`、链 `android`/`log`，EGL 运行时 dlopen，与 Pojav gl4es-114-extra 同形态）；硬约束 `CMAKE_CXX_STANDARD 23`（NDK ≥r26，r30 满足）；`DEFAULT_ES=3`，远古 MC 运行时 `LIBGL_ES=1/2` 回落。注意：COMPILE.md 的 Android 节是上游陈旧文档（提 Android.mk，实际无此文件），以 CMake 工具链文件为准。
- [x] **② 直移 `vert=4→3`**（`src/gl/gl4es.c` L222）：AVD 坏窗口回归 = 0.0.14a_08 / 0.0.23a_01 天空色块 + classic 色块全消，正常场景零回归。
- [ ] **③ 产物入库主仓**：`libng_gl4es.so` 改名（建议 `libss_gl4es.so`，随品牌定名）进 `Seashore-Launcher/app/src/main/jniLibs/<abi>/`；主仓壳层注册渲染器条目并设默认（直接规避 QCL"切版本回落默认渲染器"的验证坑）。主仓侧维持 fork 期零 NDK。
- [ ] **④（可选）C 侧零法线防御**：FPE `normalize` 前加零向量保护——可兜住一切发零法线的应用（不止 MC），但属对 GL spec 未定义区做行为固化，进 PR 前单独评估（卷宗 §35 选项 B）。
- [ ] **⑤ 上游反哺**：C3F 一行修复提 BZLZHH/NG-GL4ES 与 ptitSeb/gl4es；ptitSeb/gl4es **#429**（b1.7.3+1.12.2 阴影三角，open 零定位）可按卷宗 §35 证据链回帖定位。
- [ ] **⑥ 扩 ABI**：armeabi-v7a 跟进；x86(32) 按主仓 §9 策略 best-effort 或不 ship。

**验收基准**：坏窗口（0.0.14a_08/0.0.23a_01）天空与 GUI 色块全消；b1.7.3 正常场景（地形/云/持物）零回归；arm64 真机 sanity 通过（发版前）。

## 纪律

- 本仓改动**不擅自 commit/push**（与主仓 gl4es 工作区同规，逐批由用户拍板）。
- 上游同步 = **cherry-pick 非 rebase**：origin 保持 fork，另加 `upstream` remote（BZLZHH/NG-GL4ES）pull-only 摘取。
- 许可：MIT 随带——LICENSE 保留上游三行版权（gl4es 两作者 + BZLZHH），追加本 fork 修改行（`Copyright (c) 2026 CreatorCSIE … modifications only`）；glslang/SPIRV-Cross/cJSON 等第三方清单进 THIRD_PARTY_NOTICES（随主仓合规面）。
- 取证复用：主仓 `Seashore-Launcher/reference/gl4es-forensics/`（wedge-series.patch 探针套件 + gltrace 脚手架）打在 1.1.4 上；本仓 drawing.c 已重构（scratch 统一索引展开），移植探针需重对锚点。

## 与主仓的接口账

| 事项 | 归属 |
|---|---|
| 天空/classic 色块修复 | 本仓（工单②） |
| 阴影三角（0611~b1.7.3） | 主仓映射表 / `-Dlf.gl4esquad`（本仓不修） |
| 渲染器条目注册、默认值、jniLibs 内置 | 主仓 Phase 1 |
| 上游 PR（C3F / #429 回帖） | 本仓出素材，用户拍板提交 |
