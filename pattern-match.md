# 模式匹配编译详解

MoonBit 的模式匹配从语法AST经类型检查、静态分析、编译到 Core IR 的 switch/if 决策树。相关模块：`pattern_typer.ml`、`check_match.ml`、`transl_match.ml`、`patmatch_db.ml`、`patmatch_static_info.ml`。

---

## 模式匹配流水线

```
match expr {
  pattern1 => action1
  pattern2 => action2
  ...
}
    │
    ▼
┌──────────────────────────────────────┐
│ 1. 语法分析 (parsing_main.ml)        │
│    产出: Pexpr_match + pattern AST   │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│ 2. 模式类型检查 (pattern_typer.ml)   │
│    - 推断每个模式的类型               │
│    - 确保 arm 类型一致               │
│    - 检查穷举性                      │
│    产出: Tpat_* types               │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│ 3. 完整性/可达性分析                  │
│    patmatch_db.ml + check_match.ml   │
│    - 判断是否穷举                     │
│    - 标记不可达分支                   │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│ 4. 编译为 Core IR (transl_match.ml)  │
│    - 构造决策树                       │
│    - 生成 switch/if                    │
│    产出: Cexpr_switch_constr /        │
│           Cexpr_switch_constant /     │
│           Cexpr_if 序列               │
└──────────────────────────────────────┘
```

---

## 1. MoonBit 支持的模式

### 语法层的模式种类

```ocaml
type pattern =
  | Ppat_constant of constant              (* 42, true, 'a', "hello" *)
  | Ppat_var of binder                     (* x *)
  | Ppat_any                               (* _ *)
  | Ppat_constr of { constr; args; is_open } (* None, Some(x), Enum::Variant *)
  | Ppat_tuple of pattern list             (* (a, b) *)
  | Ppat_record of { fields; is_closed }   (* { x, y } *)
  | Ppat_map of { elems; is_closed }       (* {1: v} *)
  | Ppat_array of array_patterns           (* [a, b, ..rest] *)
  | Ppat_or of { pat1; pat2 }              (* 1 | 2 *)
  | Ppat_range of { lhs; rhs; inclusive }  (* 1..10, 'a'..='z' *)
  | Ppat_alias of { pat; alias }           (* pat as name *)
  | Ppat_constraint of { pat; ty }         (* (pat : Type) *)
```

### 数组模式

```ocaml
type array_pattern =
  | Pattern of pattern              (* 元素模式 *)
  | String_spread of string         (* 字符串展开 *)
  | String_spread_const of { binder; pkg; loc_ }  (* 具名字符串展开 *)

type array_patterns =
  | Closed of array_pattern list     (* [a, b, c] *)
  | Open of array_pattern list * array_pattern list * binder option
    (* [a, ..rest, b] -- open pattern *)
```

---

## 2. 模式类型检查 (`pattern_typer.ml`, 898行)

### 主要职责
1. 为每个模式推断/检查类型
2. 处理 `or` 模式（两侧类型必须一致）
3. 处理别名模式（跟踪变量绑定）
4. 穷举性检查

### 类型化后的模式 AST (`typedtree.ml`)

```ocaml
type pat =
  | Tpat_any
  | Tpat_var of binder
  | Tpat_alias of { pat; alias; ty }
  | Tpat_constant of { c; ty }
  | Tpat_constr of { constr; tag; args; is_open; ty }
  | Tpat_tuple of { pats; ty }
  | Tpat_record of { fields; is_closed; ty }
  | Tpat_array of { pats; ty }
  | Tpat_or of { pat1; pat2; ty }
  | Tpat_map of { elems; is_closed; ty }
  | Tpat_range of { lhs; rhs; inclusive; ty }
  | Tpat_constraint of { pat; ty }
```

---

## 3. 穷举性与可达性分析

### patmatch_db.ml

维护一个模式匹配的"用例数据库"：
- 对于每种类型的构造器集合，跟踪哪些分支已被覆盖
- 判断 match 是否穷举所有可能的构造器/值

### check_match.ml

```ocaml
(* 标记可达模式 *)
val mark_reachable : book -> pat_id -> unit

(* 检查是否不可达 *)
val is_unreachable : book -> pat_id -> bool

(* 报告未覆盖的警告 *)
val report_unreachable : diagnostics -> match_case -> unit
val report_unused_pat : diagnostics -> pat -> unit
```

算法：
1. 构造一个覆盖矩阵（pattern matrix）
2. 逐行检查每个模式是否可能匹配
3. 标记不可达的 arm
4. 对未被任何 arm 覆盖的值域发出警告

---

## 4. 编译为 Core IR (`transl_match.ml`)

### 核心翻译数据结构

```ocaml
type case = {
  pat : Typedtree.pat;            (* 模式 *)
  pat_binders : pat_binders;      (* 模式中的变量绑定 *)
  action : Core.expr;             (* arm的action *)
  guard : (true_case → false_case → expr) option;  (* if guard *)
  ...
}

type arm_info =
  | Joinpoint of { arm_id; pat_params; guard_params; action }
  | Inline of { pat_params; action }
```

### 匹配臂翻译策略

每个 arm 可以有三种编译方式：

| 方式 | 条件 | 翻译 |
|------|------|------|
| Inline | 简单 arm + 无共享 | 直接在决策树中生成代码 |
| Joinpoint | 复杂 arm / 多次使用 | 生成一个 join 函数，决策树跳转到它 |
| 混合 | — | guard 测试 inline，body 为 joinpoint |

### 决策树生成

**枚举/构造器匹配** → `Cexpr_switch_constr`：
```
match x {
  None => a
  Some(v) => b
}
→ Cexpr_switch_constr(obj=x, cases=[(None, a), (Some, b)])
```

**整数/字符常量匹配** → `Cexpr_switch_constant`：
```
match x {
  1 => a
  2 => b
  _ => c
}
→ Cexpr_switch_constant(obj=x, cases=[(1,a), (2,b)], default=c)
```

**元组/记录匹配** → 嵌套的解构 + if：
```
match (x, y) {
  (1, z) => a
  (_, _) => b
}
→ let tmp1 = x; let tmp2 = y in
  if tmp1 == 1 then a[z=tmp2] else b
```

**范围模式** → 比较链：
```
match x {
  1..=10 => a
  _ => b
}
→ if x >= 1 && x <= 10 then a else b
```

**or 模式** → 复制 arm body：
```
match x {
  1 | 2 => a
}
→ Cexpr_switch_constant(obj=x, cases=[(1,a'), (2,a')])
(其中 a' 是 a 的副本)
```

**数组模式** → 长度检查 + 元素解构：
```
match arr {
  [a, b, ..rest] => body
}
→ if arr.length >= 2 {
    let a = arr[0]; let b = arr[1]; let rest = arr[2..];
    body
  } else { ... }
```

### 数组展开模式

`[a, ..s, b]` 模式翻译为：
```ocaml
(* Closed: [a, b, c] *)
→ 检查长度 == 3，解构三个元素

(* Open: [a, ..rest, c] *)
→ 检查长度 >= 2，解构第一个和最后一个，rest = arr[1..len-2]
```

### Guard 表达式

```mbt
match x {
  Some(v) if v > 0 => a
  Some(v) => b
}
```

翻译为嵌套 if：
```
if x is Some(v) && v > 0 then a
else if x is Some(v) then b
else ...
```

---

## 5. 优化

### 内联决策

`inline_action` 字段用于判断 arm body 是否足够简单可以直接内联（vs 生成 joinpoint）：
- 短小的常量表达式 → inline
- 大的、多次使用的 body → joinpoint

### 模式矩阵优化

`patmatch_static_info.ml` 在静态分析阶段：
- 对 switch 的 case 顺序进行优化（最常见 case 在前）
- 合并等价的 arm

---

## 6. 完整示例

```mbt
enum Option[T] { None; Some(T) }

fn get_or[T](x : Option[T], default : T) -> T {
  match x {
    None => default
    Some(v) => v
  }
}
```

编译为 Core IR：
```
Cexpr_switch_constr {
  obj = x;
  cases = [
    (None_tag,  Cexpr_var { id = default }),
    (Some_tag,  Cexpr_field { record = x; pos = 0 })
  ];
  default = Cexpr_unit  (* 穷举，不会走 *)
}
```

Clam IR 抽象：
```
Lswitch {
  obj = x;
  cases = [
    (0, Lvar {default}),          (* None: tag=0, 返回default *)
    (1, Lget_field {x, pos=0})    (* Some: tag=1, 读取payload *)
  ];
  default = unreachable
}
```

Wasm GC：
```wasm
block $exit (result ...)
  local.get $x
  struct.get $Option 0   ;; 读 tag
  br_table 0 1 $unreachable  ;; 跳转
end
```
