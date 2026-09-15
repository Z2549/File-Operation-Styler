# 文件操作窗口美化（File Operation Styler）· 简体中文汉化版

> 本仓库是 [**digart11/File-Operation-Styler**](https://github.com/digart11/File-Operation-Styler) 的
> **简体中文汉化 + 新版 shell32 兼容性修复** 分支（fork）。
>
> 原版作者：**digART**　·　许可：**GPL-3.0**　·　上游仓库：[digart11/File-Operation-Styler](https://github.com/digart11/File-Operation-Styler)
>
> 本分支沿用同一许可，原作者的署名、许可与英文原文一律保留。英文说明见 [README_EN.md](README_EN.md)。

用现代化的自绘界面替换 Windows 11 默认的文件操作窗口（复制 / 移动 / 删除 / 回收站），
同时**完全保留系统原生文件操作引擎**——真正的复制、移动、删除、冲突与错误处理仍然由 Windows 负责。

## 截图

![File Operation Styler](images/file-operation-styler.png)

### 原生界面 vs 美化后

![Default vs File Operation Styler](images/file-operation-styler-compare.png)

### 内置主题

![File Operation Styler Themes](images/file-operation-styler-themes.png)

## 功能

- 现代化的复制 / 移动进度窗口
- 环形百分比指示器
- 已传输大小、剩余项目数、速度与预计剩余时间
- 「更多详细信息」视图中的速度曲线图
- 同一个窗口内可同时显示多个文件操作
- 暂停 / 继续 / 取消按钮
- 与系统的冲突、错误对话框正常共存
- 多套内置主题
- 可自定义颜色、字体、文字大小与进度条粗细

## 支持的版本

| Windows / shell32 版本 | 状态 |
| --- | --- |
| Windows 11 **24H2**（shell32 `10.0.26100.4768`，build 26100） | ✅ **已实测通过** —— 本汉化分支的主要目标 |
| 其他 Windows 11（更早的 shell32，如 22621 / 22631） | ⚠️ 代码层面已保留旧拼写兼容（`idOperationTile`、`eltRateChart_New` 等仍可命中），但**未在真机实测** |
| Windows 10 及更早版本 | ❌ 不支持 —— 本 Mod 针对的是 Windows 11 的文件操作窗口 |

> **一句话结论**：原版 **1.0.0** 在 Windows 11 24H2 上会**完全失效**（装上后没有任何效果）。
> 本分支修复后已在该版本上实测可用，并且**没有破坏对旧版 shell32 的兼容**。

## 修复详情

原版把 shell32 的**符号名**与 **DirectUI 元素 id** 写成了硬性假设，系统版本一变就整体失效。
在 24H2 上这是**三层互相独立**的故障——修好一层才会露出下一层，所以「改一处就生效」是不成立的。

### 问题 1：Windhawk 引擎从不注入该 Mod

- **现象**：Mod 显示已启用、符号也在解析，但 `explorer.exe` 的模块列表里**从来没有**这个 DLL。
- **根因**：24H2 的 shell32 **不再生成普通析构符号** `??1OperationTileElement@@`（它被内联进了删除析构），
  而原版把这个符号列为**硬性必需**。Windhawk 是「先解析符号、解析不过就不加载」的设计，
  于是一个符号的缺失导致整个 Mod 从未被加载。
- **改法**：
  1. 该符号在符号表里改为**可选**（`SYMBOL_HOOK` 第 4 字段 `false` → `true`）；
  2. 新增 `scalar / vector deleting destructor`（`??_E` / `??_G`）作为**等价替代**并挂上钩子
     —— 有虚析构的类，`delete` 走 vtable 最终就是落到删除析构；普通析构被内联时，
     删除析构是**唯一还能挂到的**析构入口；
  3. 从「硬失败清单」里删掉该符号，安装钩子处加空指针保护，可选钩子失败只告警不中止；
  4. 清理动作改写为**幂等**函数（`ReleaseOperationTilePresentationResources`），
     这样 `??_E` 与 `??_G` 同时挂上也安全。
- **验证**：符号缓存里该条目从 `error:` 变为已解析；两个删除析构解析到**同一 RVA**
  （被链接器 ICF 折叠）；DLL **首次出现在 explorer 的模块列表**。

### 问题 2：磁贴根元素抓不到，每次都保留原生界面

- **现象**：DLL 加载了、钩子也触发了（日志有 `hook.CreateTileElement-fired`），但界面**完全没变**。
- **根因**：24H2 把操作状态磁贴的 DirectUI **根元素改名为** `idOperationTile_old` / `idTileHeader_old`，
  原版硬编码裸名 `lstrcmpW(resourceName, L"idOperationTile")` ⇒ 永远匹配不上 ⇒
  根元素恒为 `NULL` ⇒ 每次都在「没有根元素」这个早退点放弃。
- **改法**：根元素匹配改为对 `""` / `_old` / `_New` 三种后缀**循环容错匹配**。
- **验证**：埋点日志出现 `collect.idOperationTile=[_old]`、`collect.idTileHeader=[_old]`，
  即根元素被成功捕获。

### 问题 3：拿到根元素后布局校验失败，仍然不上皮肤

- **现象**：`bail.3-unsupported-layout`，界面依旧不变。
- **根因**：原版按 `eltRateChart_New` 查找速率曲线，而 24H2 用的是**裸名** `eltRateChart`
  —— 后缀被「焊」进了基名。**单纯往名字后面追加后缀，永远拼不出裸名**，
  于是速率曲线恒为 `NULL`，结构校验随之失败。
- **改法**：新增**候选名生成器**，为每个逻辑名生成有序去重的候选表：

  ```
  1) 调用方原样传的名字          ("eltRateChart_New")
  2) 原名字 + 各已知后缀         ("eltRateChart_New_old")
  3) 剥掉末尾已有后缀的裸基名     ("eltRateChart")      ← 24H2 靠这条命中
  4) 裸基名 + 各已知后缀         ("eltRateChart_old")
  ```

  统一入口 `FindDescendentBySkinId()`，**一处修复覆盖全部约 20 个元素名**
  （`eltSummary` / `eltDetails` / `eltRateChart` / `eltProgressBar` …）。

  > 顺带一个反面教训：最初尝试过「子元素优先套用根元素匹配到的后缀」这种启发式
  > （假设「根被改名 ⇒ 子也被改名」），实测**在 24H2 上不成立**——根是 `_old`，子却是裸名。
  > 所以最终改成了**确定性的候选表**，不使用启发式排序。
- **验证**：埋点出现 `applied.SUCCESS=1`、`circle.CREATED-ok=1`，界面实际发生变化。

## 汉化说明

汉化遵循 **Windhawk 官方本地化规范**（`@key:lang` 形式），核心原则是
**英文原文一律保留，只在后面叠加 `zh-CN` 变体**，英文用户完全不受影响：

| 位置 | 写法 |
| --- | --- |
| Mod 名称 | `// @name`（英文）+ `// @name:zh-CN 文件操作窗口美化` |
| Mod 描述 | `// @description` + `// @description:zh-CN ...` |
| 作者 | `// @author` + `// @author:zh-CN ...` |
| 设置项名称 | `$name:` + `$name:zh-CN:` |
| 设置项描述 | `$description:` + `$description:zh-CN:` |
| 下拉选项文字 | `$options:` + `$options:zh-CN:`（在同一位置再列一遍选项） |

另外两处不属于 Windhawk 官方本地化机制、但影响中文体验的地方也做了处理：

- **自绘界面文字**（完成 / 更多详细信息 / 收起详细信息 / 取消 / 正在计算… / 正在删除项目）：
  官方机制不覆盖运行期绘制的文字，因此改为按**系统 UI 语言**自动切换
  （简体中文系统显示中文，其余显示英文），实现为 `UiText(英文, 中文)` 辅助函数。
- **特殊状态对话框标题**：原版只按英文标题识别「替换或跳过文件 / 文件正在使用 /
  文件夹正在使用 / 找不到项目」，中文版 Windows 上这些标题是本地化的，只匹配英文会漏判。
  本分支中英同时匹配（中文额外放宽为包含匹配）。

> `// ==WindhawkModReadme==` 块**不支持**本地化（Windhawk 只解析单个 readme 块），
> 所以 Mod 详情页的说明以中文为主，末尾附一段英文小节。

## 安装

### 方式一：从本仓库安装（推荐）

1. 打开 Windhawk → **探索**（Explore）→ 右上角 **安装 Mod** → 选择从 URL / 源码安装；
2. 指向本仓库的 `file-operation-styler.wh.cpp`
   （`https://raw.githubusercontent.com/Z2549/File-Operation-Styler/master/file-operation-styler.wh.cpp`）；
3. 安装完成后，**重启 `explorer.exe`** 或注销/重启，让 Mod 注入生效；
4. 在资源管理器里手动做一次复制 / 移动（**必须是从资源管理器 UI 发起的操作**，见下方「已知限制」）。

### 方式二：本地编译

```powershell
# 用 Windhawk 自带的命令行工具直接编译
& "D:\Program\Windhawk\windhawk-cli.exe" mod compile file-operation-styler
```

或把 `file-operation-styler.wh.cpp` 放到 Windhawk 的便携数据目录
`<AppData>\ModsSource\file-operation-styler.wh.cpp`，再用 Windhawk 编辑器编译。

## ⚠️ 更新注意

本分支的 `@id` 与原版**保持一致**（`file-operation-styler`），原因是：
设置项（主题、颜色、字号…）都按这个 id 存储，改 id 会让你的设置全部丢失。

副作用是：如果 Windhawk 内置商店里存在同 id 的官方版本，
**商店的自动更新可能会把本分支覆盖回原版**（从而丢掉兼容性修复）。
如果遇到这种情况，关掉该 Mod 的自动更新，或重新安装本仓库的源码即可。

本分支已把 `@version` 提升到 **1.0.1**（上游为 1.0.0）以便区分。

## 已知限制

- **Windows 10 及更早版本不支持**：Mod 针对的是 Windows 11 的文件操作窗口。
- **较早的 Windows 11 shell32 未经真机实测**：代码层面保留了旧拼写兼容，但没有设备可验证。
- 中文对话框标题的识别是**「尽力而为」**的启发式（原版对此的注释即写明 "non-authoritative"）：
  真正的判定仍以 DirectUI 布局校验为准，因此即使某个中文标题没匹配上，也只会退化为原版行为，不会误判。
- 自绘文字的本地化只区分「简体中文系统」与「其他」，暂未支持繁体中文。
- Mod 只改变**外观**：复制、移动、删除、冲突与错误处理仍由 Windows 负责
  （这是原版的设计，本分支未改动）。

## 许可与致谢

- 原版作者：**[digART](https://github.com/digart11)** —— 本分支的全部核心实现均来自上游
  [File-Operation-Styler](https://github.com/digart11/File-Operation-Styler)。
- 许可：**GPL-3.0**（见 [LICENSE](LICENSE)），本分支同样以 GPL-3.0 发布。
- 本分支（汉化 + 兼容修复）维护者：**[Z2549](https://github.com/Z2549)**。
