# OlivOS · NovelAI 画图插件

> **这是一个 OlivOS 插件**（QQ 机器人 / OneBot 11）。把 `NovelAIPainterV9` 目录丢进 OlivOS 的 `plugin/app/`，重载插件即可在群里用「画图」「智能生图」「创意生图」等指令出图。
>
> 当前版本：**v10.6.0**（2026-11）

OlivOS 0.11.81 + OlivaDice 3.4.52 环境实测可用；纯 Python 标准库实现，**没有任何第三方依赖**（只需要 OlivOS 自带的运行环境）。

---

## 这是什么

一个把 **NovelAI 出图** 搬进 QQ 群的骰娘/机器人插件：

- **生图通道**：`novelai.me` 积分站「开放接口」token 直连（`Authorization: Bearer <uai_ token>`），同时兼容 `sdapi` 兼容层（`/sdapi/v1/txt2img`，HTTP Basic `api:<token>`）。
- **纯文生图**：不再依赖参考图；sdapi 兼容层不解析角色框，因此多角色场景会把角色框合并成单条 prompt。
- **提示词由 LLM 写**：`LLM_PROVIDERS` 是按 `priority` 排序的多渠道列表，连续失败达阈值才判定该渠道不可用并降级到下一条（认证类 401/403 立即降级）。
- **v10.6 写词语料**：从 391 张真实 V5 成图（韩网作品集，本地二进制反读）提炼的写法灌进系统提示词——画师串四层结构 + `draw body like artist:X` 人体定向借用、负强调配置表、`source#/target#/mutual#` 互动归属语法、角色框「一个画面元素一框」等。
- **额度与权限**：分级闸门（0 健全 / 1 轻度 / 2 诱惑 / 3 硬核 + 隐藏档 4 重口）、白名单、周额度硬闸门（周一 0 点重置；骰主不限）。
- **零停机换 key**：`热更 nai|web|ds <key>`、`热更 ds.<渠道名> <key>`，落盘到数据目录，不重启插件。

---

## 安装

```text
OlivOS/
└── plugin/
    └── app/
        └── NovelAIPainterV9/      ← 把本仓库内容放到这里
            ├── app.json
            ├── main.py
            └── __init__.py
```

1. 克隆或下载本仓库，把**目录内容**放进 `OlivOS/plugin/app/NovelAIPainterV9/`（`main.py` 与 `app.json` 同级）。
2. 启动 OlivOS → 插件列表里会出现「NovelAI画图」，启用并重载。
3. 首次使用先配 key（见下一节），然后在群里发 `画图帮助`。

> 目录名本身不影响加载，`app.json` 的 `namespace` 才是插件身份（本版为 `NovelAIPainterV9`）。
> 如果你从旧的 `NovelAIPainterV8` 目录升级：插件身份已更正为 V9，与代码里的 `_NS` 一致；数据目录 `plugin/data/NovelAIPainterV9/data/` 无需搬迁。

### 环境要求

| 项 | 值 |
|---|---|
| OlivOS | 0.11.x（实测 0.11.81），`app.json` 声明 `compatible_svn: 190` |
| Python | OlivOS 内置环境即可（3.8+，实测 3.11） |
| 第三方包 | 无 |

---

## 配置密钥

**本仓库不含任何真实 key。** 所有密钥只有两个来源，都不存在时对应渠道直接跳过：

### 方式一：环境变量（推荐容器/无交互部署）

| 变量 | 用途 |
|---|---|
| `NAI_API_TOKEN` | novelai.me 积分站「开放接口」token（`uai_` 前缀，站点「个人中心 → 开放接口」自助签发） |
| `NAI_WEB_TOKEN` | novelai.me 网页端登录 JWT（解锁 Vibe/图生图与**真实积分余额**查询） |
| `NAI_DS_KEY` | LLM 主渠道 key（默认 DeepSeek 官方 `api.deepseek.com`） |
| `NAI_MASTER_QQ` | 骰主 QQ（留空 = 无骰主，「仅骰主」指令对所有人不可用） |
| `NAI_GEMINI38_URL` / `_KEY` / `_MODEL` | 第二个写词渠道（任意 OpenAI 兼容中转站） |
| `NAI_GLM53_URL` / `_KEY` / `_MODEL` | 第三个写词渠道 |
| `NAI_GROK43_URL` / `_KEY` / `_MODEL` | 第四个写词渠道 |

### 方式二：群内热更（推荐自建机器人）

```text
热更 nai <uai_ token>          # 积分站开放接口 token
热更 web <JWT>                 # 网页端登录态（余额/参考图）
热更 ds <key>                  # LLM 主渠道
热更 ds.<渠道名> <key>         # 指定 providers 里的某一条
热更                            # 查看当前各渠道配置状态（key 一律脱敏显示）
```

热更结果落盘到 `OlivOS/plugin/data/NovelAIPainterV9/data/api_keys.json`（文件优先于环境变量）。
**这个文件永远不要提交进仓库**——`.gitignore` 已经排除。

---

## 指令速查

| 指令 | 说明 |
|---|---|
| `形状` / `形状 <档位>` | 竖图 / 横图 / 方图（默认竖图，均在 0 积分免费包络内） |
| `画图/绘图/生图/作图 <英文tag>` | 按当前形状生图 |
| `透明背景 <英文tag>` | 透明底 PNG 直出（表情包 / 立绘素材） |
| `再来一张` / `重画` | 用上一次完整参数换 seed 重发 |
| `智能绘图/智能画图/智能生图/智能作图 <要求>` | AI 写提示词并生图（全年龄，限额 20/200，骰主不限） |
| `智能生图nsfw <要求>` | NSFW 版智能生图（仅白名单，默认硬核） |
| `智能生图轻度/诱惑/硬核/重口 <要求>` | 按指定分级生图（仅白名单） |
| `分级` / `分级 <0-4\|名称>` | 查看或切换生图分级 |
| `创意生图 <模糊要求>` | 创意模式（AI 自主补全 2-4 个风格化元素，全年龄） |
| `创意生图nsfw <模糊要求>` | 创意模式 NSFW 版（仅白名单） |
| `改图/微调/修改生图 <修改意见>` | 多轮对话式差量微调（保留人设与构图） |
| `画风` / `画风 <代号>` | 用户自定义画风体系 |
| `质量` / `质量 <档位>` | 无 / 轻度 / 重度 / 人像（默认重度） |
| `白名单` / `白名单 添加\|移除 <QQ>` | 白名单管理（仅骰主） |
| `周额度` / `本周额度` | 查看本周还剩几张 |
| `周额度 重置` / `周额度阈值 <白> <白分> <普通>` | 仅骰主 |
| `热更` | 查看 / 更新各渠道 key（仅骰主） |
| `画风导出` / `画风导入 <JSON>` | 画风预设整体导出/导入（骰主 + 白名单） |
| `添加画风 <代号> <名称> <配方>` / `删除画风 <代号>` | 单条画风增删 |
| `画图余额` / `查余额` | 查积分余额（需 `web` token） |
| `画图帮助` | 帮助 |

> `tag` 里可以用 `UC:` 指定反向提示词；`分级` 的 0-3 为公开档位，`4 重口` 为白名单隐藏档。

---

## 数据目录

```text
OlivOS/plugin/data/NovelAIPainterV9/data/
├── api_keys.json            # 热更写入的 key（勿提交）
├── artist_presets.json      # 画风预设
├── character_library.json   # 角色库
├── switch_config.json       # 各群生图开关 / 分级等运行配置
├── usage_weekly.json        # 周额度账本
├── usage_monthly.json       # 月账本
└── llm_providers_state.json # 渠道故障计数
```

---

## 版本沿革

| 版本 | 关键变化 |
|---|---|
| 8.0.0 | 创意模式：AI 在模糊需求上自主补全 2-4 个风格化元素（忠实度铁律：用户明确指定不可覆盖、SAFE 零自主 NSFW） |
| 9.0 | 生图通道切到 `novelai.me` 积分站开放接口（sdapi 兼容 + Basic Auth）；删除参考图/图生图链路；多角色框合并单 prompt；LLM 改多渠道 `providers`（priority + 故障转移） |
| 9.1 | 「人像」重新成为独立负面档位 |
| 10.5 | NSFW 破限硬化：系统提示词追加 `_NSFW_HARD_LOCK`（取自 [dsh-purge](https://github.com/YuJunZhiXue/dsh-purge)，MIT），运行时对拒答/截断回喂重试 |
| 10.6 | 语料经验同步：391 张真实 V5 成图反读的画师串 / 负强调 / 互动归属写法灌入系统提示词；计费口径更正（≤1MP 且 ≤28 步 且 单张不耗积分）；周额度硬闸门 |

详细见 [CHANGELOG.md](CHANGELOG.md)。

---

## 安全说明

- 仓库里**没有任何真实 API key / token / 个人 QQ**：`api_keys.json` 与骰主 QQ 全部走环境变量或数据目录。
- 提交前请自查：`git diff` 里不应出现 `sk-`、`uai_`、`eyJ`（JWT）之类的字符串。
- 插件自带的 NSFW 分级只对**白名单**生效，默认公开档位止于「3 硬核」；请按你的群规配置白名单与阈值。

## 许可与致谢

- 本项目以 **MIT** 协议开源，见 [LICENSE](LICENSE)。
- `_NSFW_HARD_LOCK` 片段取自 [YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge)（MIT）。
- 感谢 OlivOS 与 OlivaDice 生态提供的插件框架与事件模型。
