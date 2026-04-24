# SDKJS 架构与调用链速览（增强版）

> 本文基于仓库源码的“可复查路径 + 实际定位命令”整理，重点覆盖：
> - 核心入口
> - 文档通信
> - 绘制链路
> - 鉴权 / License / 限制
> - 开发调试与构建

---

## 0. 仓库定位与范围

- 项目定位：ONLYOFFICE 前端编辑器 SDK（Word/Cell/Slide/Visio/PDF）
- 关键目录：`common/`、`word/`、`cell/`、`slide/`、`visio/`、`pdf/`、`build/`、`configs/`

---

## 1. 核心入口（Editor API）

### 1.1 统一基类

- `common/apiBase.js` 的 `baseEditorsApi` 是各编辑器共享入口与生命周期基类：
  - 打开文档（open）
  - 协同初始化（coauthoring）
  - 权限与限制
  - 通用事件分发

### 1.2 各编辑器入口类

- Word: `word/api.js` -> `asc_docs_api(config)`
- Cell: `cell/api.js` -> `spreadsheet_api(config)`
- Slide: `slide/api.js` -> `asc_docs_api(config)`
- Visio: `visio/api.js` -> `VisioEditorApi(config)`
- PDF: `pdf/api.js` -> `PDFEditorApi(config)`

### 1.3 我在仓库里如何定位这些入口（命令）

```bash
rg -n "function asc_docs_api|function spreadsheet_api|function VisioEditorApi|function PDFEditorApi" \
  word/api.js cell/api.js slide/api.js visio/api.js pdf/api.js

rg -n "baseEditorsApi\.prototype\.openDocument|_onEndLoadSdk|asc_LoadDocument" common/apiBase.js
```

---

## 2. 构建架构（Build）

### 2.1 构建工具链

- `build/Gruntfile.js` + `google-closure-compiler`
- `configs/*.json` 驱动每个编辑器的打包顺序（`min/common`）

### 2.2 关键任务

- `compile-word / compile-cell / compile-slide / compile-visio`
- `compile-sdk`
- `copy-other`
- 默认任务：`clean-deploy -> compile-sdk -> copy-other`

### 2.3 我在仓库里如何定位构建路径（命令）

```bash
rg -n "registerTask\('compile-|registerTask\('default'|closure-compiler|getFilesMin|getFilesAll" build/Gruntfile.js

sed -n '1,140p' configs/word.json
sed -n '1,90p'  configs/cell.json
sed -n '1,90p'  configs/slide.json
sed -n '1,90p'  configs/visio.json
```

---

## 3. 文档通信链路（打开、协同、RPC）

### 3.1 打开流程（主线）

1. `baseEditorsApi._getOpenCmd()` 组装 open 参数（id/url/format/outputformat 等）
2. `asc_LoadDocument()` 调用 `CoAuthoringApi.auth(viewMode, openCmd)`
3. 协同层建立会话后回传文件数据
4. `onEndLoadFile()` -> 各编辑器 `openDocument(file)`

### 3.2 协同通道

- 包装层：`CDocsCoApi`
- 底层：`DocsCoApi`
- 报文能力：`auth/openDocument/rpc/message/cursor/...`

### 3.3 我在仓库里如何定位通信代码（命令）

```bash
rg -n "_getOpenCmd|asc_LoadDocument|_coAuthoringInit|onEndLoadFile|openDocument\(" common/apiBase.js

rg -n "getAuthCommand|_send\(|openDocument|callPRC|sendMessage|sendCursor|expiredToken|refreshToken" common/docscoapi.js
```

---

## 4. 绘制链路（以 Word 为例）

### 4.1 UI 与画布初始化

- `word/api.js -> CreateComponents()`：创建 `id_viewer`、`id_viewer_overlay`、`id_target_cursor`
- `_onEndLoadSdk()`：构造 `WordControl = new AscCommonWord.CEditorPage(this)`，随后 `CreateComponents()` + `WordControl.Init()`

### 4.2 DrawingDocument 绑定

- `word/Drawing/HtmlPage.js` 构造 `CEditorPage`
- 创建并绑定 `m_oDrawingDocument`
- 绑定 overlay 与 input target

### 4.3 绘制循环

- `StartMainTimer()` 启动 `PaintMessageLoop`
- `onTimerScroll` 驱动 target 更新、页面刷新、协同光标更新
- `OnPaint` 执行可见页绘制（page draw + overlay）

### 4.4 我在仓库里如何定位绘制代码（命令）

```bash
rg -n "CreateComponents|_onEndLoadSdk|WordControl\.Init\(|id_viewer|id_viewer_overlay|id_target_cursor" word/api.js

rg -n "function CEditorPage|m_oDrawingDocument|StartMainTimer|onTimerScroll|OnPaint|PaintMessageLoop" word/Drawing/HtmlPage.js
```

---

## 5. 鉴权、License、限制操作

### 5.1 会话鉴权（token/jwt）

- `DocsCoApi.init` 保存 token、permissions、jwtOpen
- `getAuthCommand()` 发送 `token/permissions/jwtOpen/jwtSession`
- `refreshToken/expiredToken` 处理会话更新或过期

### 5.2 License 能力控制

- `license` 消息 -> `_onLicense`
- `apiBase` 将许可映射为 `asc_CAscEditorPermissions`
- `onLicenseChanged` 可动态调整限制参数（如上传大小/格式、maxChangesSize）

### 5.3 操作限制（Restriction + Permissions）

- `restrictions` 位控制：View / OnlyForms / OnlyComments / OnlySignatures
- `canEdit()` 由 `isViewMode + restrictions` 决定
- `DocInfo.permissions`（例如 copy）影响 `copyEnabled`

### 5.4 我在仓库里如何定位鉴权/限制代码（命令）

```bash
rg -n "license|onLicense|onLicenseChanged|expiredToken|refreshToken|getAuthCommand|jwt|token|permissions" \
  common/docscoapi.js common/apiBase.js

rg -n "asc_setRestriction|asc_addRestriction|asc_removeRestriction|canEdit\(|isRestriction" common/apiBase.js

rg -n "asc_getPermissions|copyEnabled" common/apiBase.js
```

---

## 6. 典型“调用链”速记

### 6.1 打开文档

`asc_LoadDocument()`
-> `CoAuthoringApi.auth(..., openCmd)`
-> 协同返回文档流
-> `onEndLoadFile()`
-> `editor.openDocument(file)`

### 6.2 刷新渲染

`WordControl.StartMainTimer()`
-> `onTimerScroll()`
-> `OnPaint()`
-> page draw / overlay draw

### 6.3 上传图片（带会话）

`ShowImageFileDialog(..., jwt)`
-> `UploadImageFiles(..., jwt)`
-> `_addImageUrl(urls, obj)`

---

## 7. 开发调试建议

### 7.1 构建

```bash
cd build
npm ci
grunt
```

### 7.2 开发模式（便于调试源码）

```bash
grunt --level=WHITESPACE_ONLY
grunt develop
```

### 7.3 与 web-apps 联动（如需）

- 根目录 `Makefile` 提供与 `../web-apps` 的集成构建目标

---

## 8. 本文补充说明

- 本文是“导航与定位说明”，不是 API 全量手册。
- 若你需要，我可以继续追加：
  1) `Word` 详细时序图（打开到首帧渲染）
  2) `Cell` 渲染与公式链路
  3) `License/Restriction` 的“可执行检查清单”

