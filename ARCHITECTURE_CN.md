# pdfrx 架构概览

pdfrx 是一个包含五个包的 monorepo，其依赖层次结构如下：

```
pdfium_dart (FFI 绑定)
    ├──→ pdfium_flutter (捆绑 PDFium 二进制文件)
    │           ↓
    └──→ pdfrx_engine (PDF API, 纯 Dart)
                ├──→ pdfrx (Flutter 组件) ←── pdfium_flutter
                └──→ pdfrx_coregraphics (Apple 平台的替代后端)
```

## 包说明

### 1. pdfium_dart (`packages/pdfium_dart/`)
PDFium 的底层 Dart FFI 绑定。
- **类型**: 纯 Dart 包。
- **功能**: 使用 `ffigen` 自动生成的 FFI 绑定，提供对 PDFium C API 的直接访问。
- **特性**: 包含 `getPdfium()` 函数，可按需下载 PDFium 二进制文件。
- **用途**: 作为更高级别包的基础。

### 2. pdfium_flutter (`packages/pdfium_flutter/`)
用于加载 PDFium 原生库的 Flutter FFI 插件。
- **类型**: Flutter 插件。
- **功能**: 捆绑了适用于所有 Flutter 平台（Android, iOS, Windows, macOS, Linux）的预构建 PDFium 二进制文件。
- **特性**: 提供运行时加载 PDFium 的实用工具，并重新导出 `pdfium_dart` FFI 绑定。

### 3. pdfrx_engine (`packages/pdfrx_engine/`)
构建在 PDFium 之上的与平台无关的 PDF 渲染 API。
- **类型**: 纯 Dart 包（无 Flutter 依赖）。
- **功能**: 提供核心 PDF 文档 API。
- **依赖**: 依赖 `pdfium_dart` 获取 PDFium 绑定。
- **用途**: 可独立用于非 Flutter 的 Dart 应用程序（如 CLI 工具、服务端处理）。

### 4. pdfrx (`packages/pdfrx/`)
跨平台的 Flutter PDF 查看器插件。
- **类型**: Flutter 插件。
- **功能**: 提供 Flutter 组件（Widgets）和 UI 组件，支持缩放、文本选择等。
- **依赖**: 依赖 `pdfrx_engine` 进行 PDF 渲染，依赖 `pdfium_flutter` 获取捆绑的二进制文件。
- **平台支持**: iOS, Android, Windows, macOS, Linux 和 Web。

### 5. pdfrx_coregraphics (`packages/pdfrx_coregraphics/`)
iOS/macOS 的 CoreGraphics 后端渲染器。
- **类型**: 实验性包。
- **功能**: 在 Apple 平台上使用 PDFKit/CoreGraphics 替代 PDFium。
- **用途**: 作为 Apple 平台的直接替换方案。

## 平台特定说明

- **iOS/macOS**: 使用预构建的 PDFium 二进制文件或 CoreGraphics。
- **Android**: 使用 CMake 进行原生构建，构建期间下载 PDFium 二进制文件。
- **Web**: 使用 WASM 版 PDFium (`pdfium.wasm`)。
- **Windows/Linux**: 基于 CMake 构建，构建期间下载二进制文件。
