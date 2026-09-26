# Birdtray-Build

这是一个用于在 GitHub Actions 上自动克隆并构建 upstream 项目 `gyunaev/birdtray` 的构建仓库（Birdtray-Build）。它只保留 CI/workflow 配置与可选补丁（overrides/），在运行时从 upstream 克隆源代码、配置并在 Windows runner 上构建、打包并（在发布标签时）生成安装器。

说明
- 上游项目（源码）： https://github.com/gyunaev/birdtray
- 本仓库职责：在 Actions runner 上拉取 upstream 源码，应用可选覆盖（overrides/），运行 CMake 构建、收集产物并可选创建 Windows 安装器。
- CI 触发：
  - 手动：workflow_dispatch（支持参数 upstream_ref、upstream_repo、debug）

主要特性
- 在运行时克隆 upstream 仓库并按指定 ref（branch/tag/commit）构建
- 可把本仓库的 overrides/ 目录内容应用到 upstream（用于补丁）
- 支持 Windows 构建（MSVC + Qt），并使用 windeployqt 打包 Qt 运行时
- 在 push 创建版本标签（vX.Y.Z）时可构建并缓存 Windows 安装器（NSIS）
- CI 中包含若干保护与诊断逻辑（回退到 upstream 默认分支、构建失败时收集日志）

快速开始（本仓库）
1. 将本仓库 checkout 到你的组织/个人空间（或直接在此仓库触发 workflow）。
2. 可通过 GitHub Actions 页面手动触发 workflow（workflow_dispatch）并传入：
   - upstream_ref：要构建的上游分支/tag/commit（默认：main，但脚本会在 remote 上找不到时回退到 upstream 的默认分支或 master）
   - upstream_repo：要构建的上游仓库 URL（默认：https://github.com/gyunaev/birdtray.git）
   - debug：true/false，开启后会打印 workspace 布局等额外调试信息

本地构建（参考）
- Windows（推荐使用 Visual Studio 2022（MSVC）和 Qt 6.x）：
  1. 安装 Qt（相匹配版本，如 6.8.x），并确保 windeployqt 在 PATH 或可在 RUNNER_TEMP 下找到。
  2. 通过 Visual Studio 提供的开发者命令行或 x64 Native Tools PowerShell 打开终端。
  3. 在本地复制 upstream 源码后运行：
     cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_WITH_TESTS=OFF -DDONT_EXECUTE_INSTALLER=ON
     cmake --build build --config Release --target birdtray

重要 CI/构建依赖（Windows）
- Visual Studio / MSVC (tested: VS 2022 / MSVC)
- Qt（由 `jurplel/install-qt-action` 在 runner 临时目录安装）
- OpenSSL（通过 Chocolatey 安装 openssl.light，工作流会向 PATH 添加 "C:\Program Files\OpenSSL"）
- NSIS（仅在发布/部署构建用于生成安装器时需要，通过 choco 安装 makensis）

overrides/（可选）
- 如果你需要对 upstream 源进行临时补丁或替换某些文件，可以在本仓库根目录创建 `overrides/` 目录并把要覆盖的文件放入（目录结构应与 upstream 对应）。
- CI 会在克隆 upstream 后把 overrides/ 的内容拷贝到 upstream 工作树，覆盖对应文件。

产物与发布
- CI 会把最终 `dist/` 目录（包含可执行文件与 translations 等）作为 artifact 上传。
- 当触发符合 `refs/tags/vX.Y.Z` 的 push（并且满足 runner/os 条件）时，CI 会尝试构建安装器并把它缓存为 artifact，后续 release 流程会把安装器添加到草稿 Release。

调试/故障排查（常见问题）
- Remote branch not found
  - 问题：当指定 upstream_ref（默认 main）并且 upstream 仓库没有该分支时，clone 会失败。
  - 解决：使用 workflow 的 upstream_ref 参数指定正确分支，或允许 workflow 自动回退到 upstream 的默认分支（工作流已实现自动回退逻辑）。
- googletest / CMake 子进程兼容性错误
  - 如果构建 tests 时，子项目使用了不兼容的 cmake_minimum_required，可能导致配置失败。
  - 解决：CI 默认将 `-DBUILD_WITH_TESTS=OFF` 以避免此类问题。若需运行 tests，请在本地/自托管 runner 上修复子项目 CMake 或提供兼容的 CMake 环境。
- windeployqt 未找到
  - 确认 Qt 已安装并 windeployqt.exe 在 PATH，或工作流的 install-qt-action dir 设置正确（工作流尝试在 RUNNER_TEMP/Qt 下查找）。
- makensis / NSIS 未找到（生成安装器时）
  - 在部署/发布构建中，生成 installer 需要 NSIS（makensis）。工作流只在部署构建时安装/使用 NSIS；若你在测试非发布构建时遇到安装器相关失败，可跳过部署触发。
- 删除 build 目录失败（Windows）
  - 原因可能为文件被进程占用或权限问题。工作流包含了 Windows 下更鲁棒的删除逻辑（使用 PowerShell Remove-Item -Recurse -Force），并在失败时列出残留文件以便诊断。

CI 运行与日志
- 查看最近的 workflow 运行（示例）：
  https://github.com/db-one/Birdtray-Build/actions/runs/30795000018
- 如果 CI 失败，请先查看对应 job 的日志，搜索关键字（cmake、msbuild、windeployqt、ERROR），并启用 workflow_dispatch 的 `debug=true` 来获得更多调试输出。

贡献
- 若需要调整 workflow（例如更改 Qt 版本、添加依赖或修改替换逻辑），请发起 Pull Request。
- 若你需要在 CI 中测试对 upstream 的补丁，请把那些补丁放到 `overrides/` 目录并在 PR 中说明用途。

许可与上游
- 本仓库仅包含构建脚本/CI 配置；上游项目 Birdtray 的源代码与许可归 upstream 所有（参见 https://github.com/gyunaev/birdtray 的 LICENSE）。
- 请在分发 upstream 二进制/安装器时遵循 upstream 的许可（GPLv3）。

联系方式
- 仓库：https://github.com/db-one/Birdtray-Build
- 上游：https://github.com/gyunaev/birdtray
- 若需要我帮你把 README 推送到仓库或创建 PR，请告诉我，我可以生成补丁或分支提交。
