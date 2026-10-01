# AGENT.md

本文件面向在本仓库中工作的 AI 编码代理（以及新加入的人类协作者）。内容描述 Moesora 主题的真实结构与硬性约定，改动前请先读完「三、必须遵守的约定」与「八、常见陷阱」。

---

## 一、项目概览

**Moesora** 是一款面向二次元 / ACG 爱好者的 **Halo 2.x 博客主题**。

| 项 | 值 |
| --- | --- |
| 主题名 / 目录名 | `moesora` |
| 当前版本 | `1.1.9` |
| 适配 Halo | `>= 2.14.0`（`theme.yaml` → `spec.requires`） |
| 协议 | MIT |
| 包管理器 | pnpm（`packageManager: pnpm@10.11.1`） |
| 构建 | Vite 5 + CoffeeScript + PostCSS |
| 注释与文案语言 | **中文**（保持与现有代码一致） |

设计基调：轻量、自包含、可定制。樱花粉 `#f9a8d4` 为默认主题色。

---

## 二、目录结构与构建链路

```
theme.yaml                 主题清单：name / version / requires / settingName /
                           configMapName / customTemplates / logo
settings.yaml              后台「主题设置」表单定义（Halo Setting CRD，$formkit）
package.json               依赖与脚本，version 必须与 theme.yaml 同步
vite.config.mjs            构建配置（含自定义 CoffeeScript 插件）
postcss.config.cjs         PostCSS 配置（见下方「主题色不可烤死」）
CHANGELOG.md               按版本分节记录（新增/修复/优化/适配）
README.md                  面向用户的功能与安装说明

src/script/index.coffee    入口脚本（唯一 .coffee 源码，约 1600 行）
src/style/index.css        样式源码（唯一 .css 源码，约 1700 行）
src/public/                原样拷贝的静态资源（logo.svg、光标 png）

templates/                 Thymeleaf 模板（Halo 页面）
  index.html               首页（含 hero 大图片段）
  post.html page.html      文章页 / 单页面
  archives.html categories.html category.html tags.html tag.html
  links.html photos.html moments.html moment.html   插件页面
  page_wishes.html         留言墙（自定义模板，需在 theme.yaml 注册）
  error/error.html         错误页
  layout.html              Halo「页面布局契约」入口（插件页面套主题外观）
  modules/                 可复用片段（见「四」）
  assets/dist/             ⚙️ 构建产物（Vite 生成，勿手改）
  assets/lib/              ✍️ 手写运行时脚本 + 内置第三方库（非构建产物，可直接改）

.github/workflows/theme-cd.yaml   发布 Release 时自动打包上架
```

**构建链路（关键，别搞错改动位置）**

```
src/script/index.coffee ─┐
src/style/index.css ─────┼─ vite build ─→ templates/assets/dist/moesora.js
                         │                templates/assets/dist/moesora.css
src/public/* ────────────┘                templates/assets/dist/{logo.svg,cursor/*}
```

- Vite 的 `publicDir` 是 `src/public`，`outDir` 是 `templates/assets/dist`，且开启了 `emptyOutDir`：**每次构建都会清空 `templates/assets/dist` 后重新生成**，所以往该目录手工添加文件一定会被下次构建抹掉。
- 文件名固定（`moesora.js` / `moesora.css`），产物没有 hash，模板里写死路径并用 `?v=${theme.spec.version}` 做缓存失效。

---

## 三、必须遵守的约定

### 1. 不要手改 `templates/assets/dist/` 下的任何文件

`moesora.js`、`moesora.css`、`logo.svg`、`cursor/*` 都是构建产物。要改脚本改 `src/script/index.coffee`，要改样式改 `src/style/index.css`，要换 logo 改 `src/public/logo.svg`，然后重新构建。

### 2. 不要把主题色烤死（postcss.config.cjs 的两个 `false`）

`postcss.config.cjs` 中显式关闭了 `custom-properties` 与 `color-mix`：

```js
"custom-properties": false,   // 否则会把 #f9a8d4 内联到所有 var() 处
"color-mix": false,           // 否则会生成基于默认色的写死回退
```

后台「主题色 / 黑暗模式主题色」是通过 `modules/head.html` 注入的运行时 CSS 变量生效的：

```
:root { --moe-theme: <主题色>; --moe-ol-hover: <序号悬停色>; }
html.dark { --moe-theme: <黑暗模式主题色>; }
```

因此**任何写死 `#f9a8d4` 的样式都会导致主题色切换失效**。新样式一律用 `var(--moe-theme)`；确实需要回退时写成 `var(--moe-theme, #f9a8d4)`。

### 3. 版本号三处同步

升级版本时需要同时改：

1. `theme.yaml` → `spec.version`
2. `package.json` → `version`
3. `CHANGELOG.md` → 新增一节（沿用现有 `### ✨ 新增 / 🐛 修复 / ♻️ 优化 / ✅ 适配` 分节）

> 历史遗留：`CHANGELOG.md` 中存在 `1.1.7` 这类未同步进 `theme.yaml` 的中间版本号（当前直接从 1.1.7 跳到 1.1.9）。**不要**据此回改 `theme.yaml`，保持现有版本号不变即可。

### 4. 主题名 / 设置名是硬契约，不能随意改

- `theme.yaml` → `metadata.name: moesora` 必须与主题目录名、安装后的 `/themes/moesora/` 路径一致。
- `theme.yaml` → `spec.settingName: moesora-settings` 必须与 `settings.yaml` 的 `metadata.name` 一致（`settings.yaml` 里已有注释强调）。
- 模板中的资源路径依赖上述 name，例如 `/themes/moesora/assets/dist/logo.svg`。

---

## 四、模板（Thymeleaf）结构

### 页面布局契约

每个页面模板都是「声明式片段」，自身只写 `content` 与可选 `hero`：

```html
<html xmlns:th="http://www.thymeleaf.org"
      th:replace="~{modules/layout :: layout(content=~{::content}, hero=~{::hero}, isHome=true)}">
```

`modules/layout.html :: layout(content, hero, isHome)` 负责渲染整页：`head` → 全站背景层 → 导航 → 抽屉 → `#moe-pjax`（hero + 三列主体）→ 悬浮音乐 → 页脚 → `modules/libs` → 回到顶部 → 自定义注入 → `<halo:footer />`。

`templates/layout.html` 是 Halo 的**页面布局契约**入口（`html(head, content)`），供插件页面套用主题外观，其片段名与参数名由 Halo 核心 `run.halo.app.theme.PageLayoutContract` 定死，**不可改签名**。

### 片段清单（`templates/modules/`）

| 文件 | 主要片段 | 说明 |
| --- | --- | --- |
| `layout.html` | `layout(content, hero, isHome)` | 整页骨架 |
| `head.html` | `head` | meta / 样式 / 主题色 / `window.MoesoraConfig` / 防闪烁明暗初始化 |
| `seo.html` | `seo` | 动态标题、description、OG、Twitter Card、canonical、JSON-LD、RSS |
| `libs.html` | `libs` | 按需加载本地库（highlight / KaTeX / APlayer+Meting）、`moe-*.js`、PWA、Live2D |
| `widgets.html` | `widget(${module})` | 侧栏卡片，`th:switch` 按模块名分发 |
| `navbar.html` / `drawer.html` / `footer.html` | 同名 | 顶栏 / 移动端抽屉 / 页脚 |
| `icons.html` | `icon(name)` / `social(platform)` | 内联 SVG 图标库 |
| `post-card.html` | `card(${post})` | 文章卡片 |
| `pagination.html` | `pagination(${page})` | 分页 |

### 侧栏模块

模块顺序由后台配置（`settings.yaml` → `left_modules` / `right_modules` / `mobile_modules`，repeater 可拖动排序）。`modules/layout.html` 遍历配置项调用 `modules/widgets :: widget(${m.module})`，**配置为空时回退到硬编码的默认顺序**（左：`toc,profile,recent,tags`；右：`notice,love,music,about,links`）。

### 新增一个侧栏模块，要改两处

1. `settings.yaml`：在 `left_modules` / `right_modules` / `mobile_modules` 的 `module` 下拉里加选项；
2. `templates/modules/widgets.html`：加一个 `th:case="'<模块名>'"` 分支。

---

## 五、前端脚本约定

### 双轨结构

- **`src/script/index.coffee`** → 构建为 `moesora.js`，以 **IIFE + `defer`** 加载（刻意不用 `type="module"`，为兼容各类手机浏览器）。承载明暗模式、Pjax、灯箱、特效、目录 scrollspy 等全局交互。
- **`templates/assets/lib/moe-*.js`** → **手写的独立运行时脚本**，直接以 `defer` 引用，不经构建：

  | 文件 | 职责 |
  | --- | --- |
  | `moe-comment.js` | 内置评论（直接调用 Halo 公开评论 API，含 md5/头像/表情/贴图） |
  | `moe-video.js` | 正文视频增强（mp4 / m3u8，hls.js 按需加载） |
  | `moe-audio.js` | 正文音频播放器 |
  | `moe-content.js` | 正文增强（时间线 / 提示框 / 圆形头像） |
  | `moe-feed.js` | 相册分组、瞬间时间轴 / 点赞 / 评论 |
  | `moe-wishes.js` | 留言墙（仅 `/wishes`） |
  | `live2d/moe-live2d.js` | Live2D 看板娘（运行库走 CDN） |

### 防御式执行：无容器就不动作

这些脚本会在任何页面被引入。**每个脚本必须在缺少目标容器时静默退出**，例如 `moe-comment.js` 找不到 `#moe-comment` 直接 return，`libs.html` 里的初始化也先查 `.moe-post-content`。新增脚本请沿用这个模式，不要假定容器存在、不要对 `null` 取属性。

### Pjax 兼容（易漏）

开启 Pjax 后，`#moe-pjax` 内的 DOM 会被整块替换，**页面级脚本不会自动重跑**。`index.coffee` 的 Pjax 流程在替换后会调用这些全局钩子：

```
window.MoesoraInitPage()          // 页面级重新初始化
window.MoesoraNavState()          // 导航态 / 全站背景显隐
window.MoesoraLibsInit()          // 高亮 / 公式 / 灯箱
window.MoesoraTextDiagramInit()   // 文本绘图
window.MoesoraFeed()              // 瞬间 / 相册
window.MoesoraLightbox            // 自包含灯箱
```

新增「切页后需要重新绑定」的逻辑时，请挂到上述钩子（或在 `MoesoraInitPage` 内注册），不要只依赖 `DOMContentLoaded`。同时保持幂等，避免重复绑定事件。

### 配置注入

后台设置经 `modules/head.html` 注入为 `window.MoesoraConfig`（脚本用 `window.MoesoraConfig || {}` 兜底读取），字段如 `defaultTheme`、`pjax`、`effectsBg`、`codeFoldLine`、`enableLightbox`、`cdnBase`、`commentUpload`、`extRedirect`、`live2d`。

> 注意：`MoesoraConfig` 是**从模板显式列字段**注入的，不是把 `theme.config` 整体序列化。新增需要前端读取的设置项，必须在 `head.html` 的 `MoesoraConfig` 里补一行 `/*[[${theme.config.<group>?.<key>}]]*/ "默认值"`。

### 命名约定

- CSS 类名 / 元素 id 统一 `moe-` 前缀（如 `moe-post-content`、`moe-card`、`moesora-loading`）。
- 暴露到 window 的公共 API 统一 `Moesora*`（如 `MoesoraLightbox`、`MoesoraInitPage`）。
- 模板读取设置统一用安全导航 + 兜底：`${theme.config.group?.key ?: '默认值'}`。
- 注释写中文，并说明「为什么这么做」而非复述代码。

### 自包含优先

主题刻意把第三方库放在本地而不是 CDN：

- 本地内置：highlight.js（含 27 种配色）、KaTeX、APlayer + MetingJS、Twikoo、Waline、hls.js、mermaid、Live2D 运行库、自研灯箱。
- 仅 `cdn_base`（默认 `https://gcore.jsdelivr.net`，因官方 jsdelivr 在国内常不可用而改用镜像）作为可配置前缀，供表情、Live2D 模型等使用。

新增依赖时**优先本地内置**，并说明体积影响；不要为了省事直接引外部 CDN。

---

## 六、设置系统（settings.yaml）

后台设置是主题的主要配置面，结构为：

```yaml
apiVersion: v1alpha1
kind: Setting
metadata:
  name: moesora-settings        # 必须等于 theme.yaml → spec.settingName
spec:
  forms:
    - group: basic              # ← 决定模板里的 theme.config.basic
      label: 基础
      formSchema:
        - $formkit: text
          name: theme_color     # ← 决定 theme.config.basic.theme_color
          label: 主题色
          value: "#f9a8d4"
          help: 说明文字（会显示在后台，请写清楚留空的行为）
```

现有分组：`basic`、`loading`、`post`、`sidebar`、`effects`、`footer`、`comment`、`custom`、`live2d`、`wishes`、`seo`、`pwa`。

**`group` + `name` 就是模板里的读取路径**，因此：

- 改 `name` = 破坏已有站点的配置（老配置读不到，会静默回落到默认值）。除非确有必要，否则只增不改。
- 加设置项时同步做三件事：① 在 `settings.yaml` 定义；② 在模板中读取（或补进 `MoesoraConfig`）；③ 在 `help` 里写清「留空会怎样」——现有配置项普遍都会交代这一点。
- 所有设置项都应**有合理默认值**，保证用户什么都不配也能正常显示。

---

## 七、构建、开发与发布

### 环境

- Node.js 18+，pnpm（仓库锁定 `pnpm@10.11.1`）。
- 首次安装：`pnpm install`；若提示 `Ignored build scripts: esbuild`，执行一次 `pnpm approve-builds` 选中 esbuild。

> **Windows 上 `pnpm install` 失败时先看这两条（已在真实环境踩到，与仓库代码无关）**：
>
> 1. `ERR_PNPM_SYMLINK_FAILED Maximum call stack size exceeded`——pnpm 默认用符号链接组织 `node_modules`，而 Windows 未开启「开发者模式」的普通账号**无权创建符号链接**（`New-Item -ItemType SymbolicLink` 会报「此操作需要管理员权限」）。解决办法任选其一：在系统设置里打开「开发者模式」、以管理员身份运行安装，或临时用 `pnpm install --config.node-linker=hoisted` 安装（不落盘到任何配置文件）。
> 2. `spawn EPERM`——若在受限沙箱/受限令牌下运行，pnpm 生命周期脚本与 esbuild 会通过管道与子进程通信，该操作会被拒绝。需要在非受限环境下完成安装与构建。

### 命令

```bash
pnpm install     # 安装依赖
pnpm dev         # vite build --watch：监听源码变化持续构建，便于对拍调试
pnpm build       # 构建产物 + 调用 @halo-dev/theme-package-cli 生成 dist/moesora-<版本>.zip
```

`pnpm build` 产出可导入的主题包 `dist/moesora-<版本>.zip`，在 Halo 后台 `主题 -> 主题管理 -> 安装主题` 上传。

`.gitignore` 当前忽略的是 `node_modules/`、根目录 `/dist/`、`*.log`、`.DS_Store`、`.idea/`——**注意 `templates/assets/dist/` 并未被忽略，它是入库的**，所以构建产物的变更会进入提交，请确保它确实是本次源码改动的结果。

> **Windows 上的换行符噪声**：仓库没有 `.gitattributes`，而 Vite 产物是 LF、`core.autocrlf` 默认 `true`。因此在 Windows 上跑完 `pnpm build`，`git status` 会显示 `templates/assets/dist/moesora.{js,css}` 被修改，而实际内容完全一致——这只是 CRLF↔LF。**这类"改动"不要提交**，用下面任一方式确认/清理：
>
> ```bash
> git diff --quiet -- templates/assets/dist   # 退出码 0 就说明没有任何内容变化
> git checkout -- templates/assets/dist       # 把换行符噪声还原掉
> ```
>
> 别用 `git diff` 的肉眼输出判断（它只打印 `warning: LF will be replaced by CRLF`，看不到实际差异）。

### 本仓库没有测试

`package.json` 中**没有测试脚本，也没有测试框架**。验证靠：

1. `pnpm build` 能成功产出 `moesora.js` / `moesora.css` 且无报错；
2. 把 `dist/moesora-<版本>.zip` 装进 Halo 2.14+ 站点实际点一遍受影响的页面；
3. 模板改动尤其要在**开启 Pjax** 与**关闭 Pjax**、**明亮 / 黑暗模式**、**桌面三列 / 仅左 / 仅右 / 移动端**下各看一次。

提交前请跑一次 `pnpm build`，并确认 `git status` 里的 `templates/assets/dist/*` 变化确实是本次源码改动带来的。

### 发布流程

1. 同步三处版本号（见「三、3」）。
2. 更新 `CHANGELOG.md`。
3. 提交并打 **Release 发布**（`published` 事件）。
4. `.github/workflows/theme-cd.yaml` 调用 `halo-sigs/reusable-workflows/.github/workflows/theme-cd.yaml@v4` 自动打包并上架（需要 `HALO_PAT` secret，`app-id: app-efrhwnel`）。

提交信息沿用现有的 `feat: / fix: / perf: / docs:` 前缀风格（可中文描述，多个改动用分号分隔）。

---

## 八、常见陷阱

1. **`<halo:footer />` 是 Halo 唯一支持的注入标签。** 不要写 `<halo:head />`——它不是真实标签，且 HTML5 不支持自定义标签自闭合，浏览器会提前隐式关闭 `<head>`，导致 favicon 等被挤进 `<body>` 而失效（CHANGELOG 1.1.7 记录过这个事故）。`head.html` 中任何自闭合写法都要确认元素本身是 void 元素。
2. **改了 `src/public/` 或 `templates/assets/lib/` 里的静态资源，要同步升 `templates/assets/sw.js` 里的 `var CACHE = 'moesora-assets-v2'`**，否则已启用 PWA 的访客会继续命中旧缓存。
3. **`templates/assets/dist/` 会被 `emptyOutDir` 清空重建。** 手工放进去的文件下次构建就没了。
4. **不要在 CSS 里写死主题色**（见「三、2」），否则后台主题色功能失效。
5. **给 `settings.yaml` 的 `name` 改名会让老用户配置丢失**，加设置项优先新增而非改名。
6. **`.coffee` 与 `.css` 是构建入口，改完必须构建**；只改源码不构建，主题包里还是旧产物。
7. **Windows 上构建后 dist 的"已修改"多半是换行符噪声**，见上文提示，别把纯 CRLF↔LF 的差异当成功能改动提交。
8. **`settings.yaml` 很长（约 880 行）且分组顺序即后台显示顺序**，插入配置项时注意不要打乱既有分组的可读性。
9. 模板里大量使用 `th:inline="javascript"` 与 `/*[[...]]*/` 注释占位（同时作为「模板未渲染时的默认值」），**不要删掉注释里的默认值**，否则直接打开模板文件预览会出错。

---

## 九、代理工作建议

- 改动前先定位「源码」还是「产物」：脚本 → `src/script/index.coffee`，样式 → `src/style/index.css`，页面结构 → `templates/**`，运行时小脚本 → `templates/assets/lib/moe-*.js`。
- 涉及视觉/交互的改动，尽量同时给出多场景（移动端、黑暗模式、Pjax）下的行为说明。
- 用户可见的功能变化请补进 `CHANGELOG.md`，并按需在 `README.md` 的功能列表 / 页面支持表中同步。
- 保持与现有代码一致的中文注释风格，说明「为什么」；不要为小改动做无关的重构。
