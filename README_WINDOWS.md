# libcimbar — Windows 编译与运行指南

本文档介绍如何在 **Windows 11** 上使用 **Visual Studio (MSVC)** 编译和运行 libcimbar 项目。

---

## 前置要求

| 软件 | 版本要求 | 说明 |
|------|----------|------|
| Visual Studio | 2022 或更高 | 需安装 "C++ 桌面开发" 工作负载 |
| vcpkg | 最新版 | Microsoft 的 C++ 包管理器 |
| Git | 任意版本 | 用于克隆代码和子模块 |
| CMake | 3.10+ | 随 Visual Studio 一起安装 |

---

## 一、安装依赖

### 1.1 安装 vcpkg（如果尚未安装）

```powershell
cd C:\
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
.\bootstrap-vcpkg.bat
.\vcpkg integrate install
```

### 1.2 通过 vcpkg 安装依赖库

```powershell
vcpkg install opencv4:x64-windows
vcpkg install glew:x64-windows
vcpkg install glfw3:x64-windows
```

> ⏱ OpenCV 编译时间较长（约 15-30 分钟），请耐心等待。

---

## 二、获取源代码

### 2.1 克隆仓库（含子模块）

```powershell
git clone --recurse-submodules https://github.com/sz3/libcimbar.git
cd libcimbar
```

### 2.2 如果已经克隆但缺少子模块

```powershell
cd libcimbar
git submodule update --init --recursive
```

> ⚠ `samples` 子模块包含测试用的样本图像，运行测试时需要。如果 GitHub 连接不畅，可使用镜像：
> ```powershell
> git config submodule.samples.url https://ghfast.top/https://github.com/sz3/cimbar-samples
> git submodule update --init samples
> ```

---

## 三、编译项目

### 3.1 创建构建目录并配置 CMake

打开 **Developer PowerShell for VS** 或普通 PowerShell（需确保 cmake 在 PATH 中）：

```powershell
cd libcimbar
mkdir build
cd build

cmake .. -G "Visual Studio 17 2022" -A x64 `
    -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
```

> **说明**：
> - 将 `C:/vcpkg` 替换为你的 vcpkg 实际安装路径
> - 如果使用 Visual Studio 2026，将生成器改为 `"Visual Studio 18 2026"`
> - 如果不需要 GUI 组件（cimbar_send/recv），可添加 `-DDISABLE_GUI=ON`

### 3.2 编译

```powershell
cmake --build . --config Release
```

编译成功后，可执行文件位于：

```
build\build\src\exe\cimbar\Release\cimbar.exe
build\build\src\exe\cimbar_extract\Release\cimbar_extract.exe
```

---

## 四、运行测试

### 4.1 运行全部测试

```powershell
cd build
ctest --output-on-failure -C Release
```

### 4.2 运行单个测试

```powershell
# 例如：只运行 fountain 测试
.\build\src\lib\fountain\test\Release\fountain_test.exe
```

### 4.3 预期测试结果

| 测试套件 | 预期状态 |
|---------|---------|
| bit_file_test | ✅ Pass |
| chromatic_adaptation_test | ✅ Pass |
| cimb_translator_test | ✅ Pass |
| compression_test | ✅ Pass |
| encoder_test | ⚠ 部分子用例因跨平台浮点差异失败（预期行为） |
| extractor_test | ✅ Pass |
| fountain_test | ✅ Pass |
| image_hash_test | ✅ Pass |

---

## 五、使用 cimbar 进行文件传输

### 5.1 编码：文件 → cimbar 图像

将任意文件编码为 cimbar 图像（默认使用 fountain 编码 + zstd 压缩）：

```powershell
cimbar.exe --encode -i myfile.txt -o output_prefix
```

输出：`output_prefix_0.png`, `output_prefix_1.png`, ... （取决于文件大小）

### 5.2 解码：cimbar 图像 → 文件

**从程序生成的图像解码**（无需透视矫正）：

```powershell
cimbar.exe -i output_prefix_0.png -o decoded_dir/ --no-deskew
```

**从相机/屏幕拍摄的图像解码**（自动透视矫正）：

```powershell
cimbar.exe -i photo_of_screen.jpg -o decoded_dir/
```

### 5.3 完整的文件传输测试示例

```powershell
# 1. 准备测试文件
echo "Hello, CIMBar!" > test.txt

# 2. 编码
cimbar.exe --encode -i test.txt -o encoded/cimbar

# 3. 解码
mkdir decoded
cimbar.exe -i encoded/cimbar_0.png -o decoded/ --no-deskew

# 4. 验证 (比较 SHA256 哈希)
Get-FileHash test.txt -Algorithm SHA256
Get-FileHash decoded\*.* -Algorithm SHA256
# 两个哈希值应完全一致
```

### 5.4 命令行参数说明

| 参数 | 说明 |
|------|------|
| `-n, --encode` | 执行编码模式 |
| `-i, --in` | 输入文件（编码时为源文件，解码时为 cimbar 图像） |
| `-o, --out` | 输出路径（编码时为文件前缀，解码时为输出目录） |
| `-m, --mode` | cimbar 模式：`B`（默认）, `4C`（旧版兼容） |
| `-z, --compression` | 压缩级别（0 = 不压缩） |
| `--no-deskew` | 跳过透视矫正（用于程序生成的图像） |
| `--no-fountain` | 禁用 fountain 编码（也会禁用压缩） |
| `--undistort` | 尝试镜头畸变矫正 |
| `--color-correct` | 色彩校正级别：2=完整, 1=简单, 0=关闭 |
| `-h, --help` | 显示帮助 |

---

## 六、从 IDE 中使用

### Visual Studio

1. 打开 Visual Studio → **File** → **Open** → **CMake...**
2. 选择 `libcimbar/CMakeLists.txt`
3. 在 CMake Settings 中设置 vcpkg toolchain 文件
4. 选择 **x64-Release** 配置
5. 编译并运行

### VS Code + CMake Tools

1. 安装 CMake Tools 扩展
2. 打开项目文件夹
3. 在 `settings.json` 中配置：
```json
{
    "cmake.configureArgs": [
        "-DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake"
    ]
}
```
4. 选择 MSVC Kit → Configure → Build

---

## 七、故障排除

### Q: 编译时报 "该文件包含不能在当前代码页(936)中表示的字符"

根目录 `CMakeLists.txt` 中已添加 `add_compile_options(/utf-8)`，如仍有问题请确认 CMake 配置已更新。

### Q: 找不到 OpenCV / GLEW / GLFW

确保 CMake 配置时指定了 vcpkg toolchain：
```
-DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
```

### Q: GUI 相关目标编译失败

如果不需要 GUI 功能，可禁用 GUI 目标：
```
cmake .. -DDISABLE_GUI=ON
```

### Q: `samples` 子模块无法下载

使用国内镜像加速：
```powershell
git config submodule.samples.url https://ghfast.top/https://github.com/sz3/cimbar-samples
git submodule update --init samples
```

### Q: encoder_test 中 DecoderTest 子用例失败

这是由于 MSVC 与 GCC 在浮点运算和 OpenCV 图像处理上的跨平台差异导致的，不影响实际编码/解码功能。

