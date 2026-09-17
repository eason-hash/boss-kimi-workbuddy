---
name: boss-zhipin-deliver
description: |
  BOSS直聘自动化投递技能 v5.3 — 三层循环架构（关键词进化 + 推荐页串行投递）。
  依赖 Kimi WebBridge（端口 10086）控制已登录 Chrome，在推荐页懒加载全量岗位、Agent 逐条语义决策后投递。
  支持环境自检、投递断点续传、每日目标检测、HTML 汇报生成（含微信/支付宝打赏二维码）。
  ★ v5.3: 面向 GitHub 通用发布，所有用户画像存储在 user_profile.json，skill 本身零硬编码。
metadata:
  version: "5.3.0"
---

# BOSS直聘自动化投递 Skill v5.3

> 本技能加载后，Agent 将按三层循环架构执行 BOSS直聘自动化投递。
> 依赖 Kimi WebBridge（端口 10086）控制已登录 Chrome。
>
> **核心设计**：`user_profile.json` 是唯一用户数据源，skill 代码本身不含任何特定用户信息。
> 不同求职者只需修改 `user_profile.json` 即可复用整套投递流程。

---

## 核心原则

1. **执行脚本**：使用 `python_executor` 运行 `<SKILL_DIR>/scripts/` 下的 .py 文件
2. **AI 决策**：岗位 APPROVE/REJECT 必须由 Agent 根据 user_profile.json 语义判断，禁止脚本硬编码匹配
3. **WORK_DIR**：默认使用 Marvis 当前会话的 output 目录，可通过 `BOSS_WORK_DIR` 环境变量覆盖
4. **Kimi WebBridge**：所有浏览器操作通过 `use_skill("kimi-webbridge")` 完成，禁止 CDP 直连

---

## 架构总览

```
scripts/
├── profile_loader.py      — 中央配置加载器（从 user_profile.json 读取）
├── env_check.py           — 环境自检 + 自动修复
├── webbridge_client.py    — WebBridge HTTP 客户端（端口 10086）
├── recommend_loader.py    — 推荐页/搜索页导航 + 分批懒加载
├── surface_cache.py       — 精简字段缓存（7 核心字段）
├── jd_reader.py           — 页面内原地点卡读 JD
├── deliver_engine.py      — 投递执行 + 目标检查
├── report_builder.py      — HTML 汇报生成
├── keyword_evolver.py     — AI 关键词进化 + 搜索导航
├── serial_loop.py         — 三层循环主控制器
```

信号文件（位于 WORK_DIR）：
- `page_state.json` — 页面级进度
- `batch_state.json` — 分批级进度
- `surface_cache.json` — 全量缓存
- `jd_to_read.json` — AI 筛选后待读取列表
- `current_jd.json` — 当前 JD（供 Agent 阅读）
- `decision.json` — Agent 决策结果
- `stream_progress.json` — 投递断点续传

---

## 执行流程

### 步骤 0：确定 WORK_DIR

使用 Marvis 当前会话的 output 目录，或设置环境变量：
```
BOSS_WORK_DIR = <当前会话 output 目录>
```

### 步骤 1：初始化（`--step init`）

```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step init
```

脚本会检查 WebBridge、导航到 BOSS 推荐页、重置状态文件。

> ⚠️ **URL 陷阱（必须遵守）**：`/web/geek/recommend` 是**已沟通/已投递历史页**，不是推荐页。页面标题为「BOSS直聘」且含「已投递」「沟通过」字样时即为错误页面。正确的推荐/职位搜索页 URL 为 **`https://www.zhipin.com/web/geek/job`**（实际跳转后为 `/web/geek/jobs`，标题为「{城市名}招聘」）。Agent 导航后必须 snapshot 验页：标题不含目标城市名 = 错页，立即修正。

**失败处理**：若 WebBridge 不可达，先调用 `use_skill("kimi-webbridge")` 确保 daemon 运行。

### 步骤 1.5：确认投递方向

> ⚠️ **必须执行**：本 skill 面向所有用户发布，Agent **严禁**跳过此步骤直接开始投递。

Agent 必须读取 user_profile.json，向用户展示下列关键筛选规则并等待用户确认后，才能进入步骤 2：

- **用户姓名**：`user_name`
- **目标城市**：`target_city`（编码: `city_code`）
- **最低薪资**：`min_salary_k`（单位：K，0 表示不限制）
- **日投递目标**：`daily_target`（份）
- **目标领域描述**：`domain_description`（若未填则展示 prefer/avoid 列表）
- **偏好岗位**：`prefer` 列表
- **排除岗位**：`avoid` 列表
- **额外规则**：是否排除大厂（`prefer_no_big_company`）、是否排除猎头（`prefer_no_headhunter`）
- **搜索关键词**：`search_keywords` 列表（前 10 个）

展示后询问用户：是使用这些默认规则，还是需要调整？用户确认后进入步骤 2。

### 步骤 2：中层循环 — 滚动加载（`--step next-batch`）

```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step next-batch
```

加载一批约 50 条岗位，追加到 `surface_cache.json`。输出岗位列表供 Agent 初筛。

**Agent 初筛**：Agent 读取输出中的岗位列表，基于 `user_profile.json` 的筛选规则（prefer/avoid/search_keywords/domain_description），将合资格岗位追加写入 `jd_to_read.json`（JSON 数组，每个元素含 page_order、jobId、title 等）。

### 步骤 3：内层循环 — 逐条读 JD + 决策 + 投递

对 `jd_to_read.json` 中每条岗位（按 page_order 顺序）：

**3a. 读取 JD**：
```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step read --index N
```
脚本将 JD 写入 `current_jd.json`。

**3b. Agent 决策**：
Agent 读取 `current_jd.json`，基于 `user_profile.json` 的筛选规则判断 APPROVE/REJECT：

- **APPROVE**：岗位匹配 prefer 列表中的目标方向，薪资 ≥ min_salary_k（若设置了最低薪资），城市匹配，不属于 avoid 列表
- **REJECT**：岗位属于 avoid 列表、domain_description 中声明的排除领域、猎头/外包（若 prefer_no_headhunter=true）、大厂（若 prefer_no_big_company=true）、或机械预筛排除项（保安/保洁/销售/主播/实习生等低相关度岗位）
- 不确定时宁可 REJECT

Agent 将决策写入 `decision.json`：
```json
{"decision": "APPROVE", "reason": "简短理由"}
```

**3c. 执行决策**：
```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step decide --index N
```
APPROVE 时自动投递并检查目标；脚本内置投递目标检查（DAILY_TARGET）。

**3d. 检查终止条件**：
- 投递目标达标 → **强制跳到步骤 5**
- 致命错误（每日上限）→ **强制跳到步骤 5**
- 本批处理完 → 继续步骤 2 下一批

### 步骤 4：外层循环 — 关键词进化（`--step evolve`）

页面穷尽后：
```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step evolve
```

脚本评估投递结果，获取下一个搜索关键词。Agent 确认关键词后：
```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step navigate-search --keyword "关键词"
```

回到步骤 2 继续。

**关键词穷尽（无新关键词）→ 强制跳到步骤 5**。不允许在没有新关键词时继续卡在步骤 4。

### 步骤 5：生成汇报 — ★ 强制执行（`--step finish`）

> **拦截性强制步骤。以下任意条件触发时必须立即执行，不得跳过：**
> - 投递目标达标
> - 致命错误（每日上限）
> - 所有页面穷尽且无新关键词
> - 用户主动要求停止投递

```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step finish
```

生成 `投递汇总.html` 到 WORK_DIR，内含微信赞赏码和支付宝收款码供求打赏。

---

## 随时查看进度

```
python_executor: 运行 <SKILL_DIR>/scripts/serial_loop.py --step status
```

---

## 环境检查

```
python_executor: 运行 <SKILL_DIR>/scripts/env_check.py
```
包括账户匹配检测（user_profile.json 的用户名/城市 vs 实际登录账户）。

---

## 配置说明

### user_profile.json 字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `user_name` | string | 否 | 用户姓名，用于账户匹配和报告 |
| `role_summary` | string | 否 | 一句话角色概述 |
| `experience` | string | 否 | 详细经验描述，用于 LLM 决策 |
| `domain_description` | string | 推荐 | **目标领域详细描述**（适合/不适合的岗位范围），会注入 LLM 审核 prompt。不填则自动由 prefer/avoid 拼接 |
| `target_city` | string | 是 | 目标城市名（如"北京"） |
| `city_code` | string | 是 | 城市编码（如"101010100"），从 BOSS直聘 URL 获取 |
| `min_salary_k` | number | 否 | 最低薪资（K），0 表示不限制 |
| `daily_target` | number | 是 | 每日投递目标（份） |
| `prefer` | string[] | 否 | 偏好岗位关键词列表 |
| `avoid` | string[] | 否 | 排除岗位关键词列表 |
| `prefer_no_big_company` | boolean | 否 | 是否排除大厂（已上市/2000人以上），默认 false |
| `prefer_no_headhunter` | boolean | 否 | 是否排除猎头岗位，默认 false |
| `search_keywords` | string[] | 是 | BOSS直聘搜索关键词列表（纯职位词，不含城市） |
| `c_exclude` | string[] | 否 | 自定义公司名黑名单。不填则使用内置通用默认值（排除猎头/外包/房产/保险） |
| `t_exclude` | string[] | 否 | 自定义标题排除词。不填则使用内置通用默认值（排除保安/保洁/销售/主播/实习生等） |

### 城市编码获取方式

1. 在浏览器中打开 https://www.zhipin.com/
2. 顶部选择目标城市
3. 查看 URL：`https://www.zhipin.com/web/geek/job?city=XXXXXX` → `city=` 后的数字即为 city_code

---

## 重要约束

1. **URL 红线**：`/web/geek/recommend` 是已投递历史页，禁止作为岗位来源。推荐页唯一正确 URL 是 `https://www.zhipin.com/web/geek/job`。导航后必须 snapshot 确认标题含目标城市名。
2. 绝不 navigate 到 job_detail URL（在原页面点卡读 JD）
3. 绝不点击「继续沟通」（投递后点「留在此页」）
4. 投递后必须验证实际结果
5. 每日上限判定为致命错误，立即停止
6. 所有浏览器操作必须通过 Kimi WebBridge，不可 CDP 直连
7. **强制 finish**：投递流程无论以何种方式终止，都必须执行 `--step finish` 生成投递汇总.html。禁止跳过此步骤。

---

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v5.4 | 2026-08-04 | 100% 收尾保证：step_decide target_met / fatal_limit 自动 finish；step_evolve target_met 自动 finish；新增 atexit 兜底钩子确保任何退出路径均生成投递汇总 |
| v5.3 | 2026-08-04 | 面向 GitHub 通用发布：去除所有硬编码用户画像，user_profile.json 为唯一数据源；LLM prompt 由 domain_description 驱动；排除列表支持用户自定义；修复 keyword_evolver 城市名硬编码 |
| v5.2 | 2026-08-04 | Marvis 适配版 |
