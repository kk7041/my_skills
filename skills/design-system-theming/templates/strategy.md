# 设计系统复用策略

> 状态：可复用方法论 + 主题契约模板。
> 适用范围：本仓库内多套视觉风格的复用与扩展。
> 与 `docs/dashboard-design-system.md`（Dashboard 具体规范）、`AGENTS.md`（Landing 规范）配套使用。

## 目的

本仓库已经同时承载多套视觉风格：青绿 SaaS 默认主题、Landing 的 brutalist/terminal 风格、Dashboard 的 warm SaaS 控制台风格。它们共用一套 shadcn 组件和工具类，靠作用域 class 换肤。

本文把这套做法沉淀成可复用策略：**复用结构与契约，把"风格"隔离成唯一可替换的变量层**。新风格不是重写一套外观，而是给同一份契约填一组新值。

## 核心原则

> 稳定契约 + 可替换皮肤。

不同站点/页面的风格差异，全部收敛到"原始调色板 + 一组主题旋钮"。组件、语义 token 名、工具类、密度规范保持不变。

## 四层模型

从下到上越来越稳定。复用的是上三层，每套风格真正要改的只有最底层。

| 层 | 内容 | 复用程度 | 代码位置 |
| --- | --- | --- | --- |
| 规范文档层 | 密度 / 排版 / 间距纪律 | 90% 继承，10% 每风格微调 | `docs/dashboard-design-system.md`、本文件 |
| 组件层 | `@/components/ui/*`（shadcn） | 近 100% 复用，不绑死颜色 | `src/components/ui/` |
| 语义 token 层 | `--primary` / `--background` / `--radius` 等契约 | 名字不变，是复用契约本身 | `globals.css` 的 `@theme inline` |
| 原始调色板层 | 具体 hex/oklch、字体、圆角值 | 每风格 100% 替换 | `globals.css` 各作用域 |

## Token 三层模型

当前实现是"作用域里直接写死语义值"。要让风格更易拆分和对比，推荐升级为三段式命名：

```css
/* 1. primitive：原始调色板，每个主题各自定义，只在本主题内出现 */
--c-brand-500: #D8A060;
--c-neutral-900: #333333;

/* 2. semantic：语义角色，名字跨主题固定，这就是复用契约 */
--primary: var(--c-brand-500);
--foreground: var(--c-neutral-900);

/* 3. component：shadcn 工具类只消费 semantic，永不直接碰 primitive */
@theme inline {
  --color-primary: var(--primary);
  --color-foreground: var(--foreground);
}
```

接入新主题时只重写 primitive 和 primitive→semantic 映射，组件层和工具类一行都不改。

### 命名约定

- primitive：`--c-{色系}-{档位}`，如 `--c-brand-500`、`--c-neutral-100`。纯调色板事实，不带语义。
- semantic：沿用 shadcn 既有名（`--background`、`--foreground`、`--primary`、`--muted`、`--accent`、`--border`、`--ring`、`--radius`、`--sidebar-*`、`--chart-*`）。不要发明新语义名，除非确实新增了一类语义角色。
- 品牌原色（供高度视觉化组件直接引用）：`--brand-{name}`，如 `--brand-cream`、`--brand-tan`。当前 Dashboard 已用此约定。

## 主题契约（Theme Contract）

任何一套风格 = 下表填一遍。风格再不同，也只是同一契约的不同取值。

| 旋钮 | 含义 | 取值示例 |
| --- | --- | --- |
| 调色板 | primitive 色板 | warm cream / cool slate / high-contrast mono |
| `--background` / `--foreground` | 底色 / 前景 | 见下方已实现示例 |
| `--primary` / `--accent` | 主色 / 强调色 | teal / tan / charcoal / orange |
| `--radius` | 圆角基准 | `0rem`(硬边) / `0.5rem` / `0.75rem`(friendly) |
| `--font-sans` / `--font-mono` / `--font-heading` | 字体栈 | IBM Plex / Nunito / Inter / 像素体 |
| 阴影强度 | 自定义工具类 | none / `shadow-xs` / 多层 `card-shadow` / `glow` |
| 密度 | 控件高度基线 | compact(`h-8`) / comfortable(`h-10`) |
| 纹理 / 动效 | 装饰工具类 | `dot-grid-bg` / `noise-bg` / `animate-glitch` / 无 |

### 契约模板（复制后填值）

```css
.theme-<name> {
  /* primitive */
  --c-...: ...;

  /* semantic mapping */
  --background: ...;
  --foreground: ...;
  --primary: ...;
  --primary-foreground: ...;
  --secondary: ...;
  --secondary-foreground: ...;
  --muted: ...;
  --muted-foreground: ...;
  --accent: ...;
  --accent-foreground: ...;
  --destructive: ...;
  --destructive-foreground: ...;
  --border: ...;
  --input: ...;
  --ring: ...;
  --radius: ...;

  /* sidebar（如使用 dashboard 布局）*/
  --sidebar: ...;
  --sidebar-foreground: ...;
  --sidebar-primary: ...;
  --sidebar-accent: ...;
  --sidebar-border: ...;
  --sidebar-ring: ...;

  /* charts */
  --chart-1: ...; --chart-2: ...; --chart-3: ...; --chart-4: ...; --chart-5: ...;

  /* fonts */
  --font-sans: ...;
  --font-mono: ...;

  color: var(--foreground);
  background-color: var(--background);
}
```

### 已实现示例（同一契约的三组取值）

| 旋钮 | `:root`（默认 SaaS） | `.mergio-landing`（brutalist） | `.mergio-dashboard`（warm 控制台） |
| --- | --- | --- | --- |
| `--background` | `oklch(0.98 0.002 240)` | `oklch(0.92 0.018 86)` cream | `#FFFFFF` |
| `--foreground` | `oklch(0.12 0.01 240)` | `#333333` | `#333333` |
| `--primary` | `oklch(0.55 0.2 170)` teal | `#333333` charcoal | `#D8A060` tan |
| `--accent` | `oklch(0.7 0.15 150)` | `oklch(0.6 0.19 40)` orange | `#F8F0E0` beige |
| `--radius` | `0.75rem` | `0rem` | `0.5rem` |
| `--font-sans` | Nunito | IBM Plex Sans | Nunito |
| 纹理/动效 | 无 | `dot-grid-bg`、`animate-glitch`、`animate-marquee` | `dot-grid-bg` |

## 新增一个主题的步骤

1. 在 `globals.css` 新增 `.theme-<name>`（或沿用 `.mergio-<name>` 命名），按契约模板填值。
2. 只填 primitive 与 semantic 映射，不要新增 `@theme inline`（它在 `:root` 已映射好，对所有作用域生效）。
3. 如风格需要字体，在 `src/app/layout.tsx` 用 `next/font` 注入 `--font-*` 变量，再在主题里引用。
4. 如需专属纹理/动效，在 `@layer utilities` 加作用域化工具类（参考 `.mergio-landing .dot-grid-bg`）。
5. 在最外层容器挂上主题 class：`<div className="theme-<name> bg-background ...">`。内部所有 `ui/*` 组件与 token 工具类自动跟随。
6. 在本文件"已实现示例"表追加一列，记录该主题的旋钮取值。

## 组件层复用规则

- 组件只用语义 token 工具类（`bg-background`、`text-primary`、`border-border`、`rounded-lg`），不写死颜色 hex/oklch。这样同一组件在任何主题作用域里自动换肤。
- 业务页面优先复用 `@/components/ui/*`，不在页面层临时发明视觉体系。
- 高度视觉化组件（canvas、glow、像素特效）可局部用品牌原色 `--brand-*` 或主题色常量，但应集中、与当前主题色板一致。
- 图标统一 `lucide-react`，`size-4`。

## 密度与排版规范（基线 + 可覆盖）

基线规范继承 `docs/dashboard-design-system.md`，每套风格只在必要处覆盖：

- 基线（建议跨主题继承）：4px/8px 间距网格（`gap-1/2/4/6`、`p-4/6`）；控件高度 `h-8/9/10`；图标 `size-4`；UI 文本 `text-sm leading-5`、辅助 `text-xs leading-4`。
- 可每主题覆盖：圆角（`--radius`）、阴影强度、标题字体与字重、字母间距（如 brutalist 用 mono + 紧间距，CJK 需 `letter-spacing: 0`）、纹理密度。
- 覆盖时写成作用域化规则（`.theme-<name> ...`），不要污染全局 `body` 或其他主题。

## 反模式（不要做）

- 不要在组件或页面里写死颜色值绕过 token。
- 不要为换主题去复制整套组件，主题差异只应存在于变量层。
- 不要在一个主题里重定义 `@theme inline` 映射，映射是全局契约。
- 不要局部覆盖 `--font-*` 到非主题指定字体，除非明确要求。
- 不要卡片套卡片、不要随意 spacing（见 Dashboard 规范）。

## 接入检查清单

- [ ] 新主题只改了 primitive + semantic 映射，没碰组件层。
- [ ] 所有 semantic token 名沿用 shadcn 契约，未发明新名。
- [ ] 组件全部用 token 工具类，无写死颜色。
- [ ] 字体通过 `--font-*` 变量注入，未在组件内覆盖。
- [ ] 专属纹理/动效已作用域化（`.theme-<name> ...`）。
- [ ] 密度/排版沿用基线，覆盖项均作用域化。
- [ ] 已在"已实现示例"表登记该主题旋钮取值。
