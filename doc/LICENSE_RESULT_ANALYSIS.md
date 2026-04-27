# window.g_asc_plugins.api.licenseResult 赋值分析

## 概述
本文档分析 SDKJS 项目中 `window.g_asc_plugins.api.licenseResult` 对象的初始化、赋值流程和相关逻辑。

---

## 1. 全局对象初始化

### 1.1 g_asc_plugins 创建
- **文件**: `common/plugins.js` (第 1894 行)
- **关键代码**:
```javascript
window.g_asc_plugins = new CPluginsManager(api);
window["g_asc_plugins"] = window.g_asc_plugins;
```

**说明**: 
- `g_asc_plugins` 是 `CPluginsManager` 的实例
- 传入的 `api` 参数来自调用者（通常是主应用API实例）
- 在 `CPluginsManager` 构造函数中，`this.api = api` 和 `this["api"] = this.api` 完成属性赋值

### 1.2 g_asc_plugins.api 的指向
- `window.g_asc_plugins.api` 就是传入的 API 实例
- 该 API 实例继承自 `apiBase.js` 的基类

---

## 2. licenseResult 初始化

### 2.1 定义位置
- **文件**: `common/apiBase.js` (第 155 行)
- **初始值**:
```javascript
this.licenseResult = null;
```

**说明**: API 实例被创建时，`licenseResult` 初始化为 `null`，默认状态下没有许可证数据。

---

## 3. licenseResult 赋值流程

### 3.1 回调函数设置
- **文件**: `common/apiBase.js` (第 1966-1970 行)
- **关键代码**:
```javascript
this.CoAuthoringApi.onLicense = function(res)
{
    t.licenseResult = res;        // 核心赋值操作
    t.isOnLoadLicense = true;
    t._onEndPermissions();
};
```

**说明**:
- `onLicense` 是协作编辑API中的一个回调函数
- 当服务器返回许可证信息时，该回调被触发
- 收到的 `res` 对象被直接赋给 `t.licenseResult`

### 3.2 回调触发源
- **文件**: `common/docscoapi.js` (第 1499-1505 行)
- **方法**: `DocsCoApi.prototype._onLicense`
- **关键代码**:
```javascript
DocsCoApi.prototype._onLicense = function(data) {
    if (!this.isLicenseInit) {
        this.isLicenseInit = true;
        this.onLicense(data['license']);  // 触发回调
        if (this.onAiPluginSettings) {
            this.onAiPluginSettings(data['aiPluginSettings']);
        }
    }
};
```

**说明**:
- `_onLicense` 方法负责处理来自服务器的许可证数据
- `data['license']` 被传递给 `onLicense` 回调
- `isLicenseInit` 标志确保初始化只执行一次

### 3.3 服务器消息触发
- **文件**: `common/docscoapi.js` (第 1849 行或附近)
- 当 WebSocket 或 HTTP 连接接收到 `license` 类型的消息时，调用 `_onLicense` 方法

---

## 4. licenseResult 的数据结构

### 4.1 包含的主要属性
根据 `common/apiBase.js` 第 1856-1872 行的使用，`licenseResult` 包含以下字段：

| 属性 | 说明 | 用途 |
|------|------|------|
| `type` | 许可证类型 | 确定许可级别 |
| `branding` | 品牌信息 | 设置品牌化选项 |
| `customization` | 自定义配置 | 应用自定义设置 |
| `light` | 轻量级模式 | 指示是否为轻量版 |
| `mode` | 许可证模式 | 编辑模式或查看模式 |
| `rights` | 用户权限 | 定义用户操作权限 |
| `buildVersion` | 构建版本 | 版本信息 |
| `buildNumber` | 构建号 | 版本号 |
| `liveViewerSupport` | 实时查看支持 | 协作查看能力 |
| `protectionSupport` | 保护支持 | 文档保护能力 |
| `isAnonymousSupport` | 匿名支持 | 匿名用户能力 |
| `advancedApi` | 高级API支持 | 扩展功能可用性 |
| `plugins` | 插件支持 | 插件加载权限 |

### 4.2 使用示例
- **文件**: `common/editorscommon.js` (第 2191 行)
```javascript
if (!window.g_asc_plugins.api.licenseResult || 
    !window.g_asc_plugins.api.licenseResult['advancedApi'])
{
    // advancedApi 不可用的处理逻辑
}
```

---

## 5. 时序图

```
应用启动
    ↓
创建 API 实例 (apiBase.js)
    ├─ licenseResult = null
    └─ 设置 CoAuthoringApi.onLicense 回调
    ↓
创建 CPluginsManager
    └─ window.g_asc_plugins = new CPluginsManager(api)
    ↓
建立服务器连接 (CoAuthoringApi)
    ↓
接收 "license" 消息
    ↓
DocsCoApi._onLicense(data)
    ↓
this.onLicense(data['license']) [调用回调]
    ↓
apiBase.js onLicense 回调执行
    └─ t.licenseResult = res
    └─ t.isOnLoadLicense = true
    └─ t._onEndPermissions()
    ↓
window.g_asc_plugins.api.licenseResult 可用
    ↓
应用可检查许可证信息
```

---

## 6. 关键代码位置总结

| 步骤 | 文件 | 行号 | 说明 |
|------|------|------|------|
| 初始化 | apiBase.js | 155 | 初始化 licenseResult = null |
| 回调设置 | apiBase.js | 1966-1970 | 设置 onLicense 回调函数 |
| 回调触发 | docscoapi.js | 1499-1505 | _onLicense 方法处理服务器数据 |
| 全局对象 | plugins.js | 1894 | 创建 g_asc_plugins 实例 |
| 使用检查 | editorscommon.js | 2191 | 检查 licenseResult.advancedApi |

---

## 7. 注意事项

1. **异步加载**: `licenseResult` 的赋值是异步的，取决于服务器响应时间
2. **单次初始化**: `isLicenseInit` 标志确保许可证信息只在第一次接收时被处理
3. **权限检查**: 许可证信息影响应用的多个功能模块（插件、品牌、自定义等）
4. **空值检查**: 使用时应始终检查 `licenseResult` 是否为 `null` 或特定属性是否存在

---

## 8. 相关方法和事件

### 事件
- `asc_onLicenseChanged`: 许可证变更事件（见 apiBase.js 1996 行）

### 方法
- `CoAuthoringApi.onLicense()`: 许可证接收回调
- `_onEndPermissions()`: 权限设置完成处理
- `isPluginsSupport()`: 检查插件支持（apiBase.js 3632 行）

---

## 9. 许可证相关的初始化流程

应用启动时的典型流程：
1. 应用初始化 API 实例
2. API 实例设置协作编辑回调（包括 `onLicense`）
3. 应用建立与服务器的连接
4. 服务器发送许可证信息
5. `DocsCoApi` 接收并处理许可证消息
6. 触发 `onLicense` 回调，更新 `apiBase` 实例中的 `licenseResult`
7. `_onEndPermissions()` 完成权限初始化
8. 应用各模块可开始检查和使用许可证信息

