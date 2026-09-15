# File Operation Styler changelog

## 1.0.1（简体中文汉化分支 Z2549）

本版本为 [digart11/File-Operation-Styler](https://github.com/digart11/File-Operation-Styler) 的
简体中文汉化 + 新版 shell32 兼容性修复分支，核心实现仍来自上游 1.0.0。

### 兼容性修复（原版 1.0.0 在 Windows 11 24H2 / shell32 10.0.26100.4768 上完全失效）

- **修复：Windhawk 引擎从不注入该 Mod。**
  新版 shell32 不再生成普通析构符号 `??1OperationTileElement@@`（已内联进删除析构），
  而原版把它列为硬性必需，导致符号预解析失败、DLL 从未被加载。
  现改为可选符号，并以 `scalar / vector deleting destructor`（`??_E` / `??_G`）作为等价替代；
  清理动作改写为幂等的 `ReleaseOperationTilePresentationResources`，同时挂两个删除析构亦安全。
- **修复：磁贴根元素抓不到。**
  24H2 把 DirectUI 根元素改名为 `idOperationTile_old` / `idTileHeader_old`，原版硬编码裸名。
  现对 `""` / `_old` / `_New` 三种后缀做容错匹配。
- **修复：布局校验失败，仍然不上皮肤。**
  原版按 `eltRateChart_New` 查找速率曲线，24H2 用的是裸名 `eltRateChart`（后缀被焊进基名），
  追加后缀无法拼出裸名。现改为**候选名生成器**（原样 → 原样+后缀 → 剥掉后缀的裸基名 → 裸基名+后缀），
  统一入口 `FindDescendentBySkinId()`，一处修复覆盖全部约 20 个元素名。

以上修复对旧版 shell32 **保持向后兼容**（原拼写仍然可以命中）。

### 简体中文汉化

- 元数据与设置项按 **Windhawk 官方本地化规范**叠加中文，英文原文一律保留：
  `@name:zh-CN`、`@description:zh-CN`、`@author:zh-CN`、
  `$name:zh-CN:`、`$description:zh-CN:`、`$options:zh-CN:`。
- 自绘界面文字改为按**系统 UI 语言**自动切换中英文
  （完成 / 更多详细信息 / 收起详细信息 / 取消 / 正在计算… / 正在删除项目 / 文件操作进行中）。
- 特殊状态对话框标题补充中文匹配（替换或跳过文件 / 文件正在使用 / 文件夹正在使用 / 找不到项目），
  修复中文版 Windows 上因标题本地化导致的漏判。
- Mod 详情页说明以中文为主，末尾附英文小节
  （`WindhawkModReadme` 块不参与 Windhawk 的本地化机制）。

## 1.0.0

- Initial public release.
- Modern custom layout for Windows 11 copy, move, delete, and recycle operations.
- Compact and expanded views with circular percentage indicator.
- Shows transferred size, current item, remaining items, speed, and estimated time.
- Progress graph in More Details view.
- Supports multiple simultaneous file operations.
- Preserves native Pause, Resume, Cancel, conflict, and error handling.
- Includes built-in themes and customizable colors, typography, and progress styling.


## 0.12.0 architecture-alpha

- Replaced the visible normal-operation DirectUI composition with one opaque,
  custom-rendered presentation surface per native operation tile.
- The custom surface now owns the description, transferred/total summary,
  items remaining, speed, time remaining, completion bar, speed graph,
  Pause/Resume button, and Cancel button. The existing circular completion
  control remains custom rendered.
- Added a custom shared More/Fewer Details footer. Pause/Resume, Cancel, and
  More/Fewer invoke the corresponding live DirectUI button through
  `DirectUI::Button::DefaultAction`; no file-operation logic or synthetic mouse
  input is implemented by the mod.
- Removed native DirectUI bounds, margins, padding, fonts, colors, chart size,
  progress-bar size, and container backgrounds from the active normal-mode
  layout path. The old mutation helpers remain temporarily in the source only
  as unreachable migration/diagnostic code and are no longer startup
  requirements.
- Retained native DirectUI discovery for tile identity, normal/special state,
  copy/move/delete classification, native description text, and native action
  lookup. Native progress, byte/item counters, and rate calculation remain the
  data sources.
- All custom HWND and GDI+ geometry starts as 96-DPI logical units and is
  converted once using the actual `OperationStatusWindow` DPI.
- Native controls stay alive underneath the opaque normal presentation. On a
  conflict, permission, file-in-use, or other detected special state, all
  custom surfaces are hidden and the untouched Explorer UI is revealed. The
  custom surfaces are recreated/repositioned when normal progress resumes.
- Presets, custom colors, custom fonts, logical layout settings, graph opacity,
  and element visibility settings continue to apply to custom-rendered UI.
  Legacy `nativeFont`, `nativeDetailSize`, `nativeValueWeight`,
  `nativeLabelWeight`, `actionSize`, and `actionWeight` settings are retained
  for settings compatibility but are temporarily inactive because 0.12 does
  not skin native DirectUI visuals.
