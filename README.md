# Seashore Renderer - 远古 Minecraft 的 GL→GLES 转译层（渲染器 Fork 线）

`Seashore Renderer` 是 [Seashore-Launcher](https://github.com/CreatorCSIE/Seashore-Launcher)（远古 Minecraft Android 启动器）的**渲染器特化 fork**：以 [Krypton Wrapper / NG-GL4ES 0.4.5](https://github.com/BZLZHH/NG-GL4ES)（本身为 [gl4es](https://github.com/ptitSeb/gl4es) + [gl4es-114-extra](https://github.com/PojavLauncherTeam/gl4es-114-extra) 双 fork）为基座，面向远古版本线（Pre-Classic → b1.8 / 1.5.2）做修复与裁剪，作为启动器内与 Krypton 并列的"Seashore 渲染器"条目出货。

**当前处于工单期**：fork 基点 `6e153a1`（0.4.5，submodule 齐），基座构建（工单①）进行中，暂无发布构件。施工与验收以 [plan.md](./plan.md) 为准。

## 为什么存在这个 fork

启动器调研期对"远古版天空/场景被大块红紫青色块覆盖，所有第三方转译路线的天花板"完成了源码级取证：根因是 **gl4es 上游 `glInterleavedArrays` 对 `GL_C3F_V3F` 的步长 typo**——Krypton 与 gl4es 1.1.4 逐字同带，故"换渲染器同结果"。这不是天花板，是一行代码：

```c
case GL_C3F_V3F:
    color = 3;
    vert = 4;   // ← 应为 3。规范 C3F+V3F = 6 float = 24B stride
```

`vert=4` 使 stride 误算为 28、vertex size=4、偏移 +12 → 逐顶点错位 4 字节 → 垃圾颜色（品红/黄/青）+ 垃圾坐标（天空巨型三角）。修复 `vert=4→3` 已在 1.1.4 基座 + Android AVD 终验通过（坏窗口 c0.0.14a_08 天空与生成进度色污染全消），本仓直移至 Krypton 基座（`src/gl/gl4es.c`，工单②）。

## 三面账（职责边界）

| 面 | 状态 | 归属 |
|---|---|---|
| 天空 / Classic 色块（c0.0.13a_03~in-20100124-2） | **本仓唯一必修面**：`vert=4→3` 直移 | 本仓工单② |
| 阴影三角（inf_20100611~b1.7.3） | **无责不修**：真凶 = 游戏预三角化产生零法线重复顶点，转译层 `normalize(0)` 只是显性化；修复在启动器侧翻字段（`-Dlf.gl4esquad`） | 主仓 / legacyfix fork |
| 间歇崩溃（listdraw EBO sizeof） | **白捡**：Krypton 已随上游修好 | 无需动作 |

取证全链（探针阶梯、四角表、版本窗口复核）见主仓卷宗 `Seashore-Launcher/research/qcl-android-shell.md` §31–§35。

## 构建

- **工具链**：NDK **r30**（硬约束 `CMAKE_CXX_STANDARD 23` → 下限 r26/clang 17+）+ CMake 交叉编译；产物 `libng_gl4es.so`（SHARED，LTO + `--gc-sections` + strip）。
- **Android 是一等目标**：`CMakeLists.txt` 自带 `if(ANDROID)` 分支（`-DNOX11`、排除 `src/glx/glx.c`、链 `android`/`log`）；EGL 运行时 dlopen，context 由启动器 `jre_launcher` 创建——与 Pojav gl4es-114-extra 同形态。
- **ABI**：首发 arm64-v8a + x86_64（AVD 验证优先）；armeabi-v7a 跟进；x86(32) best-effort 或不 ship。
- **注意**：`COMPILE.md` 的 Android 节是上游陈旧文档（提 Android.mk，本仓无此文件），以 CMake 工具链文件为准。
- **运行时**：`DEFAULT_ES=3`（GLES3 后端）；远古 MC 可 `LIBGL_ES=1/2` 回落，其余 `LIBGL_*` 环境变量接口沿 gl4es 传统（见 [USAGE.md](./USAGE.md)）。

## 使用

本仓不独立面向终端用户：产物预编译进 `Seashore-Launcher/app/src/main/jniLibs/<abi>/`，由壳层注册渲染器条目并设默认。集成接口账见 [plan.md](./plan.md) 末节。

## 许可与上游致谢

MIT License（随带 [LICENSE](./LICENSE)，本仓修改代码同为 MIT）。血脉与第三方组件：

- [gl4es](https://github.com/ptitSeb/gl4es)（MIT）— 基础转译层，作者 [ptitSeb](https://github.com/ptitSeb)（[赞助](https://paypal.me/0ptitSeb)）
- [gl4es-114-extra](https://github.com/PojavLauncherTeam/gl4es-114-extra)（MIT）— Pojav 定制分支
- [Krypton Wrapper / NG-GL4ES](https://github.com/BZLZHH/NG-GL4ES)（MIT）— 本仓直接上游；部分代码来自 MobileGlues
- [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross)（Apache-2.0）、[glslang](https://github.com/KhronosGroup/glslang)（多许可）、[cJSON](https://github.com/DaveGamble/cJSON)（MIT）

上游同步纪律 = **cherry-pick 非 rebase**；`upstream` remote（BZLZHH/NG-GL4ES）保持 pull-only。
