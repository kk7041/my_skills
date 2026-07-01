# my_skills

存放我自己沉淀的 Agent Skills，方便在不同 Hermes/Haviland 环境中复用。

## Skills

### `design-system-theming`

构建可复用、多主题的设计系统。

- 位置：`skills/design-system-theming/`
- 适用：设计系统、多主题 CSS、主题 token、前端 UI 规范沉淀

### `aicodewith-image-generation`

使用 AICodeWith API 进行文生图 / 图生图 / 图片编辑，并沉淀了 Haviland 当前的生图工作流。

- 位置：`skills/aicodewith-image-generation/`
- 适用：生图、生成图片、画图、做图、出图、海报、logo、icon、illustration、poster、product shot、参考图编辑等
- 核心能力：
  - 使用 `https://api.aicodewith.com` 创建与查询图片生成任务
  - 支持 `gpt-image-2` / `gpt-image-2-beta`
  - 生成前强制确认模型、比例、数量、质量、分辨率
  - 支持参考图分析、图生图、任务轮询、失败处理和重试策略
  - 记录 Telegram 图片回传、IP/Logo 吉祥物、网站横幅等常见坑位

## 安全说明

仓库只保存 Skill 文档和可复用流程，不保存真实密钥。

- AICodeWith API Key 应放在本地环境变量：`AICODEWITH_API_KEY`
- 不要提交 `.env`、`.env.*`、私钥、证书或 `secrets/` 目录
- `.gitignore` 已覆盖常见敏感文件和本地缓存

## 使用方式

把需要的 skill 目录复制到 Hermes profile 的 skills 目录下，例如：

```bash
~/.hermes/skills/devops/aicodewith-image-generation/
```

然后在对话中触发生图相关任务时，Haviland 会加载并按该 Skill 的流程执行。
