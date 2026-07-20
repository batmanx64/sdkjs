# sdkjs WebAssembly 模块分析

## 概述

sdkjs 内置 5 个 WASM 模块，均通过 **Emscripten** 将 C/C++ 开源库编译为 WebAssembly，并保留 asm.js 降级方案。

---

## 模块一览

| # | WASM 文件 | 位置 | 开源项目 | 用途 |
|---|-----------|------|----------|------|
| 1 | `fonts.wasm` | `common/libfont/engine/` | **FreeType** + **HarfBuzz** + **Zlib** + **Hyphen** | 字体解析、文字塑形、连字符、压缩 |
| 2 | `spell.wasm` | `common/spell/spell/` | **Hunspell** | 拼写检查 |
| 3 | `engine.wasm` | `common/hash/hash/` | **xmlsec / OpenSSL** (SHA) | 哈希运算 |
| 4 | `zlib.wasm` | `common/zlib/engine/` | **Zlib** | 压缩/解压 (OOXML ZIP) |
| 5 | `drawingfile.wasm` | `pdf/src/engine/` | **自研 C++ PDF 引擎** | PDF 渲染 |

---

## 详细来源与编译配置

### 1. fonts.wasm — 字体引擎 (最复杂)

编译配置: `core/DesktopEditor/fontengine/js/libfont.json`

| 开源组件 | 版本 | 用途 |
|----------|------|------|
| **FreeType** | 2.10.4 | 字体文件解析、字形加载、光栅化 |
| **HarfBuzz** | latest | 文字塑形 (阿拉伯语、印度语系等复杂文字) |
| **Zlib** | 1.2.11 | ZIP 压缩 (内嵌于 FreeType 的 gzip/zlib 流) |
| **Hyphen** | - | 连字符断词 |
| **Brotli** | - | FreeType WOFF2 字体压缩解码 |
| **CxImage** | - | 图片编解码 (JPEG/PNG/TIFF/GIF 等) |
| LibJPEG / LibPNG / LibTIFF | - | 图片格式支撑 |

导出的关键 C API（通过 `AscFonts.*` JS 包装调用）:

```
_ASC_FT_Init / _ASC_FT_Done_FreeType     — FreeType 生命周期
_ASC_FT_Open_Face / _ASC_FT_Done_Face     — 字体打开/关闭
_ASC_FT_Load_Glyph / _ASC_FT_Get_Glyph_* — 字形加载与度量
_ASC_HB_ShapeText                         — HarfBuzz 文字塑形
_hyphenWord                               — 连字符断词
_Zlib_*                                   — ZIP 读写
_Raster_Encode / _Raster_DecodeFile       — 图片编解码
```

### 2. spell.wasm — 拼写检查器

编译配置: `core/Common/3dParty/hunspell/hunspell.json`

| 开源组件 | 版本 | 用途 |
|----------|------|------|
| **Hunspell** | (LibreOffice 使用的拼写引擎) | 拼写检查与建议 |

导出的 C API:

```
_Spellchecker_Create / _Spellchecker_Destroy — 引擎生命周期
_Spellchecker_AddDictionary                  — 加载词典
_Spellchecker_Spell                          — 拼写检查
_Spellchecker_Suggest                        — 拼写建议
```

### 3. engine.wasm — 哈希引擎

编译配置: `core/DesktopEditor/xmlsec/src/wasm/hash/hash.json`

| 开源组件 | 用途 |
|----------|------|
| **OpenSSL / xmlsec** (部分) | SHA 哈希运算 |

用于文档完整性校验与签名验证。

### 4. zlib.wasm — 压缩引擎

编译配置: `core/OfficeUtils/js/zlib.json`

| 开源组件 | 版本 | 用途 |
|----------|------|------|
| **Zlib** | 1.2.11 | DEFLATE 压缩/解压 |

OOXML (.docx/.xlsx/.pptx) 本质是 ZIP 包，此模块负责高速解压。

> **注意**: fonts.wasm 内也包含了 Zlib，但那是给 FreeType 内部用的。此模块是独立暴露给 JS 层做通用 ZIP 操作。

### 5. drawingfile.wasm — PDF 渲染引擎

编译配置: `core/DesktopEditor/graphics/pro/js/drawingfile.json`

| 组件 | 说明 |
|------|------|
| **自研 C++ PDF 引擎** | ONLYOFFICE 自己开发的 PDF 渲染器 |

导出的 C API:

```
_InitializeFontsBin / _SetFontBinary — 字体初始化
_Open                                — 打开 PDF 文档
_GetPixmap                           — 获取渲染像素图
_ScanPage                            — 扫描页面结构
```

---

## 编译方式

所有 WASM 模块通过 `core/Common/js/make.py` 统一编译：

```
# 编译流程:
core/Common/js/make.py <target.json>
  |
  ├── 1. 自动 git clone emsdk (emscripten-core/emsdk)
  ├── 2. 读取 target.json 中的:
  │     ├── compile_files_array — C/C++ 源文件列表
  │     ├── include_path        — 头文件路径
  │     ├── define              — 编译宏
  │     └── exported_functions  — 导出到 JS 的函数
  ├── 3. 执行:
  │     emcc -O3 -s WASM=1 -o <name>.js <sources>  (WASM 版)
  │     emcc -O3 -s WASM=0 -o <name>_ie.js <sources> (asm.js 回退)
  └── 4. 输出:
        ├── <name>.wasm     (WASM 二进制)  → 部署到 sdkjs/*/engine/
        ├── <name>.js       (JS glue)      → 部署到 sdkjs/*/engine/
        └── <name>_ie.js    (asm.js 回退)  → 部署到 sdkjs/*/engine/
```

**在 sdkjs/ 目录下找不到 C 源码的原因**：C/C++ 源码全部在 `core/` 目录下，sdkjs/ 只包含编译产物。编译发生在 `core/` 的 CI 流程中。

---

## 浏览器加载机制

5 个模块采用统一的 feature-detection 模式：

```javascript
// 伪代码 — 每个模块的 loader 都类似
if (window["WebAssembly"] && window["WebAssembly"].instantiate) {
  // 支持 WASM → 加载 .wasm 版本
  loadScript("fonts.js")   // Emscripten glue, 内部 fetch fonts.wasm
} else {
  // 不支持 → 加载 asm.js 降级版本
  loadScript("fonts_ie.js") // 纯 JS, 无 .wasm 依赖
}
```

每个模块的 loader 与 glue 文件：

| 模块 | Loader | WASM Glue | asm.js 回退 | 原生回退 |
|------|--------|-----------|-------------|---------|
| fonts | `common/libfont/loader.js` | `fonts.js` | `fonts_ie.js` | `fonts_native.js` |
| spell | `common/spell/spell.js` | `spell/spell.js` | `spell/spell_ie.js` | - |
| hash | `common/hash/hash.js` | `hash/engine.js` | `hash/engine_ie.js` | - |
| zlib | `common/zlib/zlib.js` | `zlib/engine/zlib.js` | `zlib/engine/zlib_ie.js` | - |
| pdf | `pdf/src/viewer.js:895` | `pdf/src/engine/drawingfile.js` | `pdf/src/engine/drawingfile_ie.js` | - |

---

## 如何查看 WASM 编译的模块源码

### 方法 1: 从 JSON 配置追溯 (推荐)

每个 WASM 模块对应一个 JSON 编译配置，其中 `compile_files_array` 列出了所有源文件路径：

```
core/DesktopEditor/fontengine/js/libfont.json
  └─ compile_files_array[0].folder + files  →  FreeType 源码
  └─ compile_files_array[1].folder + files  →  HarfBuzz 源码
  └─ ...

core/Common/3dParty/hunspell/hunspell.json
  └─ compile_files_array[0].folder + files  →  Hunspell 源码
```

### 方法 2: JSON 关键字段解读

```json
{
  "name": "fonts",                              // 输出文件名
  "wasm": true,  "asm": true,                   // 同时编译 WASM + asm.js
  "compiler_flags": ["-O3"],                     // Emscripten 编译标志
  "exported_functions": ["_ASC_FT_Init", ...],   // 暴露给 JS 的 C 函数
  "include_path": [...],                         // 头文件搜索路径
  "compile_files_array": [{                      // 源文件列表
    "folder": "./../../graphics/pro/js/freetype-2.10.4/src",
    "files": ["base/ftdebug.c", "autofit/autofit.c", ...]
  }]
}
```

### 方法 3: JS glue 中查找对应关系

在 `fonts.js` 或 `fonts_ie.js` 中搜索 C 函数名字符串，如 `"__ABI"` 开头的 `_ASC_FT_Load_Glyph`，可以找到 WASM 导出的函数表。

---

## 附录: 关联的 JSON 编译配置索引

| WASM 模块 | JSON 配置路径 |
|-----------|--------------|
| fonts.wasm | `core/DesktopEditor/fontengine/js/libfont.json` |
| spell.wasm | `core/Common/3dParty/hunspell/hunspell.json` |
| engine.wasm (hash) | `core/DesktopEditor/xmlsec/src/wasm/hash/hash.json` |
| zlib.wasm | `core/OfficeUtils/js/zlib.json` |
| drawingfile.wasm | `core/DesktopEditor/graphics/pro/js/drawingfile.json` |
