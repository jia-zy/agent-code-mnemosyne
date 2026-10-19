
---
name: "复盘助手"
description: "强制自动调用的代码变更与问题复盘库：涉及代码修改、排障、报错、Git 操作时自动触发。先查库→立改造契约与检查点（防漂移、可回退）→分步改并验证→每次改动完成自动入库并归档（可追溯），无需用户提醒。"
---

# 复盘助手

超级轻量极简版的"代码与问题向量库"：用一个 SQLite 文件沉淀本项目的代码变更、设计决策、解决过的问题和 bug。**本 skill 为强制自动调用，助手在以下场景必须自动执行，不需要等用户说"记录一下"。**

本 skill 同时负责四条工程纪律，缺一不可：

1. **不跑偏**：改动前立"改造契约"，改动后做一致性自检，防止代码风格漂移、做出来的东西和用户要求不一致（§十）。
2. **可回退**：动手前建检查点，小步改+每步验证，出问题按标准流程回退，而不是乱试或回退不干净（§十一）。
3. **可追溯**：每次改动留下"检查点 + 验证证据 + 复盘记录"三件套，出问题能复盘（§十二）。
4. **可检索**：所有结论落到 `.review/review.db`，下次开工先查库复用历史方案（§三~§六）。

## 一、库位置与表结构

- 库文件：当前项目根目录下 `.review/review.db`（不存在则自动创建）
- 配套文件：
  - `.review/checkpoints/<yyyyMMdd_HHmmss>/`：改动还原点（见 §十一）
  - `.review/changelog.md`：人类可读的改动时间线（见 §十二）
  - `.review/git_standards.md`：项目级 Git/代码规范清单（见 §九）
- 唯一数据表 `problems`：

```sql
CREATE TABLE IF NOT EXISTS problems (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,            -- 变更/问题一句话（如 "DeepSeekTranslateClient 新增 classify 多标签方法" 或 "ESReader-04 column不能为空"）
  category TEXT,                  -- 类别：代码/配置/环境/插件/DataX/ES/Kafka/设计决策/排障/Git规范/回退...
  problem TEXT,                   -- 变更背景/问题现象 + 根因（精简）
  solution TEXT,                  -- 变更内容/解决方案（精简步骤）
  keywords TEXT,                  -- 检索关键词，逗号分隔（文件名/函数名/方法名/报错码/组件名/关键配置项）
  files TEXT,                     -- 相关文件路径（逗号分隔）
  checkpoint TEXT,                -- 关联检查点目录，如 .review/checkpoints/20260908_180000
  verify TEXT,                    -- 验证证据：实际执行的命令 + 结果（如 "pytest -q → 67 passed"）；未验证必须写明"未验证"
  status TEXT DEFAULT '已完成',    -- 已完成 / 进行中 / 已回退
  created_at TEXT DEFAULT (datetime('now', 'localtime'))
);
```

**旧库自动升级（幂等，写入前必做）**：老库只有前 7 列，写入前先补列，缺哪列补哪列：

```bash
python - <<'PY'
import sqlite3
con = sqlite3.connect(".review/review.db")
con.execute("""CREATE TABLE IF NOT EXISTS problems (
  id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT NOT NULL, category TEXT,
  problem TEXT, solution TEXT, keywords TEXT, files TEXT,
  checkpoint TEXT, verify TEXT, status TEXT DEFAULT '已完成',
  created_at TEXT DEFAULT (datetime('now', 'localtime')))""")
have = {r[1] for r in con.execute("PRAGMA table_info(problems)")}
for col, ddl in [("checkpoint", "TEXT"), ("verify", "TEXT"), ("status", "TEXT DEFAULT '已完成'")]:
    if col not in have:
        con.execute(f"ALTER TABLE problems ADD COLUMN {col} {ddl}")
con.commit(); con.close()
print("表结构已就绪")
PY
```

**status 语义**：`进行中`（还在改/未验证）→ `已完成`（已验证通过）→ `已回退`（改动被撤销，保留失败路径便于复盘）。

## 二、强制自动触发场景

以下场景助手必须主动调用本 skill，不需要用户提醒：

1. **开始修改代码前**：先查 `.review/review.db` 是否有相关历史记录。
2. **改动前**：写"改造契约"并建立检查点（§十、§十一）。
3. **每完成一个可验证的改动（一个函数/方法/文件/配置/功能点）**：**无需用户提醒，立即自动入库**——写入 `problems` 记录 **并** 追加 `.review/changelog.md` 一行；不等"攒一批再记"。
4. **排障/修复/报错处理结束后**：立即自动入库并归档。
5. **危险操作前**（删除文件、批量重命名、依赖变更、数据库迁移、全局配置、`-f` 类破坏性命令）：先列影响面、建检查点、写下回退命令（§十一）。
6. **改动完成后**：做一致性自检 + 记录验证证据，并**自动完成入库与归档**（§十、§十二）。
7. **回退之后**：立即记录一条 `category=回退` 的记录（§十一）。
8. **Git push 失败（REJECT 等）时**：立即记录并更新 `.review/git_standards.md`。
9. **每次改动完成都必须归档（自动执行，无需用户提醒）**：
   - 改动级 → 追加 `.review/changelog.md` 时间线一行
   - 任务/子任务级 → 追加 `.review/session_notes_yyyyMMdd.md` 归档片段
   - 上下文接近 60% 或对话超 30 轮 → 全量压缩归档（§七）
10. **任务收尾前**：强制自查是否漏写复盘记录、漏记检查点/验证证据、漏归档。

## 三、强制流程（不可跳过）

以下步骤是**强制流程**，凡是涉及代码修改、排障、修复、报错处理的任务都必须执行，不允许跳过：

1. **动手前先查库**：开始修改代码或处理新问题时，**第一步强制检索**复盘库。查库要写出实际查询结果（命中几条、哪几条），不可口头声称"已查过"。命中则先看历史方案，根因一致直接复用。
2. **动手前立契约、建检查点**：先写"改造契约"（需求复述 + 验收标准 + 改动范围 + 遵循的既有约定），再建检查点目录并写入回退命令（§十、§十一）。**禁止无契约、无检查点直接改代码。**
3. **小步改、每步验证**：一次只改一个文件/函数，改完立即验证（语法检查 / ruff / 单测 / 启动冒烟）。验证通过才进入下一步；同一个点连续失败 2 次必须停下来向用户汇报，不许反复硬试（§十一）。
4. **改动完成即自动入库（无需用户提醒）**：
   - 每完成一个**可验证的改动**（一个函数、一个方法、一个文件、一段配置、一个功能点），立即写入 `problems`，字段齐全：`category/problem/solution/keywords/files/checkpoint/verify/status`。
   - **排障/修复/报错处理**解决后同样立即写入。
   - 若在库内查到了相关历史，仍按最终根因/最终变更再写一条新记录，不要改旧记录。
   - 缺 `verify`（验证证据）时 `status` 只能标 `进行中`；写入后输出 `已写入复盘库 #<id>: <title>` 确认。
   - **禁止攒着不写**：不许等用户说"记录一下"，也不许拖到会话结束再一次性补一堆。
5. **改动完成即自动归档（无需用户提醒）**：
   - **改动级**：向 `.review/changelog.md` 追加一行（时间 / 任务 / 文件 / 检查点 / 验证 / 结果）。
   - **任务或子任务级**：向 `.review/session_notes_yyyyMMdd.md` 追加归档片段（关键决策 / 改动文件 / 检查点与回退 / 待跟进）。
   - **上下文接近 60% 或对话超 30 轮**：执行 §七 全量压缩归档。
   - 归档完成后向用户一行说明归档位置。
6. **任务收尾强制自查**：每次会话/任务收尾前，自查本次是否做过代码变更或解决过问题：
   - 做过变更/解决过 → **必须已入库且已归档**（漏写立即补录），且 `status` 与真实验证状态一致
   - 还在排障/变更中 → 记录一条 `status=进行中` 的记录，解决后再补全
   - 完全没涉及代码/排障 → 无需写入

## 四、写入步骤

1. 用 Python 内置 sqlite3（`python3` 或 `python` 均可），库路径 `.review/review.db`，先建表。
2. 按上面表结构 INSERT 一条记录，字段务必精简（几行到十几行内说清）。
3. 记录后输出一行确认，如 `已写入复盘库 #id: <title>`。

参考片段（在项目根目录执行）：

```bash
python3 - <<'PY'
import sqlite3, os
db = ".review/review.db"
os.makedirs(os.path.dirname(db), exist_ok=True)
con = sqlite3.connect(db)
con.execute("""CREATE TABLE IF NOT EXISTS problems (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL, category TEXT, problem TEXT,
  solution TEXT, keywords TEXT, files TEXT,
  created_at TEXT DEFAULT (datetime('now', 'localtime')))""")
con.execute(
  "INSERT INTO problems (title, category, problem, solution, keywords, files) VALUES (?,?,?,?,?,?)",
  ("DeepSeekTranslateClient 新增 classify 多标签方法", "代码",
   "英文数据不再翻译，改为用大模型打 7 类能源标签，一条数据可能多个标签",
   "新增 classify/classifyOnce/parseCategories 方法，category 输出为数组；失败回退中文关键词分类",
   "DeepSeekTranslateClient,classify,多标签,能源分类,EnglishTranslateToEsJob",
   "src/main/java/org/example/utils/DeepSeekTranslateClient.java")
)
con.commit(); con.close()
print("已写入复盘库")
PY
```

## 五、检索步骤

1. 从当前问题/变更提取关键词：函数名、方法名、文件名、报错码、报错关键字、组件名、关键配置项。
1-1. 关键词扩展：基于当前问题、现象、报错，生成扩展检索词集合，包含：等价报错描述、业务别名、同义表述、不同人可能使用的措辞；不要脱离实际业务，不要生成无关关键词。
1-2. 将原始关键词 + 扩展得到的全部关键词集合，一并投入检索，拿到数据库返回的候选结果列表。
2. 用 LIKE 模糊匹配检索（多个关键词逐条查，合并结果）：

```bash
python3 - <<'PY'
import sqlite3
con = sqlite3.connect(".review/review.db")
con.row_factory = sqlite3.Row
kw = "column"  # 换成实际关键词
for r in con.execute(
    "SELECT * FROM problems WHERE title LIKE ? OR keywords LIKE ? OR problem LIKE ? OR solution LIKE ?",
    (f"%{kw}%", f"%{kw}%", f"%{kw}%", f"%{kw}%")):
    print(f"#{r['id']} [{r['category']}] {r['title']}\n 问题: {r['problem']}\n 方案: {r['solution']}\n 相关文件: {r['files']}\n 检查点: {r['checkpoint']}\n 验证: {r['verify']}\n 状态: {r['status']}\n")
con.close()
PY
```

3. 对数据库返回的全部候选记录执行相关性重排过滤：根据当前实际问题，逐条判断每条候选记录的真实相关程度，过滤掉字面匹配但业务无关的误命中条目；只保留真正相关的记录再输出，禁止直接把全部原始候选直接返回。
4. 命中则把历史方案作为排查/变更起点；若按历史排查后发现根因不同/方案有更新，解决后**再写一条新记录**，不要改旧记录。**命中 `状态=已回退` 的记录说明那条路走不通，不要照抄。**
5. 未命中则正常处理，解决/变更后写入。


## 六、写入规范（精简原则）

- 一条记录 = 1 个变更/问题 + 1 个根因/背景 + 1 套最终方案/变更内容，控制在精简描述内。
- `keywords` 是检索命中的关键，务必包含：函数名、方法名、文件名、报错码/错误关键字、插件或组件名、关键配置项名。
- 同类问题/同类变更（同一文件不同函数）可作为新记录，keywords 保持一致便于归并检索。
- **`checkpoint`、`verify`、`status` 三个字段必填**：`checkpoint` 填本次的检查点目录；`verify` 填真实执行的验证命令与结果（没验证就写"未验证"并把 `status` 置为 `进行中`）；`status` 只能是 `已完成`/`进行中`/`已回退`。
- **禁止谄媚式记录**：绝不写"问题已全部解决"却没验证证据；失败/回退同样要记，且比成功记录更有价值。

## 七、长对话自动压缩与归档（强制）

当单次会话出现以下任一情况时，必须主动把上下文压缩精简并归档到 `.review/session_notes_yyyyMMdd.md`（项目根目录），避免上下文爆炸：

1. 当前助手上下文 token 占用接近上限（约 60% 以上）。
2. 对话轮数超过 30 轮或连续处理了 3 个以上独立子任务。
3. 用户在上下文中明确要求"总结一下"、"精简一下"、"太长了我看不过来"。

压缩格式（追加到文件末尾）：

```markdown
## 2026-09-01 会话归档片段

### 关键决策
- xxx
- xxx

### 修改过的文件/函数
- `src/.../Xxx.java`：修改了 `methodA`，原因是...
- `src/.../Yyy.java`：新增 `methodB`，作用是...

### 检查点与回退
- 检查点：`.review/checkpoints/20260901_143000`
- 回退情况：无 / 已回退 <文件>，原因是 ...（回退后验证：<命令 → 结果>）

### 待跟进
- xxx
```

归档后向用户说明："上下文已压缩归档到 `.review/session_notes_20260901.md`，后续可继续基于精简摘要推进。"

## 八、Git 推送/拉取强制确认

任何涉及 Git 仓库写操作（`git push`、`git pull`、`git merge`、`git rebase` 等可能改变本地/远程仓库状态的操作）必须**先经用户明确同意**才能执行，不允许自动执行。具体流程：

1. **推送前**：
   - 必须先运行 `git status`、`git diff --cached`（或等效命令）列出**待推送文件清单**。
   - 向用户展示：变更文件列表、每个文件的修改类型（新增/修改/删除/重命名）、commit message。
   - **必须等用户回复"同意推送"或"可以"之后**，才能执行 `git push`。
   - 用户未明确同意时，只停留在本地 commit 或 diff 展示阶段，不推送。

2. **拉取前**：
   - 如果用户要求更新代码，先说明当前本地是否有未提交变更、预计拉取会影响哪些文件。
   - **必须等用户明确同意后才能执行 `git pull`**。

3. **禁止行为**：
   - 不得使用 `git push -f`、`git push --force-with-lease`、`git reset --hard`、`git checkout .`、`git clean -f` 等破坏性命令，除非用户明确书面要求。
   - 不得以"我先 push 试试"、"应该没问题"等理由绕过确认。

## 九、Git 规范错误预防与复盘（强制）

为避免反复犯同样的 git 规范错误，**规范必须在写代码阶段就内化**，而不是等到 push 失败再修。

### 1. 编码阶段强制读取规范清单

每次开始写代码前，**必须先读取** `.review/git_standards.md`（项目级 Git/代码规范清单）。若文件不存在，则按本 skill 内置的初始规范执行，并在首次遇到规范错误后创建/更新该文件。

`.review/git_standards.md` 初始内容应包含（随错误类型动态追加）：

```markdown
# 项目代码与 Git 规范清单

## Java 编码规范
1. **禁止 Magic Value**：硬编码字符串/数字/正则必须提取为常量或配置项。
   - 反例：`text.split("[,，]")`
   - 正例：`private static final String CATEGORY_DELIMITER = "[,，]"; text.split(CATEGORY_DELIMITER);`
2. **集合必须给初始容量**：`new HashMap<>(2)`、`new ArrayList<>(10)`，避免无参构造。
3. **导入完整性**：新增代码用到的类必须显式 import，提交前 GetDiagnostics 验证无编译错误。
4. **删除未使用 import/变量**：IDE 警告必须清零。

## Git 提交规范
1. **分支名**：develop、feature-xxxx、hotfix-xxxxBUG、release-x.x。
2. **commit message**：首行说明"为什么"修改，不超过 72 字符。
3. **LF/CRLF**：出现 LF will be replaced by CRLF 警告时确认是否可接受。

## 历史错误类型（动态追加）
- REJECT 4 / magic value
- REJECT 4 / HashMap 无初始容量
```

### 2. 编码时主动规避

写代码过程中，**每写一个方法/函数/文件**，必须对照上述规范自检：

- 有硬编码字符串/数字/正则 → 立即提取常量
- 有 `new HashMap<>()` / `new ArrayList<>()` 无参构造 → 立即补初始容量
- 有新增 import → 检查是否都写上了
- 有未使用 import/变量 → 立即删除

### 3. 提交前强制自检清单

每次 `git add` / `git commit` 前必须对照以下清单自查（命中任一项必须修复后再 commit）：

- [ ] **Magic value 检查**：Java 代码中无未提取常量的硬编码字符串/数字正则。
- [ ] **集合初始容量**：`new HashMap<>()`/`new ArrayList<>()` 等已知大小时必须给初始容量。
- [ ] **导入检查**：新增代码用到的类都有 import，无编译错误。
- [ ] **LF/CRLF 警告**：出现 `"LF will be replaced by CRLF"` 警告时确认是否可接受，必要时调整 `core.autocrlf`。
- [ ] **分支名规范**：当前分支名符合仓库允许的命名模式。
- [ ] **commit message**：简洁说明"为什么"修改，不超过 72 字符首行。
- [ ] **未使用 import/变量**：删除未使用的 import 和局部变量。

### 4. Push 失败后的强制动作

一旦 `git push` 被 remote hook 拒绝（如 `REJECT 4`、pre-receive hook declined）：

1. **立即停止**：不再重试 push。
2. **记录复盘库**：把拒绝原因、触发文件、修复方案写入 `problems` 表（category 标 `Git规范`）。
3. **更新规范清单**：把本次新增的错误类型追加到 `.review/git_standards.md` 的"历史错误类型"章节。
4. **修复后再次自检**：修复完成必须重新跑一遍清单，确认无同类问题后再提交。
5. **推送前展示清单**：再次 push 前向用户展示已完成的自检项和新增注意事项。

### 5. 写入示例

```bash
python3 - <<'PY'
import sqlite3, os
db = ".review/review.db"
os.makedirs(os.path.dirname(db), exist_ok=True)
con = sqlite3.connect(db)
con.execute("""CREATE TABLE IF NOT EXISTS problems (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL, category TEXT, problem TEXT,
  solution TEXT, keywords TEXT, files TEXT,
  created_at TEXT DEFAULT (datetime('now', 'localtime')))""")
con.execute(
  "INSERT INTO problems (title, category, problem, solution, keywords, files) VALUES (?,?,?,?,?,?)",
  ("git push REJECT 4: parseCategories 含 magic value", "Git规范",
   "pre-receive hook 检测到 DeepSeekTranslateClient.parseCategories 中硬编码正则，触发 REJECT 4",
   "提取常量 CATEGORY_DELIMITER；push 前增加 magic value 自检；更新 .review/git_standards.md",
   "REJECT 4,magic value,git push,pre-receive hook,CATEGORY_DELIMITER",
   "src/main/java/org/example/utils/DeepSeekTranslateClient.java,.review/git_standards.md")
)
con.commit(); con.close()
print("已写入复盘库")
PY
```

## 十、防漂移：改造契约与风格一致性（强制）

解决的问题：改到后面越改越偏、风格前后不一致、做出来的东西跟用户要求的不一样。

### 1. 开工"四查"

动手前必须依次完成，并把结果写进契约：

| 顺序 | 查什么 | 具体动作 |
|---|---|---|
| 1 | 查复盘库 | 按 §五 检索历史方案，输出"命中 N 条：哪几条" |
| 2 | 查项目约定 | 读 `AGENTS.md`、`CLAUDE.md`、`.review/git_standards.md`、README 中与本次相关的约定 |
| 3 | 查同类实现 | 挑 1~2 个**最接近本次改动的既有文件/函数作为风格模板**，写明路径 |
| 4 | 查验收口径 | 明确"什么样算做完"，写 2~4 条**可验证**的验收标准 |

### 2. 改造契约（写入检查点目录 `plan.md`，并先对用户复述）

```
## 需求复述（我的理解）
- <用 1~3 条复述用户要求，若与用户原话有偏差必须先确认>
## 验收标准（可验证）
- [ ] <例如：AI 回答不再出现 [来源N] 字样>
## 改动范围
- 会改：<文件/函数清单>
- 不会改（本次不碰）：<明确排除项，防止顺手重构>
## 遵循的既有约定（风格模板）
- 命名/日志/错误处理/注释语言/依赖：以上述模板文件为准
## 分步计划
1. <一次一步，每步可独立验证>
## 回退命令
- <预先写好的回退命令，见 §十一>
```

### 3. 风格冻结红线（默认禁止，除非用户明确要求）

- 不引入**新依赖、新目录结构、新命名风格、新日志格式、新配置项命名**。
- 不顺手重构、不格式化无关代码、不重排 import、不改动与本次需求无关的文件。
- 新增代码必须**模仿第 3 查选定的模板文件**：同样的分层、同样的异常处理方式、同样的注释语言与粒度。
- 不新增"平行实现"（已有工具函数能做的，不要另写一份）。
- 若确实必须偏离，**先在回复里显式说明"偏离项 + 原因 + 影响"**，得到认可再动手。

### 4. 完工一致性自检（必须输出，不得省）

逐条对照契约，输出检查结果：

```
一致性自检
- 验收标准 1：✅ 已满足（证据：<命令/截图/测试结果>）
- 验收标准 2：❌ 未满足（原因：... 建议：...）   ← 未满足的不得声称完成
- 偏离项：无 / <列出偏离与原因>
- 无关改动：无 / <列出，必须回退>
```

## 十一、安全改动与可回退（检查点机制，强制）

解决的问题：改得太快炸了不知道怎么办、回退不正确、回退不干净。

### 1. 动手前建检查点

每个任务（或每个独立子任务）在**第一次修改前**建立检查点，目录 `.review/checkpoints/<yyyyMMdd_HHmmss>/`：

```bash
python - <<'PY'
import datetime, pathlib, shutil, subprocess

ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
root = pathlib.Path(".review/checkpoints") / ts
(root / "files").mkdir(parents=True, exist_ok=True)

def git(*args):
    return subprocess.run(["git", *args], capture_output=True, text=True).stdout

# 1) 基线快照（只读命令，不改仓库状态）
(root / "status.txt").write_text(git("status", "--short"), encoding="utf-8")
(root / "before.patch").write_text(git("diff"), encoding="utf-8")

# 2) 计划与回退命令（改造契约写入 plan.md）
(root / "plan.md").write_text(
    "# 需求复述\n\n# 验收标准\n\n# 改动范围\n\n# 分步计划\n1. \n\n"
    "# 回退命令\n"
    f"- 撤销已跟踪文件改动: git apply -R .review/checkpoints/{ts}/before.patch\n"
    "- 删除本次新建文件: <对照 status.txt 逐个删除>\n"
    "- 恢复被删/大改文件: 用 files/ 内备份覆盖\n",
    encoding="utf-8",
)
print("检查点已建立:", root)
PY
```

**必须额外做备份的情况**：本次要**删除文件**或**大段重写**时，把原文件复制到 `<检查点>/files/`（`git diff` 无法恢复未跟踪文件与删除内容）。

### 2. 小步闸门（禁止大爆炸式改动）

- 一次只改一个文件/函数，改完**立即验证**：语法/诊断 → lint（如 ruff）→ 相关单测 → 必要时启动冒烟。
- 验证通过才进入下一步；**同一点连续失败 2 次立即停止**，向用户说明现状（已试什么、报什么错、建议怎么走），不许反复盲改。
- 严禁"一次改 10 个文件再一起跑测试"。

### 3. 危险操作前置闸门

删除文件、批量重命名/移动、依赖增删、数据库 schema 迁移、全局配置修改、任何 `-f`/破坏性命令，执行前必须：①列出**影响面清单**；②完成备份；③把**回退命令写在 plan.md**；④在回复里说明"我要做什么、怎么退"。用户未明确要求时，不做破坏性操作。

### 4. 标准回退流程（照做，不许即兴）

1. **定位**：找到对应检查点目录，读 `plan.md` 与 `status.txt`。
2. **回退**（按类型选，可组合）：
   - 已跟踪文件的修改 → `git apply -R .review/checkpoints/<ts>/before.patch`
   - 本次新建的文件 → 对照 `status.txt` 逐个删除
   - 被删除/大改的文件 → 用 `<ts>/files/` 里的备份覆盖回去
3. **验证恢复**：重跑第 2 步里用过的验证命令，确认回到基线（这一步不许省——"回退完不管"是最常见的二次事故）。
4. **记录**：写一条 `category=回退` 的复盘记录，`status=已回退`，写清"回退了什么、为什么、现在基线是什么"。

### 5. 禁止拿这些当回退手段

`git stash`（会混淆现状）、`git reset --hard`、`git checkout .`、`git clean -f`、`git push -f`——除用户明确书面要求外一律不得使用（与 §八 一致）。

## 十二、可追溯：改动三件套与时间线（强制）

解决的问题：改动不可追溯，出问题没法复盘。

### 1. 每次改动必须留下三件套

| 件 | 内容 | 位置 |
|---|---|---|
| 检查点 | 基线快照 + 契约 + 回退命令 | `.review/checkpoints/<ts>/` |
| 验证证据 | **实际执行的命令 + 真实结果**（如 `pytest -q → 67 passed`、`ruff check → All checks passed`） | 写入库 `verify` 字段 |
| 复盘记录 | 背景 + 方案 + 关键词 + 文件 + checkpoint + verify + status | `.review/review.db` |

**记录纪律**：没有验证证据的改动，`status` 只能是 `进行中`；不得用"应该没问题""理论上可行"等措辞充当证据；不得编造测试结果。

### 2. 人类可读时间线 `.review/changelog.md`

每次改动**追加一行**（最新在最上），与库双写：库用于关键词检索，时间线用于快速回溯"哪天动了什么"。

```markdown
# 改动时间线（最新在上）

| 时间 | 任务 | 改动文件 | 检查点 | 验证证据 | 结果 |
|---|---|---|---|---|---|
| 2026-09-08 18:00 | RAG 回答去引用 | app/ai/chains/rag.py, app/ai/prompts/rag_answer.yaml | checkpoints/20260908_175500 | pytest -q → 67 passed；真库冒烟无 [来源] 文本 | 已完成 |
```

### 3. 写入样例（含新字段）

```bash
python - <<'PY'
import os, sqlite3
db = ".review/review.db"
os.makedirs(".review", exist_ok=True)
con = sqlite3.connect(db)
con.execute("""CREATE TABLE IF NOT EXISTS problems (
  id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT NOT NULL, category TEXT,
  problem TEXT, solution TEXT, keywords TEXT, files TEXT,
  checkpoint TEXT, verify TEXT, status TEXT DEFAULT '已完成',
  created_at TEXT DEFAULT (datetime('now', 'localtime')))""")
have = {r[1] for r in con.execute("PRAGMA table_info(problems)")}
for col, ddl in [("checkpoint", "TEXT"), ("verify", "TEXT"), ("status", "TEXT DEFAULT '已完成'")]:
    if col not in have:
        con.execute(f"ALTER TABLE problems ADD COLUMN {col} {ddl}")
con.execute(
    "INSERT INTO problems (title, category, problem, solution, keywords, files, checkpoint, verify, status)"
    " VALUES (?,?,?,?,?,?,?,?,?)",
    ("RAG 回答去除来源引用", "代码",
     "用户要求回答只给答案、不展示 [来源N] 与引用卡片",
     "改 rag_answer.yaml 规则；rag.py 删引用解析与 citation 事件，citations 恒空；删 citations.py 与其单测",
     "rag_answer,rag.py,引用,citation,[来源N]",
     "app/ai/prompts/rag_answer.yaml,app/ai/chains/rag.py",
     ".review/checkpoints/20260908_175500",
     "pytest -q → 67 passed；ruff → 通过；真库冒烟回答无 [来源] 文本",
     "已完成")
)
con.commit(); con.close()
print("已写入复盘库（含 checkpoint/verify/status）")
PY
```

### 4. 检索时关注新字段

查库时除方案外，一并输出 `verify`（上次怎么验证的）、`status`（是否已回退）——**命中 `status=已回退` 的历史要特别注意，说明那条路走不通**。

### 5. 改动完成 → 自动入库 + 自动归档（闭环，无需用户提醒）

| 时机 | 自动动作 | 产出 |
|---|---|---|
| 完成一个可验证改动 | 写库 + 追加 changelog | `problems` 一条 + `.review/changelog.md` 一行 |
| 完成一个任务/子任务 | 写库 + changelog + 归档片段 | 上述 + `.review/session_notes_yyyyMMdd.md` 片段 |
| 上下文接近 60% / 对话超 30 轮 | 全量压缩归档 | session_notes 片段（含检查点与回退） |
| 发生回退 | 写库（`status=已回退`）+ changelog + 归档说明 | 失败路径留痕，避免重走 |
| 任务收尾 | 强制自查 | 结论：本次改动是否全部已入库、已归档 |

**铁律**：
- 三件事不可拆：**入库**（可检索）+ **changelog**（时间线）+ **归档**（压缩上下文）。
- 没有验证证据的记录不得标 `已完成`。
- 不允许"等用户要求"或"攒到最后"才记录——**记录是改动的一部分，不是附加动作**。
- 每条记录都必须能回答四个问题：改了什么、为什么改、怎么验证的、出问题怎么退。

## 十三、注意事项

- 库文件是普通 SQLite，无外部依赖、无需索引服务，删除 `.review` 目录即清空复盘库。
- 写入/检索都用 SQL 明文，禁止编造不存在的历史记录；库为空时正常检索即可。
- 若 `.review/review.db` 不存在但项目已有排障/变更历史，可在首个问题写入时同时把已知历史批量补录一次。
