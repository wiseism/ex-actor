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
