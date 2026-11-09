# 症状清单（命令）

> 这些命令的价值主要是**排除**（跑过没发现），不是发现。
> 真正的高价值发现靠"对照"（见 `finding-methods.md`）和读代码。
> 用这些命令把"显然没有的"划掉，省下时间读该读的。

按 ripgrep 写；没有 `rg` 换 `grep -rn`。先排除 `node_modules/dist/build`。

---

## 1. 结构

```bash
# 大文件排行
git ls-files | grep -vE 'node_modules|dist/|build/' | xargs wc -l 2>/dev/null | sort -rn | head -15

# 空 catch / 只 log 的 catch
rg -n "catch\s*(\([^)]*\))?\s*\{\s*\}" -g '!node_modules' -g '!dist'

# 吞错误：catch 后直接返回默认值
rg -n -A2 "catch" -g '*.ts' -g '*.tsx' | grep -E "return (null|\[\]|\{\}|0|false)" | head -20

# 类型逃生口密度
rg -c "\bany\b|@ts-ignore|as unknown as" -g '*.ts' -g '*.tsx' | sort -t: -k2 -rn | head

# 同名函数出现多次（双胞胎候选；不同名同义的靠读，见对照法 F4/F5）
rg -n "^(export )?(async )?function \w+" -g '*.ts' -g '*.tsx' | awk -F: '{print $3}' | sort | uniq -d | head

# 函数最多的文件（同文件内重复概率最高）
rg -c "^(export )?(async )?function \w+" -g '*.ts' | sort -t: -k2 -rn | head -10
```

---

## 2. 过程症状

```bash
# 巨型提交（一次加很多、很少删 = vibe coding 签名）
git log --shortstat --pretty=format:"%h %ad %s" --date=short | awk '/insertion/{if($4>500)print prev" | +"$4" -"$6} {prev=$0}' | head -20

# 改动热点（与"大文件"交叉 = 重点）
git log --since="90 days ago" --name-only --pretty=format: | sort | uniq -c | sort -rn | head -20

# 只加不删的文件
git log --numstat --pretty=format: --since="180 days ago" | awk 'NF==3{a[$3]+=$1;d[$3]+=$2} END{for(f in a) if(a[f]>200 && d[f]<a[f]*0.1) print a[f], d[f], f}' | sort -rn | head

# 重构提交占比（用于排除自己上一次的重构，别被自己误导）
git log --oneline --since="180 days ago" | grep -cE "^\[?(surgery|refactor)\]?" 
```

---

## 3. 幻觉与供应链

```bash
# JS 包名是否存在（不存在的就是 AI 编的）
jq -r '.dependencies//{}|keys[]' package.json | while read p; do npm view "$p" version >/dev/null 2>&1 || echo "HALLUCINATED?: $p"; done

# 只用了一处的依赖（可能白装）
rg -n "from ['\"]([^'\"]+)" -g '*.ts' -g '*.tsx' -o -r '$1' | sort | uniq -c | sort -n | head -20

# devDependencies 里 0 引用的（先问再删）
jq -r '.devDependencies//{}|keys[]' package.json | head -30
```

---

## 4. 状态蔓延（前端/全栈专项）

```bash
# useEffect 配额：单组件超过 3 个就值得看
rg -c "useEffect\(" -g '*.tsx' -g '*.ts' | sort -t: -k2 -rn | head

# effect 内 set 另一个 store（双向同步 = 死循环风险）
rg -n -A8 "useEffect" -g '*.tsx' | grep -E "set[A-Z]\w+\(" | head -20

# 派生数据被另存：useState 的初值来自 props 或别的 state
rg -n "useState\([a-zA-Z_]\w*(\?\.|\.)" -g '*.tsx' | head -20
```

判据（三条都比"总数"有用）：

| 检查 | 判据 | 后果 |
|---|---|---|
| 派生数据被另存 | 新 state 能由已有 state 算出 | 迟早不同步 |
| effect 内 set 别的状态 | 一个 effect 改别人 | 死循环 / 竞态 |
| effect 配额 | 单组件 >3，或单 effect 依赖 >4 | 不许再新增，先收成单向数据流 |

---

## 5. 测试质量

```bash
# 断言很少的测试文件（可能是"跑一遍不报错"型）
rg -c "expect\(|assert" -g '*.test.*' -g '*.spec.*' | sort -t: -k2 -n | head

# mock 数 vs 断言数（mock 比断言多 = 只连不通）
rg -c "vi\.mock|jest\.mock" -g '*.test.*' | head

# 测试名承诺错误路径，断言里没有
rg -n -A6 "it\(['\"].*(reject|throw|error|invalid|fail)" -g '*.test.*' | grep -v "toThrow\|rejects\|toThrowError"
```

---

## 6. 扫描边界

先确认各工具的覆盖范围：**lint 的 `include` 常大于 tsc 的 `include`**。

```bash
cat tsconfig*.json | grep -A10 '"include"'
cat .eslintrc* .oxlintrc* biome.json 2>/dev/null | head -30
```

把"来自不参与构建的文件"的告警单独归类（例：图标草稿脚本），别混进主结论。
