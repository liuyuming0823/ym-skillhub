---
name: ym-skillhub
slug: ym-skillhub
display_name: SkillHub 一条龙 · 搜索/安装/发布
display_name_en: SkillHub Toolkit
displayName: SkillHub 一条龙
description: "SkillHub（skillhub.cn）全套操作：搜索技能、安全安装第三方技能、把本地技能一键创建或更新到商店并复核线上状态。当用户说「skillhub」「技能商店」「搜索技能」「安装技能（从 skillhub）」「看看有没有 XX 技能」，或说「发布技能到 skillhub」「更新 skillhub 上的技能」「重新发一版」「技能发版」「上架 skillhub」「发布预检」「查线上技能状态」时使用。"
description_zh: "SkillHub 技能商店的搜索、安全安装与一键发布"
description_en: "Search, safely install, and publish skills on SkillHub"
summary: "SkillHub 技能商店的搜索 / 安全安装 / 一键发布与线上状态复核"
category: dev-programming
version: 2.2.0
author: 刘玉明
tags: [skillhub, 技能商店, 搜索技能, 安装技能, 技能发布, 技能更新, 技能发版, 发布预检]
trigger:
  - skillhub
  - 技能商店
  - 搜索技能
  - 安装技能
  - 发布技能到skillhub
  - 更新skillhub技能
  - 技能发版
  - 上架skillhub
  - 发布预检
  - 查线上技能状态
agent_created: true
---

# SkillHub 一条龙 · 搜索 / 安装 / 发布 (ym-skillhub)

国内优先的技能源。一份技能管三件事：**找别人的技能、安全地装上、把自己的发上去**。

它省掉的不是敲命令的力气，而是三类**静默失败**：字段写错不报错、发布成功但其实没上线、
目录里躺着个不该有的文件导致整单被拒。

## 这个技能管什么

| 用户说 | 场景 | 用什么 |
|---|---|---|
| 「skillhub 上有没有 XX 技能」 | **A · 搜索与安装** | `skillhub search` / `install` + `check_slug.py` |
| 「把这个技能发到 skillhub」「重新发一版」 | **B · 发布/更新** | `skillhub_publish.py release` |
| 「商店里名字是英文 / 字段不对」 | **C · 补齐字段** | `skillhub_publish.py fix` |
| 「发了吗？线上什么版本」 | **D · 查状态** | `skillhub_publish.py status` |
| 「分类不对」 | **E · 分类** | 网页 dashboard（脚本改不了） |

## 入口与环境

- **CLI 主程序**：`C:\Users\liuyuming\.skillhub\skills_store_cli.py`
- **命令入口**（已在本机 PATH，PowerShell / Git Bash 都能直接 `skillhub`）：
  `C:\Users\liuyuming\.local\bin\skillhub.cmd`（cmd/PS）、`skillhub`（Git Bash）
- **Python**：默认系统 `python.exe`（managed 3.13 跑网络请求会 segfault）；
  覆盖用环境变量 `SKILLHUB_PYTHON`。
- **登录**（仅发布需要）：`skillhub login --key skh_xxx`，token 落 `~/.skillhub/credentials.json`。
  不确定就跑 `doctor` 自检四项。
- 搜索 / 安装 / 查状态**无需登录**；发布要联网并过平台三线审核。

> ⚠️ 在 Git Bash 里直接跑 `skillhub` 会 `RuntimeError: Could not determine home directory`
> （shim 的 HOME/PATH 残缺）。脚本已内置 `cli_env()` 绕开；手工排错时也要显式传 `HOME` / `USERPROFILE`。

## 场景 A · 搜索与安装第三方技能

```bash
skillhub search <关键词>
skillhub install <slug> --namespace <ns> --dir "C:/Users/liuyuming/.workbuddy/skills"
skillhub skill reports <slug> --namespace <ns>      # 安全报告（腾讯云）
skillhub skill evaluation <slug> --namespace <ns>   # 边界/触发质量评分
```

搜索结果每行 `- install: skillhub install <slug> --namespace <ns>` 就是可复制的命令，**补上 `--dir` 即可**。

**标准流程（第三方技能必须走完）**：

1. **搜索** → 汇总候选（名称 / 命名空间 / 版本 / 一句话功能 / 是否需付费 Key），让明哥挑。
2. **隔离安装**：`--dir` 先指到临时 quarantine 目录，不碰正式技能目录。
3. **逐文件审查**：扫 `os.system` / `subprocess` / `eval` / `exec` / `rmtree` / 外联域名 / 读取无关文件；
   同时拉 `reports` + `evaluation`。
4. **冒烟测试**：缺 Key / 缺输入时跑一次，确认「不发起任务、不扣点、退出码符合文档」。
5. **告知风险**再拍平：说清会上传什么数据到哪里、是否扣费、是否要配 Key。
6. **拍平落位** + 改 lock 文件（社区命名空间装完是两层嵌套，WorkBuddy 扫不到）。
7. 提醒明哥：新技能可能要**重启 WorkBuddy** 才被识别。

> 完整审查清单、拍平脚本、命名空间说明见 `references/search-install.md`。

## 场景 B · 发布或更新自己的技能

```bash
python scripts/skillhub_publish.py release <技能目录> --changelog "本次改了什么"
```

首次使用先跑 `doctor` 自检（CLI / python / 登录 / 网络四项）。

脚本按 5 步走，每步都打印结论：

| 步 | 做什么 | 失败怎么办 |
|---|---|---|
| 1/5 fix | 补 `slug` / `displayName` / `summary` / `tags`，把块标量、多行列表、行尾注释规整成平台能读的形态 | 分类不在枚举内会提示用 `--category` 指定 |
| 2/5 bump | 版本递增（默认 patch） | 当前值不是 SemVer 时改用 `--set X.Y.Z` |
| 3/5 check | 平台字段 + 解析器怪癖 + 扩展名白名单 + 本地市场体检 + CLI dry-run | 有阻断**自动中止发布**，逐条列出原因 |
| 4/5 publish | 调 `skillhub CLI publish` | 返回非 `✓` 即失败，把原始输出贴出来 |
| 5/5 status | 查线上 latestVersion / category / 版本列表 | 网络问题只告警，不影响发布结论 |

**讲结果时这三件事不能省**：① 版本号与 skillId；② 线上**当前** `latestVersion` 是哪个、
新版本在不在「已过审版本」列表里；③ **明确说「已上线」还是「审核中」** —— 两者在命令输出上
长得一模一样，只有版本列表能区分。

`release` 开关：`--set X.Y.Z` / `--minor` / `--major` / `--fix-only`（只补字段不发布）/
`--force`（预检有阻断也发）。退出码：`0` 通过 / `1` 告警 / `2` 阻断 / `3` 环境不可用。

## 场景 C · 只想补齐字段（不改版本、不发布）

```bash
python scripts/skillhub_publish.py fix <技能目录> --category dev-programming
python scripts/skillhub_publish.py check <技能目录>
```

`fix` 是**行级改写**，只动要改的那几行，正文一字不改；重复跑不会累积改动。

## 场景 D · 只想知道线上什么状态

```bash
python scripts/skillhub_publish.py status <slug> --expect 1.2.0
```

`--expect` 的版本出现在**已过审版本列表**里 = 已上线；`stats.versions` 比列表长度大 = 有版本在审核中。
不带 `--expect` 只报现状。

## 场景 E · 分类要改

**改不动就是改不动**：CLI 发布 payload 里没有 `category`，平台也不解析 SKILL.md 里的 `category`。
`fix --category` 只能让仓库里的声明正确，**线上分类必须去 https://skillhub.cn/dashboard 手动改**。
遇到这种情况直接告诉明哥去哪儿点，别反复重发。

## 命令速查

```bash
S=~/.workbuddy/skills/ym-skillhub/scripts

# ── 搜索 / 安装 ──
skillhub search <关键词>
skillhub install <slug> --namespace <ns> --dir "C:/Users/liuyuming/.workbuddy/skills"
python $S/check_slug.py ym-foo ym-bar            # 批量查 slug 是否被占

# ── 发布 / 更新 ──
python $S/skillhub_publish.py doctor             # 环境自检
python $S/skillhub_publish.py check <技能目录>    # 发布预检
python $S/skillhub_publish.py fix   <技能目录>    # 补平台字段（幂等）
python $S/skillhub_publish.py bump  <技能目录>    # 版本递增
python $S/skillhub_publish.py status <slug> [--expect X.Y.Z]
python $S/skillhub_publish.py release <技能目录> --changelog "..."

# ── 批量上架（一次推一批：发布 + 同步 GitHub + 双项校验）──
python $S/batch_release.py ym-foo ym-bar --gap 40          # 指定技能
python $S/batch_release.py --from-list list.txt            # 从清单读
python $S/batch_release.py ym-foo --stage github           # 只补推 GitHub
python $S/batch_release.py ym-foo ym-bar --dry-run         # 先看计划
```

## 目录说明

- `scripts/skillhub_publish.py` —— 发布全部逻辑（frontmatter 行级读写、CLI 调用、状态查询）
- `scripts/check_slug.py` —— 批量自查 slug 占用
- `scripts/batch_release.py` —— **批量上架**：发布（固定间隔防限流）+ 建仓推 GitHub + 双项校验，
  退出码 `0` 全成功 / `1` 有失败 / `2` 参数错。命令幂等，重跑失败项即可。
- `references/search-install.md` —— 搜索、安全审查清单、命名空间拍平、安装后配置
- `references/platform-fields.md` —— 平台字段规范、13 个分类枚举、解析器怪癖、文件白名单
- `references/troubleshooting.md` —— 按症状排错：发布 ≠ 上线、分类改不了、slug 占用、环境坑

正文够用就不必读 `references/`；遇到「字段没生效」「发布被拒」「不知道线上为什么没变」再去查。

## 常见坑

1. **把 `✓ Published` 当「已经上线」** —— 它只代表平台**受理**。线上字段分三档：`tags` 秒级生效；
   `version` / `summary` / 描述 / 版本列表 / 下载包要等三线审核（实测约 3 分钟）；`category` 走 CLI 永不变。
   判据是**版本列表里有没有那个版本号**。

2. **看到商店没变就立刻重发** —— `stats.versions` 把**待审版本也计入**，所以「计数涨了但版本列表没动」
   = 审核中。重发只会多出无意义的版本号。

3. **以为改 SKILL.md 的 `category` 能改分类** —— 分类写对、过审成为 `latestVersion` 之后线上仍是旧值。
   平台根本不解析它，只能去网页 dashboard 改。

4. **`displayName` 写成 `display_name`** —— 平台只认驼峰。缺了**不报错、发布照样成功**，
   只是商店显示英文 slug。要发布的技能**两套都写**。

5. **`category` 用了 `development`** —— 不在 13 枚举内，对应 `dev-programming`；填错不报错，显示「未分类」。

6. **`description` 写成 YAML 折叠块（`>-`）** —— 平台只读「`key: 值`」那一行，`>-` 被读成**字面量两个字符**。
   `tags:` 写多行 `- 项` 同理读空。长文本折一行、列表用 `[a, b]`、注释独立成行。

7. **把 `trigger` 也一起「规整」了** —— `trigger` / `display_name` / `agent_created` 是 **WorkBuddy 本机字段**，
   平台不看。脚本只规整 `PLATFORM_KEYS`；擅自压平本机字段会把技能弄坏。

8. **技能目录里留着非白名单文件** —— 平台按扩展名**逐个校验**，命中一个整单拒收
   （「不支持的文件类型: xxx」）。最易忽略：`.gitignore` / `.gitattributes`（迁到 `.git/info/`）、
   `*.template`（改 `.md`）、附件与 zip（放目录外）。**「本地打包器排除了」≠「发布时不校验」。**
   自查：`git ls-files | sed 's/.*\.//' | sort -u`。

9. **`--dir` 传了 MSYS 风格路径** —— `/c/Users/...` 会被原生 python 当**相对当前盘**解析，
   实际装到 `D:\c\Users\...`。**永远写 `C:/Users/...`**。任何传给脚本的路径同理。

10. **社区命名空间技能装完是两层嵌套** —— `~/.workbuddy/skills/@<ns>/<slug>/SKILL.md`，
    WorkBuddy 只扫平铺目录，必须拍平并改 lock 文件里的 `installDir`。

11. **拿下载接口判断「slug 是不是被别人占了」** —— 200/302 只说明「该 slug 已被发布」，
    **自己发的也是 302**；404 也不代表可用（草稿/审核中/私有同样是 404）。要认作者看 `namespace.handle`。

12. **用搜索接口判重** —— `search?q=` 是模糊匹配，短前缀词会返回兜底热门榜，看着像无结果其实是没匹配上。
    判重用 `check_slug.py`（下载接口），但仍需在发布表单里最终确认。

13. **连续发布不加间隔** —— SkillHub 有频率限制，连发到第 4 个开始报「发布频率过高」。
    **每个之间 sleep 40 秒**（实测连发 30 个零失败，约 20 分钟跑完）。批量发布直接照这个节奏，
    别等报错再补救式重试。失败后同样按 40 秒间隔重发可以全部追回。

14. **批量建仓时 GitHub 返回 422** —— 连推十几个仓库后 `POST /git/trees` 或 `/git/commits`
    会返回 `422 Unprocessable Entity`，这不是参数错，是**次级限流**。隔一会儿重跑同一个仓库即可，
    通常报「tree 已一致，无需推送」。`push_via_api.py` 已内建 403/422/429/5xx 退避重试（3 次）。

15. **仓库里有中文等非 ASCII 文件名** —— git 默认 `core.quotepath=true`，
    `git ls-tree -r --name-only` 会把中文文件名（例如 `风险规则清单.md`）输出成
    `"\351\243\216..."` 这种八进制转义串，脚本据此 `cat-file` 必然取不到对象，
    报「HEAD 中不存在该路径」而**整仓库静默失败**。`push_via_api.py` 现已改用
    `git -c core.quotepath=false ... -z`，中文文件名可正常推送；自建脚本也要带这两个参数。

## 给明哥的操作约定

1. 搜索后先汇总候选让明哥挑，**不要擅自安装**。
2. 装前说清来源、数据流向、是否扣费、是否需要 Key。
3. 装完提示：新技能可能需要重启 WorkBuddy 才被识别。
4. 安装时 `--dir` 显式写 `C:/Users/liuyuming/.workbuddy/skills`。
5. 明哥说「安装 XX 技能」但 XX 可能是**发布方**而非单个技能（如「零一数科」= 命名空间 `org-28ib33ph`，
   下挂 19 个 `lingyi-*`）—— 先搜清楚是技能还是发布方，是发布方就列清单让明哥挑。
6. 发布完必须说清「已上线 / 审核中」，别只回一句「发布成功」。

## 环境注意（Windows）

- 技能发现/安装**优先用 `skillhub`**；无匹配或不可用时再回退其他来源并说明。
- Bash 工具里用 `D:\git\Git\bin\bash.exe` 并先
  `export PATH="/d/git/Git/usr/bin:/d/git/Git/mingw64/bin:$PATH"`（默认 bash shim 的 PATH 残缺，<!-- skill-audit: ignore -->
  连 `mkdir` / `dirname` 都没有）。
- 中文乱码是控制台编码问题：包装器已设 `PYTHONIOENCODING=utf-8`；PowerShell 读长输出用
  `| Out-File "$env:TEMP\x.txt" -Encoding utf8` 再用 Read 看（直接返回的 stdout 常被吞）。
- CLI 的 `config` 只支持 `list`，**不能持久化默认 `--dir`**，所以每次都要显式传。
- `skillhub list` 看不到用 `--dir` 装的技能（只扫默认目录）；`skillhub verify` 对社区命名空间返回 **405**。
- CLI 发布走上传通道，**不需要**先推 GitHub；只有走「网页绑定 GitHub 发布」才要保证仓库同步。
  本机沙箱常拦 `git push`（`Empty reply from server` / `502`），但 `gh api` 通：
  `python ~/.workbuddy/tools/push_via_api.py <仓库目录>`（**验证只看 tree sha**）。

## 版本历史

### v2.2.0 (2026-09-16)

新增 `scripts/batch_release.py` —— 把「一批技能从本地推到线上」固化成一条命令：
发布（固定 40 秒间隔防平台限流）+ 自动建仓推 GitHub（走 `push_via_api`，含退避重试）+ 双项校验，
幂等可重跑。实测一次推 30 个技能：SkillHub 30/30 上线、GitHub 30/30 tree 一致。

### v2.1.0 (2026-09-16)

批量发布实战补充：新增「常见坑」13~15（SkillHub 发布需 40 秒间隔、GitHub 批量建仓的 422
次级限流、中文文件名被 `core.quotepath` 转义导致整仓库静默推送失败）。
`push_via_api.py` 同步修复：`-c core.quotepath=false` + `-z` 解析路径，并内建退避重试。

### v2.0.0 (2026-09-15)

合并原 `skillhub-store`（搜索/安装）与 `ym-skillhub-publisher`（发布执行器）为一份，
统一 `ym-` 命名。新增 `references/search-install.md`，两个脚本并入同一 `scripts/`。

### v1.0.0 (2026-09-15)

初版 `ym-skillhub-publisher`：doctor / check / fix / bump / publish / status / release 七个子命令。
