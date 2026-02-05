# pdfrx 分层架构详解

pdfrx 采用清晰的分层架构，从底层的原生二进制文件到顶层的 Flutter 组件，各层职责分明。以下是从下往上的详细架构描述：

## 第 1 层：原生层 (Native Layer)
**核心组件**: `libpdfium.so` (Linux/Android), `pdfium.dll` (Windows), `pdfium.dylib` (macOS), `PDFium.framework` (iOS), `pdfium.wasm` (Web)

- **职责**: 实际执行 PDF 解析、渲染和操作的 C++ 库。
- **来源**: Google 的 PDFium 开源项目。
- **交互**: 暴露 C 语言 API（如 `FPDF_LoadDocument`, `FPDF_RenderPageBitmap`）。

## 第 2 层：FFI 绑定层 (FFI Binding Layer)
**所属包**: `package:pdfium_dart`

- **职责**: 将原生 C API 映射为 Dart 代码。
- **关键组件**:
  - **`pdfium_bindings.dart`**: 由 `ffigen` 工具自动生成的代码。包含 C 结构体（如 `FPDF_DOCUMENT`）的 Dart `ffi.Struct` 映射，以及 C 函数的 Dart 方法映射。
  - **`pdfium.dart`**: 负责加载动态链接库 (`DynamicLibrary.open`) 并初始化绑定。

## 第 3 层：引擎层 (Engine Layer)
**所属包**: `package:pdfrx_engine`

这一层是核心逻辑层，负责管理原生资源、线程调度和对象抽象。

- **职责**:
  1. **隔离 (Isolation)**: 使用 Dart `Isolate` 将耗时的 PDF 操作（解析、渲染）与主线程（UI 线程）分离，避免阻塞 UI。
  2. **抽象 (Abstraction)**: 提供面向对象的 Dart API (`PdfDocument`, `PdfPage`)，屏蔽底层 C 指针和内存管理细节。
  3. **资源管理**: 自动处理内存释放（`dispose`），防止内存泄漏。

- **关键组件**:
  - **`BackgroundWorker` (`worker.dart`)**: 核心调度器。创建一个后台 Isolate，通过消息传递机制 (`SendPort`/`ReceivePort`) 接收主线程的请求，在后台执行 FFI 调用，并将结果返回。
  - **`_PdfDocumentPdfium` (`pdfrx_pdfium.dart`)**: `PdfDocument` 的具体实现。它不直接调用 FFI，而是将请求（如“渲染第 1 页”）封装成消息发送给 `BackgroundWorker`。
  - **`PdfPage.render`**: 请求后台渲染页面，返回包含像素数据的 `PdfImage`。

## 第 4 层：Flutter 接口层 (Flutter Interface Layer)
**所属包**: `package:pdfrx`

这一层提供开发者直接使用的 UI 组件。

- **职责**:
  1. **UI 展示**: 将引擎层返回的图像数据绘制到屏幕上。
  2. **交互处理**: 处理手势（缩放、平移、点击）、滚动和页面跳转。
  3. **状态管理**: 管理当前页码、缩放比例、可见区域等状态。

- **关键组件**:
  - **`PdfViewer`**: 核心 Widget。它监听 `PdfDocument` 的状态变化。
  - **`PdfViewerController`**: 允许外部代码控制查看器（如 `controller.goToPage(5)`）。
  - **`_PdfPageImageCache`**: 缓存已渲染的页面图片，优化滚动性能。
  - **`CustomPainter` (`_paintPages`)**: 实际的绘图逻辑。它获取 `PdfPage.render` 生成的图片数据，使用 `canvas.drawImage` 绘制到 Flutter 的画布上。

---

## 数据流向示例：渲染一页 PDF

1. **UI 层 (`PdfViewer`)**: 用户滚动到第 5 页。组件调用 `page.render()` 请求该页图像。
2. **引擎层 (`pdfrx_engine`)**: `PdfPage` 将渲染请求（包含页码、缩放比例等）发送给 `BackgroundWorker`。
3. **后台 Isolate**: Worker 接收请求。
4. **FFI 层 (`pdfium_dart`)**: Worker 调用 `FPDF_RenderPageBitmap` C 函数。
5. **原生层 (`libpdfium`)**: PDFium 库在内存中绘制像素数据。
6. **返回**: 像素数据通过 Isolate 返回给 UI 线程。
7. **UI 层**: Flutter 将像素数据转换为 `ui.Image` 并绘制到屏幕。
