# 2026-11-10 windows 工具链 libzmq(CMake4) 配置失败修复

## 现象

`cmake_multi_platform` workflow 的 `windows-latest` 工具链（cl）在 Configure 阶段失败，其余 ubuntu 工具链全部成功：

```
CMake Error at build/_deps/libzmq-src/CMakeLists.txt:7 (cmake_minimum_required):
  Compatibility with CMake < 3.5 has been removed from CMake.
```

## 根因

CPM 拉取的 `libzmq 4.3.5` 声明了 `cmake_minimum_required(3.0)`，CMake 4.x 移除了 `<3.5` 的兼容策略，导致配置被硬性拒绝（ubuntu 可通过环境规避，Windows 无法）。

## 修复

在 `.github/workflows/cmake_multi_platform.yml` 的 Configure 步骤追加官方提供的过渡开关：

- `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`
- 同时追加 `-DCPM_SOURCE_CACHE=${{ github.workspace }}/.cpm-cache`，避免每次全量拉取依赖。

## 参考

- https://cmake.org/c/help/latest/variable/CMAKE_POLICY_VERSION_MINIMUM.html

## 二次修正

首轮加 `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` 后 Windows 工具链在 Configure 前即报
`Invalid CMAKE_POLICY_VERSION_MINIMUM value "3"`（runner 默认 PowerShell 把 `3.5`
拆成 `=3` 和 `.5` 两个参数，并产生 `Ignoring extra path ".5"` 告警）。给 Configure
步骤增加 `shell: bash` 并把该参数改为引号形式 `-DCMAKE_POLICY_VERSION_MINIMUM="3.5"` 后解决。


## 三次修正

上一轮把共享的 Configure 步骤整个加上了 `if: runner.os == 'Windows'`，导致 Linux 工具链没有执行
configure（`Error: .../build is not a directory`）；同时 `${{ github.workspace }}` 里的反斜杠
被 Windows 上的 MSYS 路径转换破坏成 `aex-actorex-actor`。

改为：
- Linux 工具链走原 Configure 命令（不带 policy 参数）；
- Windows 工具链单独一步 `shell: bash`，全部使用 `$GITHUB_WORKSPACE`（正斜杠形式）+ policy 参数。

## 验证记录
| Run | 结果 |
|---|---|
| 42833627961 (8471d1b) | failure — pwsh 参数拆分问题 |
| 42838763197 (1d3f1de) | failure — workspace 路径问题 |
| 42840023260 (d265309) | ✅ success |
