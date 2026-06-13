<p align="right"><a href="./README.md"><b>English</b></a></p>

# Fable Skill

> 给编程 Agent 的一套工程**思维方式**。它教模型先想清楚代码会**怎么坏**，再去想它怎么跑通——并专门抓那些**不出声**的失败。

Fable 是一个可移植的 **Agent Skill**：一份 `SKILL.md`，加载进你的 AI 编程 Agent（Claude Code、Codex，或任何能读 skill 的 Agent）即可。它不是库，也不是 linter——它是一种思考方式。

核心理念：**最坏的 bug 不是崩溃的那个，而是安静的那个。** 静默失败看起来和成功一模一样，却在悄悄破坏数据、丢掉某个行为、或在高负载下劣化。Fable 训练你的 Agent 在这类局面上线前就**认出**它们。

---

## 它真的有用吗？一个消融实验（Ablation Study）

我们用**同一个模型**（Claude **Opus 4.8**）评审**同一份代码**——一个植入了 **8 个隐性缺陷**的 Python 服务（都是不会崩、快速扫一眼会直接错过的那种）。一组不挂 skill，另一组挂载 Fable。每组各跑两次。

| | 命中缺陷（满分 8） | 是否抓到**最安静**的失败¹ | 是否按「怎么失败」归类 | 额外发现的隐性失败模式 |
|---|:---:|:---:|:---:|:---:|
| **Opus 4.8 — 不挂 skill** | 7、7 &nbsp;(均值 **7.0**) | ✗ 两次都漏 | 否 | — |
| **Opus 4.8 + Fable** | 8、8 &nbsp;(均值 **8.0**) | ✓ 两次都中 | 是 | +2 |

¹ 两次「不挂 skill」**都漏掉的同一个**缺陷：模型输出在触顶 token 上限后被截断，却被当成**完整结果**继续往下游传——正是 Fable 所说的*「把劣化结果当完整结果下传」*。挂载 Fable 后两次都抓到了，并提醒评审者：信任输出前先检查 finish reason。

**结论：** 在生产里，把你击垮的很少是那个响亮的 bug，而是最后那个没人发现的、最安静的。强模型本来就能自己抓到响亮的；Fable 的增量,恰恰是**最后那一个**。而且它把每条发现都按**怎么失败**来归类，而不只是停在**失败了**。挂载 Fable 的两次还额外揪出了裸模型完全没提的失败模式（例如：内存限流器在每次部署时静默清零；某个返回值无法区分「不存在」和「出错」）。

<details>
<summary>实验方法与完整测试代码（可复现）</summary>

两组收到完全相同的提示：*「你是一名资深工程师，在一个服务上线生产前评审这份 Python 模块。该服务跑在负载均衡后、有多个 worker 进程、会连续运行数周。列出你发现的每一个问题及其原因。」* 实验组额外把 Fable 的 `SKILL.md` 作为评审思维方式一起给出。模型相同、采样默认相同，按固定的 8 个植入缺陷评分卡打分。

```python
import json, sqlite3

_login_attempts = {}  # ip -> attempt count

def allow_login(ip):                      # R1 进程内计数器（跨 worker 失效）
    n = _login_attempts.get(ip, 0)        # R2 字典永不清理 -> 无界增长
    if n > 100:
        return False
    _login_attempts[ip] = n + 1
    return True

async def get_report(user_id):            # R3 在事件循环里阻塞 sqlite
    conn = sqlite3.connect("app.db")      # R4 连接从不关闭 -> 泄漏
    row = conn.execute("SELECT body FROM reports WHERE uid=?", (user_id,)).fetchone()
    return row[0] if row else None

def dedup(new_papers, past_reports):      # R5 静默空转：只看第一份报告，
    seen = past_reports[0]["papers"] if past_reports else []   #   且比的是整个 dict 而非 id
    return [p for p in new_papers if p not in seen]

def save_profile(state):                  # R6 非原子写 -> 崩溃时撕裂/损坏
    with open("data/profile.json", "w") as f:
        json.dump(state, f)

def summarize(text):                      # R7 无上限重试循环
    while True:
        out = call_llm(text, max_tokens=500)   # R8 截断结果被当成完整结果
        if out["content"]:
            return out["content"]
```

</details>

---

## 安装

对你的编程 Agent 说：

> **从 https://github.com/hashiruu/fable-skill 安装 Fable skill**

有能力的 Agent 会自动把仓库克隆进它的 skills 目录。手动安装的话——克隆到对应目录，让 Agent 自动发现 `SKILL.md`：

```bash
# Claude Code
git clone https://github.com/hashiruu/fable-skill.git ~/.claude/skills/fable

# Codex / 读取 ~/.agents/skills 的 Agent
git clone https://github.com/hashiruu/fable-skill.git ~/.agents/skills/fable
```

然后开一个新会话——只要你在写代码、评审、调试或运维，skill 就会激活。没有 skills 目录？把 [`SKILL.md`](./SKILL.md) 的内容直接粘进系统提示词即可。

---

## 里面有什么

九种「看起来没问题、其实有问题」的局面，每一种都配一段要内化的第一人称推理：

- **静默空转** —— 代码跑了、没报错、却什么都没做。
- **偶然的安全** —— 一个性质只是碰巧成立；在这个巧合失效前把它变成强制不变量。
- **一个调用者就能冻住的共享资源** —— 共享循环里一次阻塞操作会冻住所有人。
- **拓扑盲的正确性** —— 「当有八个我同时跑时，这还成立吗？」
- **重构悄悄丢掉了隐式行为** —— 一个值同时承担两种含义。
- **过于宽容的解释器** —— 宽松解析悄悄放大范围、污染数据而不报错。
- **已经上线的污染** —— 修代码只是一半，还要修它已经弄脏的数据，并靠计数去验证。
- **把劣化结果当完整结果下传** —— 拒绝把截断/部分结果当成已完成。
- **诱人的错误修法** —— 当一个修复显得太轻松，问问它对根因做了什么。

外加一套面向「长时间运行 / 多进程」系统的运维心法（选举而非假设单例、用持久快照而非内存共享状态、凡是重复的都要设上限、原子写、向安全态失败），以及 Fable 在动笔前、调试时、上线前问自己的那几个问题。

---

## 许可证

[MIT](./LICENSE)。随便用、随便 fork、随便上线。
