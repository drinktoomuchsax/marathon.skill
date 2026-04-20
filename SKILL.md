---
name: marathon
description: 当用户想让 Claude Code 无人值守自主跑数小时（跨多轮 agent loop）完成一个开放式任务时触发。典型触发词：「自主跑 N 小时」、「无人值守」、「marathon」、「overnight 跑」、「开一个 12h 任务」、「让 claude 自己做完」、「autoloop」。本 skill 用 Stop hook 拦截每次退出并注入续跑指令，直到时长或 block 次数到上限。
---

# Marathon · 无人值守自主长跑

## 你在处理什么

用户要让 Claude Code **无人值守**连续跑数小时（常见 4h / 8h / 12h），期间不会回答任何问题。实现方式：一个 Stop hook 脚本在每次 agent 想退出时拦住它、喂入续跑指令，满足硬停条件（时长到 / block 次数到上限）才真正放行。

整套机制 3 个文件支撑：`.claude/hooks/stop.py`（拦截逻辑）+ `.claude/settings.local.json`（注册 hook）+ `task.md`（给用户看的任务契约）+ `CLAUDE.md`（给 agent 看的工作协议）。

## 流程（严格按此 3 步执行）

### Step 1 · 确认参数

通过对话问清楚 4 件事：

1. **任务目录**（绝对路径）。若用户没指明，建议一个新建目录，**不要**混进有业务代码的仓库
2. **任务标题**（短，用于文件头和日志识别）
3. **时长**（默认 12h，接受 `2h / 4h / 8h / 12h / 24h` 或自由数字）
4. **任务范围**（用户一句话即可，你负责在 task.md 里展开）

### Step 2 · 生成 4 个文件

按顺序写入用户指定的目录：

1. `task.md` —— 照 `template/task.md.tmpl` 生成；用 Step 1 的信息填占位符；范围、不做、交付标准你根据用户描述起草
2. `CLAUDE.md` —— 照 `template/CLAUDE.md.tmpl` **原样**拷贝（这份是固定骨架，不修改）
3. `.claude/hooks/stop.py` —— 照 `template/stop.py.tmpl` 拷贝；**仅**修改文件头的 `DURATION_HOURS` 常量
4. `.claude/settings.local.json` —— 照 `template/settings.local.json.tmpl` **原样**拷贝

同时创建空目录 `logs/` 和 `reports/`。

### Step 3 · 告诉用户怎么启动（不要替用户启动）

打印下面两行（按用户的 shell 选一条），并**明确告诉用户 review `task.md` 后再敲命令**：

```powershell
# PowerShell（Windows 推荐）
cd <任务目录>
python -c "import time; open('exp-start','w').write(str(int(time.time())))"
claude --dangerously-skip-permissions "进入无人值守自主模式。严格阅读并遵循 CLAUDE.md 与 task.md，立刻开始交付第一个最小可运行子任务。"
```

```bash
# Git Bash / Linux / macOS
cd <任务目录>
date +%s > exp-start
claude --dangerously-skip-permissions "进入无人值守自主模式。严格阅读并遵循 CLAUDE.md 与 task.md，立刻开始交付第一个最小可运行子任务。"
```

**关键原则：skill 只负责搭好结构、起草 task.md；启动键由用户自己按。** 12 小时无人值守前必须有用户 review + 点头。

## 重要约束

- **同一任务一个目录**。不支持同目录多任务命名空间；想多任务 = 多目录
- **hook 是 Python 脚本**（不是 bash），因为 Windows 下 MSYS bash 被 Claude Code `CreateProcess` 调起时会失败
- **Stop hook 的合法 schema** 只能用 `decision` + `reason`，**禁止**加 `hookSpecificOutput`（Stop 事件不支持，会被 Claude 拒绝）
- **`exp-start` 存 Unix epoch 秒**，必须用 `python -c "import time; ... int(time.time())"` 或 `date +%s` 生成，**不能用** PowerShell 的 `Get-Date -UFormat %s`（Windows 下时区基准不对）
- **`task.md` 面向用户**，写具体做什么；**`CLAUDE.md` 面向 agent**，写工作协议。分工不可混淆
- **chmod +x 对 .py 没意义**；启动脚本（如果有）才需要执行位，但本 skill 没有启动脚本，直接命令行一行 done

## 常见坑位（排错参考）

| 症状 | 原因 | 修法 |
|---|---|---|
| `Hook JSON output validation failed` | 你往 payload 里塞了 `hookSpecificOutput` | 删掉，Stop 事件只接受 `decision` + `reason` |
| `cannot execute binary file` | settings 里 hook command 写的是 `bash xxx.sh`，Windows `bash` 指向 WSL 或 MSYS 内部 bash | 改为 `python .claude/hooks/stop.py` |
| `elapsed=-26639` 负数 | `exp-start` 时间戳基准错（PowerShell `Get-Date -UFormat %s` 在 Windows 时区下会偏） | 用 `python -c "import time; ..."` 重写 |
| 中文被转成 `\u00xx` | Python stdout 编码是 cp1252 | 脚本里用 `sys.stdout.buffer.write(json.dumps(..., ensure_ascii=False).encode("utf-8"))` |
| Agent 开始乱改仓库其他文件 | task.md 没明确"不碰目录外文件" | task.md 的非范围清单必须写"禁止访问本任务目录之外的文件" |

## 监控与收尾

运行中查日志（提醒用户另开窗口）：

```powershell
Get-Content <任务目录>\logs\stop-hook.log -Wait -Tail 20
```

紧急放行（下次 Stop hook 立刻让 agent 退出，不用 Ctrl+C）：

```bash
# 把 exp-start 倒推到 13 小时前
python -c "import time; open(r'<任务目录>/exp-start','w').write(str(int(time.time())-13*3600))"
```

12h 到后，产出都在任务目录的 `progress.md` 和 `reports/hour-*.md` 里。
