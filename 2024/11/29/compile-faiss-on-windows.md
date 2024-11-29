# 在 Windows 上编译 faiss

前置要求：

- `Windows Terminal`+`nushell`：可选。
- `vs 2022`：编译器、`cmake`。
- `Intel oneMKL`：`faiss` 依赖。
- `vcpkg`：用来安装 `gflags`。

本文命令行均运行在 `nushell`，如需运行在 `PowerShell` 中，请自行微调。

打开 `Windows Terminal` 的 `nushell for vs`（参见附录）或 `Developer PowerShell for VS 2022`。

因为 `Developer PowerShell` 会修改 `VCPKG_ROOT` 为 vs 2022 的，而这个版本似乎用不了，报错缺少 `vcpkg.json`，因此这里要重置 `vcpkg` 根目录为你安装的 `vcpkg` 目录（如果你用的 `nushell for vs` 则不需要这个步骤，文末的附录里有恢复 `vcpkg` 路径的代码）：

```nushell
# 记得修改这里的 vcpkg 路径。
$env.VCPKG_ROOT = 'C:\sdk\vcpkg'
# 或使用 PowerShell：
# $env:VCPKG_ROOT = 'C:\sdk\vcpkg'
```

安装 `gflags`：

```nushell
vcpkg install gflags
```

构建：

```nushell
# 将 vs2022 的 cmake 添加到 Path 路径，如果是自己装的，编译的时候会报错 Could NOT find OpenMP_CUDA。
path add 'C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin'
# 或使用 PowerShell：
# $env:Path = "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;$env:Path"

cmake -B build -DFAISS_ENABLE_C_API=ON -DBLA_VENDOR=Intel10_64_dyn '-DMKL_LIBRARIES=C:\Program Files (x86)\Intel\oneAPI\mkl\2025.0\lib' -DFAISS_ENABLE_PYTHON=OFF -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF .
cmake --build build --config Release 
```

安装到一个目录，如 `C:\sdk\faiss`：

```nushell
cmake --install build --prefix 'C:\sdk\faiss'
```

## 附录：`Windows Terminal` 的 `nushell for vs` 配置

打开：`%USERPROFILE%\AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json`，在 `profiles` → `list` 中添加：

```json
{
  "commandline": "powershell.exe -Command \"&{ $vcpkg = $env:VCPKG_ROOT; Import-Module \"\"\"C:\\Program Files\\Microsoft Visual Studio\\2022\\Enterprise\\Common7\\Tools\\Microsoft.VisualStudio.DevShell.dll\"\"\"; Enter-VsDevShell -VsInstallPath \"\"\"C:\\Program Files\\Microsoft Visual Studio\\2022\\Enterprise\"\"\" -SkipAutomaticLocation -DevCmdArguments \"\"\"-arch=x64 -host_arch=x64\"\"\"; if ($vcpkg) { $env:VCPKG_ROOT = $vcpkg; } nu }\"",
  "font": {
    "size": 13.0
  },
  "guid": "{2f76c985-1644-43e8-b016-239e7f8dd7f1}",
  "hidden": false,
  "name": "nushell for vs"
}
```
