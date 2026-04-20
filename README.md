# marathon.skill

让 Claude Code 无人值守连续跑数小时完成一个开放式任务。

灵感来自 [loopndroll](https://github.com/lnikell/loopndroll)（Codex CLI 的 infinite mode），在 Claude Code 上用 **Stop hook** 拦截每次退出、注入续跑指令，直到时长到达硬停。

## 它解决什么问题

你想启动一个"大活"，然后去睡觉 / 上班 / 吃饭，12 小时后回来看产出。Claude 正常完成一轮任务会退出，于是你睡一觉醒来发现它只跑了 20 分钟。marathon 让它持续工作——每次想退出时被 hook 拦住、喂入续跑 prompt、进入下一轮。

硬停条件只有两个：
- **时长到**（默认 12h）
- **连续 block 次数超过上限**（默认 5000，防任务空洞导致秒循环烧 token）

## 安装

```bash
git clone git@github.com:drinktoomuchsax/marathon.skill.git ~/.claude/skills/marathon
```

安装后在任意 Claude Code 会话里说"开一个 12h 自主任务做 X"即可触发。

（也支持把整个仓库直接当 skill 目录放到 `~/.claude/skills/<任意名字>/`，Claude Code 会自动读取 `SKILL.md` 里的 frontmatter。）

## 使用方式（3 步）

### 1. 让 Claude 起草

在 Claude Code 里说：

> 开一个 12 小时 marathon 任务，做 [你的任务]

Claude 会问你任务目录、时长等参数，然后在目标目录生成：

```
<任务目录>/
├── task.md                  ← 面向你：任务契约
├── CLAUDE.md                ← 面向 agent：工作协议
├── .claude/
│   ├── hooks/stop.py        ← Stop hook 脚本
│   └── settings.local.json  ← 注册 hook
├── logs/                    ← hook 写运行日志
└── reports/                 ← agent 写每小时小结
```

### 2. 你 review task.md

必要时改它——这是 agent 的唯一任务契约。

### 3. 启动

```powershell
# PowerShell（Windows）
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

然后去干别的事。

## 监控

另开一个窗口看 hook 日志：

```powershell
Get-Content <任务目录>\logs\stop-hook.log -Wait -Tail 20
```

每行形如 `[2026-04-20T03:14:00+08:00] CONTINUE: elapsed=7890s remaining=9h48m block=42`，反映循环健康度——如果 block 计数在几分钟内飙到几十，说明任务设计太空洞，agent 在空转。

## 紧急放行

让下一次 Stop hook 立刻结束 agent（不需要 Ctrl+C，让它走完最后一轮）：

```bash
python -c "import time; open(r'<任务目录>/exp-start','w').write(str(int(time.time())-13*3600))"
```

## 文件结构

```
marathon.skill/
├── SKILL.md                          触发规则 + 给 Claude 的执行说明
├── README.md                         你正在看的
└── template/
    ├── task.md.tmpl                  任务契约模板
    ├── CLAUDE.md.tmpl                agent 工作协议（固定骨架）
    ├── stop.py.tmpl                  Stop hook 脚本
    └── settings.local.json.tmpl      Stop hook 注册
```

## 工作原理简述

```
Claude 完成一轮任务，准备退出
      ↓
Claude Code 读 .claude/settings.local.json，触发 Stop hook
      ↓
启动 python .claude/hooks/stop.py
      ↓
脚本读 exp-start 算 elapsed、读 exp-block-count
      ↓
   elapsed >= 12h? ──是──> exit 0（放行，Claude 结束）
      ↓否
   block_count >= 5000? ──是──> exit 0（兜底放行）
      ↓否
   stdout 输出 {"decision":"block","reason":"继续推进..."}
      ↓
Claude 看到 block → 不退出 → 读 reason 作为新一轮指令 → 继续 agent loop
```

Stop hook 的 JSON schema 只接受 `decision` 和 `reason`（以及可选 `continue` / `stopReason` / `suppressOutput` / `systemMessage`）。不要往里塞 `hookSpecificOutput`——那是 PreToolUse / UserPromptSubmit / PostToolUse 的字段，Stop 事件会拒绝。

## 兼容性

- **Claude Code CLI**（测试过 2.1.x）
- **Windows + PowerShell / Git Bash**：hook 必须用 Python 写，因为 Windows 下 `bash` 可能指向 WSL 或 MSYS 内部 bash（不能被 `CreateProcess` 直接调起）
- **macOS / Linux**：同样用 Python 脚本，跨平台一致

## 已知坑

- PowerShell 的 `Get-Date -UFormat %s` 在 Windows 时区下生成的时间戳有偏，会让 `elapsed` 算出负数。**始终用 `python -c "import time; ..."` 或 `date +%s`**（Git Bash 下）生成 `exp-start`
- `.ps1` 文件含中文必须 UTF-8 with BOM，否则 PowerShell 按 ANSI 解读导致语法错误
- `--dangerously-skip-permissions` 是无人值守的前提；白名单方案半夜触发新权限请求会卡死

## 致谢

- [loopndroll](https://github.com/lnikell/loopndroll) —— Codex CLI 的同类实现，给了核心机制的灵感
- [Claude Code Hooks 文档](https://code.claude.com/docs/en/hooks.md)

## License

MIT
