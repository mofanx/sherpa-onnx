# sherpa-onnx 在 Android Termux 下构建 wheel 并 pip 安装（sherpa-onnx 1.12.20）

本文档面向 **Termux (Android/bionic)**，目标是：

- 在 Termux 中本地编译 `sherpa_onnx` Python 扩展（pybind11 + CMake）
- 生成可安装的 wheel，并使用 `pip install` 安装
- 解决 Termux 下常见问题：
  - wheel 打包阶段拷贝不存在的 CLI 二进制导致失败
  - 安装后 `ImportError: dlopen failed: cannot locate symbol ...`（静态库符号未被拉入）

> 注意：Termux 通常 `platform.system()` 显示为 `Linux`，但运行时是 Android bionic，不能直接复用 Linux 预编译依赖。

---

## 0. 已包含的项目改动（本仓库已处理）

项目根目录已做如下兼容修改（无需你额外操作）：

- `cmake/cmake_extension.py`
  - 增加 Termux 检测 `is_termux()`
  - Termux 默认关闭（可通过 `SHERPA_ONNX_CMAKE_ARGS` 覆盖）：
    - `SHERPA_ONNX_ENABLE_PORTAUDIO=OFF`
    - `SHERPA_ONNX_ENABLE_WEBSOCKET=OFF`
    - `SHERPA_ONNX_ENABLE_BINARY=OFF`
- `setup.py`
  - Termux 下不再打包 CLI 二进制（避免 wheel 构建报错）
  - `data_files` 仅在目标文件存在时才打包
  - `packages` 包含 `sherpa_onnx.lib`（配合 `sherpa_onnx/lib/__init__.py`）
- `sherpa-onnx/python/sherpa_onnx/lib/__init__.py`
  - 新增子包初始化文件，使 `from sherpa_onnx.lib._sherpa_onnx import ...` 可用
- `sherpa-onnx/python/csrc/CMakeLists.txt`
  - 修正 `_sherpa_onnx` 的 RPATH 为 `$ORIGIN`
  - Termux 下用 `--whole-archive` 强制把 `sherpa-onnx-core` 静态库对象拉入 `_sherpa_onnx`，避免运行时缺符号
  - Termux 下链接 `log`（用于 `__android_log_print`）
- `sherpa-onnx/csrc/CMakeLists.txt`
  - Termux 下为 `sherpa-onnx-core` 增加编译宏 `SHERPA_ONNX_NO_ANDROID_ASSET=1`
- `sherpa-onnx/csrc/file-utils.h`
  - Termux 下禁用 Android AssetManager 相关头文件依赖（通过 `SHERPA_ONNX_NO_ANDROID_ASSET`）
  - 保留 `AAssetManager` 前向声明与 `ReadFile(AAssetManager*, ...)` 声明，保证相关代码可编译
- `sherpa-onnx/csrc/file-utils.cc`
  - Termux 下为 `ReadFile(AAssetManager*, ...)` 提供 stub 实现（退化为 `ReadFile(filename)`），避免运行时引用 `AAssetManager_open`

---

## 1. Termux 环境依赖安装

```bash
pkg update
pkg install -y python clang cmake ninja make pkg-config git patchelf
pip install -U pip setuptools wheel
```

---

## 2. 准备 onnxruntime（必须）

`sherpa-onnx` CMake 会优先尝试使用“已安装的 onnxruntime”。在 Termux 下**强烈建议显式指定**：

- `SHERPA_ONNXRUNTIME_INCLUDE_DIR`：包含 `onnxruntime_cxx_api.h` 的目录
- `SHERPA_ONNXRUNTIME_LIB_DIR`：包含 `libonnxruntime.so` 的目录

例如你把 onnxruntime 放在：

- `$PREFIX/opt/onnxruntime/include`
- `$PREFIX/opt/onnxruntime/lib`

则：

```bash
export SHERPA_ONNXRUNTIME_INCLUDE_DIR=$PREFIX/opt/onnxruntime/include
export SHERPA_ONNXRUNTIME_LIB_DIR=$PREFIX/opt/onnxruntime/lib
ls -l $SHERPA_ONNXRUNTIME_LIB_DIR/libonnxruntime.so
```

> onnxruntime 可以来自 Android 预编译包（常见目录结构包含 `headers/` 与 `jni/arm64-v8a/`）。

---

## 3. 构建 wheel

在项目根目录执行：

```bash
# 推荐使用 Ninja，更快
export SHERPA_ONNX_CMAKE_ARGS="-G Ninja -DCMAKE_BUILD_TYPE=Release"
export SHERPA_ONNX_MAKE_ARGS="-j$(nproc)"

python setup.py bdist_wheel
```

### 产物位置

- 期望生成：`dist/*.whl`
- 如果没有 `dist/`，先检查命令是否真正跑到了 `bdist_wheel`（而不是 `build_ext` 或 `install`）。

---

## 4. 安装 wheel

```bash
pip install --force-reinstall dist/*.whl
```

---

## 5. 验证

```bash
python -c "import sherpa_onnx; print('ok', sherpa_onnx.version)"
python -c "from sherpa_onnx.lib import _sherpa_onnx; print('ext ok', _sherpa_onnx.version())"
```

---

## 6. 常见问题排查

### 6.1 wheel 构建时报错：can't copy 'build/sherpa_onnx/bin/sherpa-onnx'

原因：wheel 打包阶段尝试打包 CLI 二进制，但 Termux 默认已关闭 `SHERPA_ONNX_ENABLE_BINARY`，因此 `bin/` 为空。

处理：本仓库的 `setup.py` 已在 Termux 下禁用该打包逻辑，并且仅在文件存在时才加入 `data_files`。

### 6.2 安装后 ImportError：dlopen failed: cannot locate symbol ...

典型表现：

- `_sherpa_onnx.cpython-XXX.so` 在加载时提示找不到 C++ 符号

原因（常见）：

- `sherpa-onnx-core` 在 CMake 中是 **STATIC** 库。在 Android/bionic 下，Python 扩展模块作为 `MODULE`/`SHARED` 链接时，链接器可能不会把静态库中的对象完整拉入，导致运行时缺符号。

处理：本仓库已在 Termux 下对 `_sherpa_onnx` 链接增加 `-Wl,--whole-archive ... -Wl,--no-whole-archive`，强制把 `sherpa-onnx-core` 全量链接进扩展。

### 6.3 安装后 ImportError：dlopen failed: cannot locate symbol "AAssetManager_open"

原因：

- `sherpa-onnx-core` 的 Android 代码路径里包含读取 App assets 的实现（`AAssetManager_open` 属于 Android NDK）。
- 在 Termux 中运行时通常不需要（也很难正确链接/加载）该能力。

处理（推荐）：

- 在 Termux 构建时禁用 Android AssetManager 相关实现，使扩展模块不再引用 `AAssetManager_open`。
- 本仓库已在 Termux 构建时为 `sherpa-onnx-core` 增加编译宏 `SHERPA_ONNX_NO_ANDROID_ASSET=1`，并对相关 include/函数实现做了条件编译。
- 修改后请清理缓存重新打 wheel 并重新安装。

验证/排错：

```bash
# 查看扩展模块依赖的 NEEDED 列表
readelf -d $(python -c "import sherpa_onnx,glob,os; import sherpa_onnx.lib as L; import pathlib; p=pathlib.Path(L.__file__).parent; print(list(p.glob('_sherpa_onnx*.so'))[0])") | grep NEEDED || true
```

重新构建（重要）：

```bash
rm -rf build dist
python setup.py bdist_wheel
pip install --force-reinstall dist/*.whl

readelf -d /data/data/com.termux/files/usr/lib/python3.12/site-packages/sherpa_onnx/lib/_sherpa_onnx*.so | grep NEEDED
```

### 6.4 找不到 libonnxruntime.so

确保：

- 构建时 `SHERPA_ONNXRUNTIME_LIB_DIR` 指向包含 `libonnxruntime.so` 的目录
- 安装后 `site-packages/sherpa_onnx/lib/` 中确实有 `libonnxruntime.so`

---

## 7. 推荐的重新构建流程（避免旧缓存）

当你修改了 CMake 或链接选项后，建议清理旧构建缓存再打包：

```bash
rm -rf build dist
python setup.py bdist_wheel
```

---

## 8. 反馈信息（用于进一步诊断）

如果仍失败，请提供以下输出：

```bash
python -V
uname -a
ls -la dist || true
ls -la build/lib.*/*/sherpa_onnx/lib || true
python -c "import sys; import sherpa_onnx; print(sherpa_onnx.__file__)" || true
```
