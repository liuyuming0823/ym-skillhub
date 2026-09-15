---
name: ym-skillhub-publisher
slug: ym-skillhub-publisher
display_name: SkillHub 发布器 · 一键创建/更新
display_name_en: SkillHub Publisher
displayName: SkillHub 发布器
description: "把本地技能目录一键创建或更新到 SkillHub（skillhub.cn）：自动补齐平台字段、递增版本号、预检文件白名单与市场规范、调用 CLI 发布、复核线上真实状态。当用户说「发布技能到 skillhub」「更新 skillhub 上的技能」「重新发一版」「技能发版」「上架 skillhub」「发布预检」「查线上技能状态」时使用。 也适用于「发布技能到skillhub」「更新skillhub技能」「重新发布技能」「上架skillhub」这类说法。"
description_zh: "一键把本地技能创建或更新到 SkillHub，并复核线上真实状态"
description_en: "Create or update a SkillHub skill via CLI, then verify the live listing"
summary: "一键把本地技能创建或更新到 SkillHub，并复核线上真实状态"
category: dev-programming
version: 1.0.0
author: 刘玉明
tags: [skillhub, 技能发布, 技能更新, 技能上架, 技能发版, 发布预检]
trigger:
  - 发布技能到skillhub
  - 更新skillhub技能
  - 重新发布技能
  - 技能发版
  - 上架skillhub
  - 发布预检
  - 查线上技能状态
agent_created: true
---

# SkillHub 发布器 · 一键创建/更新 (ym-skillhub-publisher)

把「把技能发到集上」从一串易错的手工命令，变成一条命令：**自动补齐平台字段 → 递增版本 → 预检 → 发布 → 复核线上真实状态**。

它替用户省掉的不是敲命令的力气，而是三类**静默失败**：字段写错不报错、发布成功但其实没上线、目录里躺着个不该有的文件导致整单被拒。

## 何时使用

| 用户说 | 走哪个子命令 |
|---|---|
| 「把这个技能发到 skillhub」 | `release`（一条龙） |
| 「更新 skillhub 上的 XX 技能」「重新发一版」 | `release`（默认 patch 递增） |
| 「先看看能不能发」「发布预检」 | `check` |
| 「商店里字段不对 / 名字是英文」 | `fix` 补字段，再 `release` |
| 「发了吗？线上是什么版本」 | `status` |
| 「不确定环境能不能发」 | `doctor` |

## 运行前提

- 本机已装 SkillHub CLI：`~/.skillhub/skills_store_cli.py`，且已登录（`~/.skillhub/credentials.json`
  或环境变量 `SKILLHUB_TOKEN`）。没登录先在 PowerShell 跑 `skillhub login --key skh_xxx`。
- **无第三方依赖**，只用标准库。默认用系统 python
  （`%LOCALAPPDATA%\Microsoft\WindowsApps\python.exe`）—— 本机 managed 3.13 跑网络请求会 segfault；
  需要覆盖时设环境变量 `SKILLHUB_PYTHON`。
- 发布要联网（平台三线审核：内容合规 + 科恩漏洞扫描 + 云鼎 AI 安全评估）。

## 命令速查

```bash
S=~/.workbuddy/skills/ym-skillhub-publisher/scripts/skillhub_publish.py

python $S doctor                          # 环境自检：CLI / python / 登录 / 网络
python $S check  <技能目录>                # 发布预检（不发 HTTP，除 CLI dry-run）
python $S fix    <技能目录> [--category X] # 自动补齐平台字段，幂等
python $S bump   <技能目录> [--patch|--minor|--major|--set X.Y.Z]
python $S publish <技能目录> [--changelog "..."] [--dry-run]
python $S status <slug> [--expect 1.2.0]  # 查线上真实状态
python $S release <技能目录> [--changelog "..."] [--category X] [--dry-run]
```

`release` 的开关：`--set X.Y.Z`（指定版本）/ `--minor` / `--major` / `--fix-only`
（只补字段不发布）/ `--force`（预检有阻断也发）。退出码：`0` 通过 / `1` 告警 / `2` 阻断 / `3` 环境不可用。

## 执行步骤

### 场景 A · 发布或更新一个技能（主流程）

1. **先自检环境**（首次使用必做）：`doctor`。四项全绿再往下；未登录就直接把登录命令给用户，别硬试。

2. **一条龙发布**：

   ```bash
   python scripts/skillhub_publish.py release <技能目录> --changelog "本次改了什么"
   ```

   脚本按 5 步走，每步都会打印结论：

   | 步 | 做什么 | 失败怎么办 |
   |---|---|---|
   | 1/5 fix | 补 `slug` / `displayName` / `summary` / `tags`，把块标量、多行列表、行尾注释规整成平台能读的形态 | 分类不在枚举内会提示用 `--category` 指定 |
   | 2/5 bump | 版本递增（默认 patch） | 当前值不是 SemVer 时报错，改用 `--set X.Y.Z` |
   | 3/5 check | 平台字段 + 解析器怪癖 + 扩展名白名单 + 本地市场体检 + CLI dry-run | 有阻断**自动中止发布**，逐条列出原因 |
   | 4/5 publish | 调 `skillhub CLI publish` | 返回非 `✓` 即失败，把原始输出贴出来 |
   | 5/5 status | 查线上 latestVersion / category / 版本列表 | 网络问题只告警，不影响发布结论 |

3. **把结果讲清楚**（这一步不能省），必须包含三件事：
   - 版本号与 skillId；
   - 线上**当前** `latestVersion` 是哪个版本，以及新版本在不在「已过审版本」列表里；
   - **明确说「已上线」还是「审核中」**——两者在命令输出上长得一模一样，只有版本列表能区分。

### 场景 B · 只想补齐字段（不改版本、不发布）

```bash
python scripts/skillhub_publish.py fix <技能目录> --category dev-programming
python scripts/skillhub_publish.py check <技能目录>      # 复核
```

`fix` 是**行级改写**，只动它要动的那几行，`SKILL.md` 正文一字不改；重复跑不会累积改动。

### 场景 C · 只想知道线上什么状态

```bash
python scripts/skillhub_publish.py status <slug> --expect 1.2.0
```

判定规则：`--expect` 的版本出现在**已过审版本列表**里 = 已上线；`stats.versions` 比列表长度大
= 有版本在审核中。不带 `--expect` 就只报现状。

### 场景 D · 分类要改

分类**改不动就是改不动**：CLI 发布 payload 里没有 `category`，平台也不解析 SKILL.md 里的
`category`。`fix --category` 只能让仓库里的声明正确，**线上分类必须去网页 dashboard 手动改**。
遇到这种情况，直接告诉用户去哪儿点，别反复重发。

## 目录说明

- `scripts/skillhub_publish.py` —— 全部逻辑（frontmatter 行级读写、CLI 调用、状态查询）
- `references/platform-fields.md` —— 平台字段规范、13 个分类枚举、解析器怪癖、文件白名单
- `references/troubleshooting.md` —— 按症状排错：发布 ≠ 上线、分类改不了、slug 占用、
  Windows/Git Bash 环境坑、只读状态接口清单

正文够用就不必读 `references/`；遇到「字段没生效」「发布被拒」「不知道线上为什么没变」再去查。

## 常见坑

1. **把 `✓ Published` 当「已经上线」** —— 它只代表平台**受理**。线上字段分三档：`tags` 秒级生效；
   `version` / `summary` / 描述 / 版本列表 / 下载包要等三线安全审核通过（实测约 3 分钟）；
   `category` 走 CLI 永远不变。发完立刻看商店「没变化」是正常的，先查版本列表确认新版本号在不在。

2. **看到商店没变就立刻重发** —— `stats.versions` 把**待审版本也计入**，所以「计数涨了但版本列表
   没动」= 审核中，不是失败。重发只会多出几个无意义的版本号。判据永远是**版本列表里有没有那个版本号**。

3. **以为改 SKILL.md 的 `category` 能改分类** —— 实测：分类写对了、过审成为 `latestVersion` 之后，
   线上仍是旧值。平台根本不解析 SKILL.md 的 `category`（CLI payload 里也没有这个字段）。
   改分类只能去网页 dashboard。

4. **`displayName` 写成 `display_name`** —— 平台只认驼峰。缺了**不报任何错、发布照样成功**，
   只是商店列表里显示英文 slug。静默失败里最容易漏的一个。要发布的技能**两个都写**。

5. **`category` 用了 `development`** —— 不在 13 个枚举内，对应值是 `dev-programming`；
   填错不报错，上架后显示「未分类」。枚举见 `references/platform-fields.md`。

6. **`description` 写成 YAML 折叠块（`>-`）** —— 平台解析器只读「`key: 值`」那一行，`>-` 会被
   读成**字面量两个字符**，描述在商店里就没了。`tags:` 写多行 `- 项` 同理被读空。
   长文本折一行、列表用 `[a, b]`、注释独立成行。

7. **把 `trigger` 也一起「规整」了** —— `trigger` / `display_name` / `agent_created` 是
   **WorkBuddy 本机字段**，平台不看。脚本只规整 `PLATFORM_KEYS` 里的字段，擅自压平本机字段
   会把技能本机就弄坏。

8. **技能目录里留着非白名单文件** —— 平台按扩展名**逐个校验仓库文件**，命中一个就整单拒收
   （报「不支持的文件类型: xxx」）。除图标外最容易忽略的：`.gitignore` / `.gitattributes`
   （迁到 `.git/info/`）、`*.template`（改 `.md` 后缀）、附件与 zip（放目录外）。
   **「本地打包器排除了」≠「发布时不校验」。** 自查：`git ls-files | sed 's/.*\.//' | sort -u`。

9. **在 Git Bash 里直接跑 `skillhub`** —— 会 `RuntimeError: Could not determine home directory`
   （shim 的 `HOME` / `PATH` 残缺）。用本脚本就不用管（它显式传 `HOME` / `USERPROFILE`）；
   手工排错时也要照做。

10. **拿下载接口判断「slug 是不是被别人占了」** —— 200/302 只说明「该 slug 已被发布」，
    **自己发的也是 302**；而 404 也不代表可用（草稿 / 审核中 / 私有同样是 404）。
    外部接口只能排除「已被别人发布」的名字，最终仍要在发布表单里试一次（要认作者看 `namespace.handle`）。
