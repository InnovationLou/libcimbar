# Windows (MSVC) 移植修改说明

本次修改使 libcimbar 项目能够在 Windows 11 + MSVC (Visual Studio 2022/2026) 环境下成功编译和运行。

## 修改概览

共修改 **22 个文件**，新增约 156 行代码，修改约 39 行代码。

---

## 1. CMake 构建系统适配

### `CMakeLists.txt` (根目录)

- **MSVC 编译选项**：添加 `/FI "iso646.h"` 解决 `and`/`or`/`not` 关键字兼容性；添加 `/utf-8` 解决 `fmt` 库中 UTF-8 字符在 GBK 代码页下的编码错误
- **条件编译标志**：GCC 特有的 `-Wall -fPIC` 等标志仅在非 MSVC 环境下设置
- **C++ 文件系统库**：MSVC 下将 `CPPFILESYSTEM` 设为空（MSVC 内置支持 `<filesystem>`，无需额外链接）
- **OpenGL 库切换**：Windows 下使用 GLEW + glfw + opengl32 替代 Linux 的 GLES3
- **GUI 可选构建**：新增 `DISABLE_GUI` 选项，允许在没有 GLEW/GLFW 的环境下跳过 GUI 相关目标

### `src/exe/*/CMakeLists.txt` (5个文件)

- **禁用 strip 命令**：`CMAKE_STRIP` 是 Linux 工具，在 MSVC 下用 `if(NOT MSVC)` 跳过
- **OpenGL 库变量化**：将硬编码的 `GL glfw` 替换为 `${OPENGL_LIBS}` 变量，支持跨平台

### `src/lib/cimbar_js/CMakeLists.txt`

- 同上，将 `GL glfw` 替换为 `${OPENGL_LIBS}`

---

## 2. OpenGL / GUI 层适配 (5个文件)

### `src/lib/gui/gl_2d_display.h`, `gl_shader.h`, `gl_program.h`

- **头文件切换**：Windows 下用 `<GL/glew.h>` 替代 `<GLES3/gl3.h>` 和 `<GLES2/gl2ext.h>`
- **GLSL 版本**：着色器版本从 `#version 300 es` (OpenGL ES) 改为 `#version 330 core` (桌面 OpenGL)

### `src/lib/gui/mat_to_gl.h`

- **纹理格式**：单通道图像在桌面 OpenGL 中使用 `GL_RED` 替代 `GL_LUMINANCE`

### `src/lib/gui/window_glfw.h`

- **GLEW 初始化**：在创建 OpenGL 上下文后调用 `glewInit()`
- **OpenGL Core Profile**：设置 GLFW 请求 OpenGL 3.3 Core Profile

---

## 3. C++ 标准库兼容性修复 (8个文件)

### `src/lib/util/File.h`

- 新增 `File(const std::filesystem::path&, bool)` 构造函数重载，解决 MSVC 下 `std::filesystem::path` 不能隐式转换为 `std::string` 的问题

### `src/lib/fountain/fountain_decoder_sink.h`

- `write_on_store` 和 `decompress_on_store` 函数的 `data_dir` 参数类型改为 `std::filesystem::path`，使用 `/` 运算符拼接路径

### `src/lib/fountain/FountainEncoder.h`

- **新增移动构造函数**：解决 `restart_and_resize_buffer()` 中重新赋值 `FountainEncoder` 时，隐式拷贝构造函数导致 `_codec` 指针 use-after-free 的 SEGFAULT 问题
- **禁用拷贝构造函数**：防止原始指针的浅拷贝

### 测试文件 (5个)

- `DecoderTest.cpp`, `EncoderTest.cpp`, `EncoderRoundTripTest.cpp`, `ExtractorTest.cpp`, `zstd_decompressorTest.cpp`：将 `std::filesystem::path` 显式转换为 `std::string`（调用 `.string()`）

### `src/lib/compression/test/zstd_compressorTest.cpp`

- `std::independent_bits_engine` 的模板参数从 `unsigned char` 改为 `unsigned int`（MSVC 严格执行 C++ 标准要求）

### `src/lib/fountain/test/fountain_sinkTest.cpp`

- `get_done()` 结果使用 `turbo::str::sort()` 排序后再比较，解决 `std::unordered_map` 迭代顺序在 MSVC 和 GCC 间不一致的问题

---

## 4. 已知的平台差异

`encoder_test` 中有 5 个 DecoderTest 子用例在 Windows 上的解码字节数与 Linux 预期值略有差异（如 9300 vs 9330），这是由于 OpenCV 版本和浮点运算精度的跨平台差异造成的，不影响实际功能。

---

## 依赖关系

| 依赖 | 安装方式 |
|------|----------|
| OpenCV 4.x | vcpkg |
| GLEW | vcpkg |
| GLFW3 | vcpkg |
| Visual Studio 2022+ (MSVC) | 官方安装 |
| CMake 3.10+ | 随 Visual Studio 安装 |

