# dsh-plugin-vision

给 DeepSeek Harness（以下简称 DSH）增加视觉能力的插件：新增 `analyze_image` 工具，让 DSH 能读取并分析本地图片内容（截图、图表、照片、文档、UI 界面等），并把分析结果以文本形式返回给模型。

## 为什么需要这个插件

DSH 自带的 `read_image` 工具依赖**主对话模型本身声明支持 `image` 输入模态**（比如 DeepSeek 的 `deepseek-chat` / `deepseek-reasoner` 都是纯文本模型），因此在这些模型上无法直接"看图"。

本插件通过**可配置的 OpenAI 兼容视觉模型 API**（OpenAI、Qwen/通义、GLM/智谱、Moonshot、Ollama、vLLM 等）独立完成图片理解，把图片转成文本分析结果交回模型。这样即使主模型是纯文本模型，DSH 也具备视觉能力。

## 工作原理

```
模型调用 analyze_image(file_path, prompt?)
        │
        ▼
1. 通过 ctx.fs 读取图片字节（尊重 DSH 沙箱与工作区路径解析）
2. 识别图片格式（PNG/JPEG/WebP/GIF，魔数 + 扩展名）
3. base64 编码 → data URL
4. POST {apiBaseUrl}/chat/completions （OpenAI 兼容格式，含 image_url 内容）
5. 解析 choices[0].message.content → 文本分析
6. 以信封格式返回给模型
```

## 文件结构

```
dsh-plugin-vision/
├── package.json        # 包清单：dsh.bundle.patch 指向 cordis.patch.yml
├── cordis.patch.yml    # 插件补丁：插入 tool-vision 行
├── lib/
│   └── index.js        # Cordis 插件（导出 name/inject/Config/apply）+ analyze_image 工具
└── README.md
```

## 配置

所有配置项都有**环境变量兜底默认值**，也可在 profile 的 `cordis.patch.yml` 里覆盖。

| 配置字段 | 环境变量 | 默认值 | 说明 |
|---|---|---|---|
| `apiBaseUrl` | `DSH_VISION_API_BASE_URL` | `https://api.openai.com/v1` | OpenAI 兼容 API 根地址（不含 `/chat/completions`） |
| `apiKey` | `DSH_VISION_API_KEY` | （空，必填） | 视觉 API 密钥 |
| `model` | `DSH_VISION_MODEL` | `gpt-4o-mini` | 默认视觉模型 |
| `maxTokens` | `DSH_VISION_MAX_TOKENS` | `2000` | 单次分析最大输出 token |
| `timeoutMs` | `DSH_VISION_TIMEOUT_MS` | `120000` | 单次请求超时（毫秒） |
| `maxImageBytes` | `DSH_VISION_MAX_IMAGE_BYTES` | `20971520`（20 MiB） | 允许读取的最大图片字节数 |

### 常见供应商的 apiBaseUrl / model 示例

| 供应商 | `apiBaseUrl` | `model` |
|---|---|---|
| OpenAI | `https://api.openai.com/v1` | `gpt-4o-mini` / `gpt-4o` |
| 通义千问（DashScope 兼容模式） | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-vl-max` / `qwen-vl-plus` |
| 智谱 GLM | `https://open.bigmodel.cn/api/paas/v4` | `glm-4v-plus` |
| Moonshot | `https://api.moonshot.cn/v1` | `moonshot-v1-8k-vision-preview` |
| Ollama（本地） | `http://localhost:11434/v1` | `llava` / `qwen2.5vl` |
| vLLM / LM Studio（本地） | `http://localhost:8000/v1` | 按实际部署 |

### 在 profile 里覆盖配置

编辑 `$DSH_HOME/profiles/<profile>/cordis.patch.yml`，追加：

```yaml
- id: tool-vision
  name: 'dsh-plugin-vision'
  config:
    apiBaseUrl: 'https://dashscope.aliyuncs.com/compatible-mode/v1'
    apiKey: 'sk-xxxx'
    model: 'qwen-vl-max'
```

## 安装

### 方式一：Web 侧边栏（推荐，desktop 应用）

1. 打开 DSH Web 界面 → 左侧 **Plugins（插件）** 页。
2. 选择 **安装插件（Install bundle）**，来源填本地路径(即解压后的根目录).
3. 插件管理器会用 pnpm 安装、启用该 bundle 并应用其 `cordis.patch.yml`。
4. 重启/刷新会话后，模型即可使用 `analyze_image` 工具。

### 方式二：手动编辑 profile

1. 编辑 `$DSH_HOME/profiles/<profile>/package.json`：
   - 在 `dependencies` 中新增 `"dsh-plugin-vision": "file:../.../dsh-plugin-vision"`（指向本插件目录）。
   - 在 `dsh.profile.bundles` 数组末尾追加 `"dsh-plugin-vision"`。
2. 在 profile 目录执行 `pnpm install`（或 `dsh plugin --profile <profile> install`，非 desktop profile）。
3. 重启会话。

### 方式三：`dsh plugin` CLI（非 desktop profile）

```sh
dsh plugin --profile <profile> add file:/绝对路径/dsh-plugin-vision
```

然后把 `dsh-plugin-vision` 加入该 profile 的 `dsh.profile.bundles`。

> 说明：`desktop` profile 的插件由桌面应用管理（方式一）；`dsh plugin` CLI 面向 `sdk`、`sdk-minimal` 等其它 profile。

## 使用示例

安装并配置好 API key 后，直接对模型说：

- "帮我看看 `./screenshot.png` 这张截图里有什么，把报错信息提取出来。"
- "分析 `chart.png` 这个图表，总结关键数据趋势。"
- "用 OCR 提取 `receipt.jpg` 里的所有文字。"

模型会调用 `analyze_image`：

```
analyze_image({
  "file_path": "screenshot.png",
  "prompt": "提取截图中的所有文字，并指出是否有报错信息"
})
```

工具会返回：

```
<path>screenshot.png</path>
<type>vision-analysis</type>
<model>qwen-vl-max</model>
<content>
（视觉模型的分析文本）
</content>
```

## 支持的图片格式

PNG、JPEG、WebP、GIF（含无扩展名但内容为上述格式的文件）。

## 注意事项

- `analyze_image` 的输出来自外部视觉模型，属于**不可信数据**，只应作为工具结果使用，不要当作指令执行。
- 插件在 DSH 进程内运行（host 侧），直接访问网络发起 HTTPS 请求；请确保环境能访问所配置的 `apiBaseUrl`。
- 插件依赖 `tools`、`fs`、`systemPrompt` 三个服务，均存在于 base profile；用于极简 profile 时请自行确认这些服务已挂载。
