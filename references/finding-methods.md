# 发现方法：找不一致，不是扫描

> 这份文件是 skill 的**核心方法论**，不是症状清单。
> 症状清单（`commands.md`）告诉你要找什么；这份告诉你怎么找到它。

---

## 0. 为什么"扫描"不够

一次真实执行的复盘：8 条发现里，**7 条来自"对照"，0 条来自"扫描"**。

| 扫描能给你的 | 对照才能给你的 |
|---|---|
| 大文件排行 | 哪两份实现其实是一回事 |
| 复杂度数字 | 哪个名字跟它做的事不符 |
| 有没有空 catch | 这条 catch 是"故意吞"还是"忘了处理" |
| 依赖清单 | 哪个依赖早就没人用了但文档还写着 |

**扫描得到的是"数量"，对照得到的是"矛盾"。而屎山的本质是矛盾，不是数量。**

---

## 1. 七种实测有效的对照

这七种都有真实命中记录，按"产出/成本"排序。

### F1 工具-源码交叉（lint 给坐标，人给性质）

**做什么**：跑 lint / tsc / 仓库自带检查 → **每一条报错都必须打开源码读一遍** → 判断它是真问题还是误报。

**为什么不能只看报告**：工具只会说"第 26 行 return 在 hook 之前"，不会说"这条路径现在走不到"。性质判断必须靠读。

**操作**：
```bash
<仓库的 lint 命令>            # 例：npx oxlint / npx eslint / ruff check
<仓库的 typecheck>            # 例：npx tsc -b --noEmit
```

**输出长这样**：
```
WaitingCard.tsx:26  return null 在 7 个 hook 之前 → hooks 数量可变
  读源码：该组件在 goal 为空时提前 return
  往上查：TodayScreen.tsx:139 的 waitingGoals 已过滤空数组
  → 结论：潜伏（触发条件是"编辑已有节点到全空"）
```

**常见误报**：lint 的 `include` 范围大于 `tsc`，会出现"不参与构建的文件"报错。单独归类，别混进主结论。

---

### F2 单一事实源对账（同一件事有几个副本）

**这是产出最高的一种。** 凡是"应该只有一个真相"的东西，都可能有三个副本，而副本总会漂移。

**该对账的东西**：版本号 · 依赖清单 · 术语命名 · 接口定义 · 默认值 · 错误码 · 超时时长 · 数据库表名。

**操作**：
```bash
# ① 找所有版本号出现的地方
rg -n "\b\d+\.\d+\.\d+\b" -g '*.md' -g '*.json' -g '*.gradle' -g '*.toml' -g '!node_modules' | head -30

# ② 确定权威源（通常是这些之一）
#    package.json / build.gradle / Cargo.toml / pyproject.toml / version.ts

# ③ 其余全部算副本，逐个核对
```

**输出长这样**：
```
版本号：权威 = build.gradle:19-20 (0.1.59) + release/ 产物名
  AGENTS.md:8   写 0.1.8 / vc10      → 漂移
  README.md:50  写 0.1.31            → 漂移
  backlog.md:3  写 0.1.37            → 漂移
```

**判据**：**实际产物 > 声明文件 > 文档。** 冲突时改文档，不改事实。

---

### F3 文档-事实交叉（文档是过期承诺）

**做什么**：把文档里提到的每一个**可验证的东西**拿去核。

该核的：命令（`pnpm test:x`）· 路径（`src/lib/x.ts`）· 依赖名 · 版本号 · 环境变量 · 已实现的功能。

**操作**：
```bash
# 文档里提到的脚本，是否真实存在
rg -o -N "(pnpm|npm run) [a-z:._-]+" -g '*.md' | sed 's/.* //' | sort -u
# 然后对照 package.json 的 scripts 键

# 文档里提到的路径，是否还存在
for p in $(rg -o -N "src/[a-zA-Z0-9_/.-]+" -g '*.md' | sort -u); do
  [ -e "$p" ] || echo "STALE PATH: $p"
done

# 文档里提到的依赖，是否还在 package.json
rg -o -N "\b[a-z@][a-z0-9._/-]{2,30}\b" -g '*.md' | sort -u | head -50
```

**输出长这样**：
```
findings.md:41  "校验用 zod"  →  代码里 0 处 zod，实际是手写 normalizePlan
                                   → 文档是过期承诺，会误导下一个 AI 去装 zod
```

**为什么重要**：**文档骗 AI 比骗人严重**。人知道文档可能旧；AI 会照做。

---

### F4 近名函数扫描（不只是同名）

同名能机器抓，**不同名但同义只能靠"名字相近"这个弱信号去捞**。

**操作**：
```bash
# ① 同名（精确）
rg -n "^(export )?(async )?function \w+" -g '*.ts' | awk -F: '{print $3}' | sort | uniq -d

# ② 近名：按前缀聚簇（粗但有效）
rg -o -N "^(export )?(async )?(function|const) \w+" -g '*.ts' -g '*.tsx' \
  | awk '{print $NF}' | awk '{k=substr($0,1,6); a[k]=a[k]" "$0} END{for(k in a) if(split(a[k],x," ")>1) print k" →"a[k]}'

# ③ 语义近名：days / until / from / diff / span 这类词根
rg -o -N "\w*(Days?|Until|From|Diff|Span|Offset)\w*" -g '*.ts' | sort -u
```

**输出长这样**：
```
date.ts:55  daysFromToday
date.ts:60  daysUntil
  → 函数体逐字相同，两名一义
```

**误报**：`daysFromToday`（负数方向）和 `daysUntil`（正数方向）在别的项目里可能真有语义差别。**必须逐字比对 + 看调用点再判**。

---

### F5 同文件相邻对照（唯一能抓"不同名同义"的可靠手段）

**做什么**：打开一个大文件，**自上而下读，把函数体结构相同的每一对记下来**。

这就是 `daysFromToday` / `daysUntil` 被抓到的原因——它们挨着。

**操作**：没有命令，就是读。但可以筛出"该读哪些文件"：
```bash
# 函数最多的文件（同一个文件里的重复概率最高）
rg -c "^(export )?(async )?function \w+" -g '*.ts' | sort -t: -k2 -rn | head -10
```

**判据**：两个函数如果**语句序列一致、只有变量名不同**，就高度可疑。这时做 10.2 的等价性验证，能验就合。

---

### F6 调用方追溯（判"活的"还是"潜伏的"）

**任何看起来有问题的地方，先往上问一句：谁会走到这里？**

**操作**：
```bash
# 谁引用了这个模块 / 这个导出
rg -n "from ['\"].*<module>['\"]|require\(['\"].*<module>" -g '!node_modules'

# 谁渲染了这个组件 / 谁调用了这个函数
rg -n "<ComponentName|\.functionName\(" -g '*.tsx' -g '*.ts'
```

**输出长这样**：
```
组件内 hook 顺序有 bug（会炸）
  → 查调用方：TodayScreen.tsx:139 用 .filter(g => g.waitingNodes?.length) 过滤了
  → 结论：潜伏。当前路径走不到，但"编辑到全空"会走到
```

**这一步决定了严重度**。同一段代码，"活的"要立刻修，"潜伏的"可以排期。报告里必须写清是哪一种。

---

### F7 项目自有记忆（免费的线索库）

**动任何代码之前，先读项目自己写的账**：

```
FINDINGS.md  findings.md  backlog.md  ROADMAP.md  TODO.md
AGENTS.md  README.md  docs/  CHANGELOG.md
```

**为什么这是最快的**：`react-wheel-picker` 白装那条，项目自己的 `backlog.md:91` 就记着"自写 WheelPad 后忘了删"。**读别人的账比自己扫快 10 倍。**

**判据**：项目记忆里记着的东西，可信度高于你的推断；但**高于不了代码事实**。记着"已修"而代码没修的，本身就是一条发现。

---

## 2. 七种拓展（同类方法，尚未实测）

以下都是"对照"的变体，逻辑成立但没有真实命中记录。用的时候标"推测性检查"。

### F8 字面量对账（同一个魔法值散在多处）

```bash
# 出现次数最多的短字符串字面量（默认值、状态名、错误码）
rg -o -N "\"[a-z_]{3,20}\"" -g '*.ts' | sort | uniq -c | sort -rn | head -20

# 硬编码数字
rg -o -N "\b(3000|5000|10000|86400|30|60|1000)\b" -g '*.ts' | sort | uniq -c | sort -rn | head
```
**看什么**：同一个 `3000` 在 5 个文件里当超时 → 该抽常量。两个地方的超时不一致 → 可能是 bug。

### F9 环境变量对账（用了没声明 / 声明了没用）

```bash
rg -o -N "process\.env\.\w+" -g '*.ts' | sed 's/process.env.//' | sort -u > /tmp/used
grep -oE "^[A-Z_]+" .env.example 2>/dev/null | sort -u > /tmp/declared
comm -23 /tmp/used /tmp/declared   # 代码用了，示例里没写 → 新人部署必踩
comm -13 /tmp/used /tmp/declared   # 示例里有，代码不用了 → 过期配置
```

### F10 入口对账（声明 vs 实际注册）

**看什么**：路由表里声明了但组件不存在的、组件存在但没挂路由的、`package.json` 的 `main` 指向一个不存在的文件。

```bash
# 路由声明
rg -n "path:\s*['\"]/" -g '*.ts' -g '*.tsx' | head -30
# 逐个核对 import 的组件是否存在
```

### F11 测试名-断言对照（测试说了但没测）

```bash
# 测试名承诺了错误路径，断言里却没有 toThrow / rejects
rg -n -A6 "it\(['\"].*(reject|throw|error|invalid|fail)" -g '*.test.*' -g '*.spec.*' \
  | grep -v "toThrow\|rejects\|toThrowError"
```

**看什么**：测试名是唯一的"规格文档"，名字和断言不符就是假测试。

### F12 类型-运行时对照（类型说了谎）

**看什么**：
- 类型声明 `data.user.id: string`，但接口实际可能不返回 `user`
- `as SomeType` 之后，字段真的都在吗
- 可选链 `a?.b?.c` 的深度 >3，说明数据形状没定清

```bash
rg -n "\?\.\w+\?\.\w+\?\." -g '*.ts' -g '*.tsx' | head
```

### F13 双环境对账（dev 能跑，prod 不跑）

**看什么**：`.env` vs `.env.production`、`vite.config` 里的 `mode` 分支、CI 与本地脚本是否同一套命令。

```bash
ls -a | grep -E "^\.env"
diff <(sort .env.example 2>/dev/null) <(sort .env.production 2>/dev/null) | head -20
```

### F14 跨层形状对账（A 的输出 vs B 的输入）

**看什么**：后端返回的字段名 / 前端类型声明 / 数据库列名，三者是否一致。这个只能读，但读法固定：

```
1. 找接口定义（类型声明或 zod schema）
2. 找消费方（谁读这个字段）
3. 找生产方（谁写这个字段）
4. 三者名字对不上 → 中间一定有 cast 或默认值在遮掩
```

---

## 3. 顺手对账三件套（15 分钟，产出最高）

时间紧、只想跑一遍时，就跑这三个：

| # | 对账 | 命令 | 典型战果 |
|---|---|---|---|
| 1 | **版本/依赖对账**（F2） | 见 F2 | 文档写的版本全是旧的；白装的依赖 |
| 2 | **文档-事实对账**（F3） | 见 F3 | 文档说用 X 校验，实际手写；提到已删的文件 |
| 3 | **近名扫描 + 相邻读**（F4+F5） | 见 F4 | 两个函数一义；人肉同步的注释 |

**这三件套在这类"看起来还行"的项目上，命中率远高于跑全套 lint。**

---

## 4. 组合套路

### 套路 A：接手一个陌生模块（30 分钟）

```
F7 读项目记忆 → F1 跑 lint → F5 读最大的文件 → F6 追调用方
```

### 套路 B：怀疑有重复实现

```
F4 近名扫描 → F5 相邻读 → 10.2 等价性验证 → 合
```

### 套路 C：文档不可信（AI 写过的项目常见）

```
F3 逐项核文档 → F2 版本/依赖对账 → 把过期项记进 FINDINGS
```

### 套路 D：怀疑"假装修好了"

```
git diff HEAD → 三条 diff 纯度审计（见 SKILL.md §9.4）
→ F11 测试名-断言对照
→ F6 看改动的函数有没有调用方覆盖
```

---

## 5. 一条纪律

**对照得出的结论，必须写清"对哪两份"。**

错：「这里重复了」
对：「`date.ts:55` 与 `date.ts:60` 函数体逐字相同（已逐行比对 7 行）」

错：「文档过期了」
对：「`findings.md:41` 声称用 zod 校验；`git grep -E "z\.object|safeParse" -- src` 结果为空；实际校验在 `schema.ts:240` 手写」

**没有"对哪两份"的结论，不是发现，是感觉。**
