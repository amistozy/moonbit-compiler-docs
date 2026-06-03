# 优化 Pass 详解

MoonBit 编译器在 Core IR 和 Mcore IR 上共运行 10+ 个优化 Pass。本章逐一分析每个 Pass 的算法和目的。

---

## Pass 执行顺序

```
Core.program (初始)
  │
  ├─ 1. InlineSingleUseJoin   ← 内联单次使用的 join
  ├─ 2. EliminateAsync        ← async → 状态机
  ├─ 3. Contification         ← 转换尾调用为 continuation
  ├─ 4. RemoveLetAlias        ← 消除 let 别名
  ├─ 5. Stackalloc            ← 可变记录的栈分配
  ├─ 6. UnboxLoopParams       ← 循环参数 unpacking
  ├─ 7. PropagateConstr       ← 构造器传播
  ├─ 8. LambdaLift            ← 闭包转换
  ├─ 9. DCE                   ← 死代码消除
  │
  ▼
Core.program (优化后)
  → 单态化 → Mcore.t
  │
  └─ 10. Layout              ← 数据布局优化
```

---

## 1. InlineSingleUseJoin (`pass_inline_single_use_join.ml`)

### 目的
将仅被调用一次（或零次）的 join / letfn 内联到调用点，消除不必要的中间绑定。

### 算法
1. **计数**：遍历表达式树，统计每个 join 的调用次数（`count_join_usage`）
2. **内联**：
   - 使用次数 = 0 → 直接删除该 join 绑定
   - 使用次数 = 1 → 将 join 体内联到唯一调用点（参数替换为实参）
   - 使用次数 > 1 → 保留

```ocaml
(* 内联单次使用的 join *)
method! visit_Cexpr_letfn ctx name fn body ty kind loc =
  match Ident.Hash.find_opt ctx.used_count_tbl name with
  | Some 0 -> self#visit_expr ctx body       (* 删除 *)
  | Some 1 ->                                 (* 内联 *)
      let fn_body = self#visit_expr ctx fn.body in
      (* 参数替换 *)
      ...
  | _ -> super#visit_Cexpr_letfn ...         (* 保留 *)
```

### 效果
- 减少不必要的 join 分配
- 简化后续 Pass 的分析

---

## 2. EliminateAsync (`eliminate_async.ml`)

### 目的
将 `async fn` 转换为基于 continuation 的状态机（CPS 变换）。

### 算法
1. **检测**：`need_cps` 检查表达式是否包含 `async` 调用或 `get_current_continuation`
2. **状态机生成**：
   - 每个 `await` 点成为一个状态
   - continuation 表示为 `join` 函数
   - async 函数体被分割为多个状态块

```ocaml
(* async fn foo(x) { ... await bar() ... } *)
(* 变换为： *)
(* state0: call bar(x, continuation=state1) *)
(* state1: continue with result *)
```

### 关键类型
```ocaml
type continuation =
  | Identity                              (* 无continuation *)
  | Return of { cont; cont_ty }           (* 返回到调用者 *)
  | Simple of { state_id }                (* 简单跳转 *)
  | Complex of (Core.expr -> Core.expr)   (* 复杂包装 *)
```

### 效果
- 消除所有 async/await 语法
- 产出纯同步的 CPS 风格 Core IR

---

## 3. Contification (`pass_contification.ml`)

### 目的
识别仅在尾位置调用的函数，将其转换为 continuation（join），避免函数调用开销。

### 算法
1. **可contification检查**：`is_contifiable` 分析函数的所有调用点
   - 仅在尾位置被调用 → `Contifiable`
   - 在非尾位置被调用 → `Not_contifiable { tail_called }`
   - 完全未被调用 → `Never_called`
2. **变换**：将 contifiable 的函数定义替换为 `letfn kind=Tail_join`

```ocaml
type contify_result =
  | Never_called
  | Contifiable
  | Not_contifiable of { tail_called : bool }
```

### 效果
- 消除尾递归的函数调用开销
- 为后续的 join 优化铺路

---

## 4. RemoveLetAlias (`pass_let_alias.ml`)

### 目的
消除 `let x = y in ...` 形式的平凡别名绑定。

### 算法
1. **收集别名**：扫描所有 `let` 绑定，记录 RHS 为变量或常量的情况
2. **替换**：将别名的使用替换为原始变量/常量

```ocaml
type alias_to =
  | Constant of Core.constant          (* let x = 42 → 直接用42 *)
  | Variable of Ident.t * ...          (* let x = y → 直接用y *)
```

### 效果
- 减少不必要的 let 嵌套
- 简化后续分析

---

## 5. Stackalloc (`pass_stackalloc.ml`)

### 目的
在安全的情况下，将可变记录（mutable record）分配从堆移到栈上。

### 算法
1. **分析阶段**（`analyze_stack_vars`）：
   - 记录每个 `let` 绑定的嵌套深度
   - 追踪记录字段访问 → 如果访问者深度大于定义者深度 → 需要栈分配
2. **变换阶段**：
   - 将符合条件的记录 `Cexpr_record` 替换为多个独立变量
   - `Cexpr_field r.f` → 对应变量
   - `Cexpr_mutate r.f = v` → 变量重赋值

```ocaml
(* 分析：记录深度 < 访问深度 → 无法优化为栈 *)
method! visit_Cexpr_field ctx record ... =
  match record with
  | Cexpr_var { id } ->
      match depth_of_vars.find id with
      | Some d when d < ctx.depth -> remove(id)  (* 逃逸了 *)
```

### 效果
- 减少 GC 堆分配
- 提升性能

---

## 6. UnboxLoopParams (`pass_unbox_loop_params.ml`)

### 目的
将循环参数从打包的记录类型中 unpack 为独立参数。

### 算法
1. **分析**：识别循环参数中哪些是仅被字段访问的记录
2. **变换**：
   - 将记录类型参数拆分为多个标量参数
   - 修改循环体的字段访问为直接变量引用
   - 更新所有 `continue` 的参数列表

### 效果
- 减少循环内的字段访问开销
- 改善循环的性能

---

## 7. PropagateConstr (`pass_propagate_constr.ml`)

### 目的
将构造器创建和模式匹配进行传播/融合。例如：
```
let x = Constr(a, b) in
match x { Constr(c, d) => body }
→ body[c/a, d/b]
```

### 算法
1. **内联决策**：基于构造器的使用次数和字段是否可变决定是否内联
2. **传播**：将构造器的参数直接替换到匹配臂中

```ocaml
type ctor_info = { mutable has_benefit_to_inline : bool }
(* 仅在所有字段不可变时才内联构造器传播 *)
let all_field_immutable genv ty = ...
```

### 效果
- 消除中间构造器分配
- 减少 match 分支

---

## 8. LambdaLift (`lambda_lift.ml`)

### 目的
将嵌套的闭包（lambda）提升为顶层函数，显式传递捕获的变量。

### 算法
1. **自由变量分析**：计算每个闭包的自由变量（`free_vars`）
2. **闭包转换**：
   - 为闭包生成一个打包环境的结构体类型（`env_ty`）
   - 将闭包体提升为顶层函数，增加一个 `env` 参数
   - 在闭包创建点分配环境并打包捕获变量
   - 在闭包调用点解包环境并调用顶层函数

```ocaml
type capture_info =
  | Single of Ident.t * Stype.t        (* 单个捕获 *)
  | Multiple of (Ident.t * Stype.t) list  (* 多个捕获 *)

type convert_context = {
  env_ty : Stype.t;    (* 环境结构体类型 *)
  captures : capture_info;
  ...
}
```

### 效果
- 消除嵌套函数定义
- 为后续的 Clam（无闭包级联）做准备

---

## 9. DCE (`core_dce.ml`)

### 目的
消除死代码——永远不会被执行的表达式。

### 算法
1. **纯度分析**：`is_pure` 判断表达式是否有副作用
2. **消除策略**：
   - 纯表达式 + 结果未使用 → 删除
   - 副作用表达式 → 保留

```ocaml
(* 纯表达式判定 *)
let rec is_pure expr = match expr with
  | Cexpr_const _ | Cexpr_var _ | Cexpr_function _ -> true
  | Cexpr_apply _ | Cexpr_mutate _ | Cexpr_assign _ -> false
  ...
```

### 效果
- 删除无用的纯计算
- 减小代码体积

---

## 10. Layout (`pass_layout.ml`)

### 目的
在 Mcore（单态化后）IR 上优化数据布局。

### 核心工作
1. **Option 优化**：将 `Option<T>` 映射为紧凑表示
   - `None` → 特殊哨兵值（如 `i32 = -1` 或 `i64 = 2^32`）
   - `Some(x)` → x 本身（当 x 不可能为哨兵值时）
   - `Option<Char>` → 直接用 i32（0=无, ≥1=字符码点+1）
2. **Null 编码**：用 `ref null` 表示 `None`，用非 null 引用表示 `Some`
3. **类型转换插入**：在需要时插入 `upcast`/`as_non_null`/`null`/`is_null`

```ocaml
(* Option 的紧凑编码 *)
let null ty = Mcore.prim ~ty Pnull []
let is_null e = Mcore.prim ~ty Mtype.T_bool Pis_null [e]
let as_non_null e ty = Mcore.prim ~ty Pas_non_null [e]
```

### 效果
- `Option<T>` 避免堆分配（对于值类型 T）
- `Option<Char>` 零开销表示
- 为 Clam 翻译提供优化的类型布局信息

---

## Pass 间的依赖与顺序

```
InlineSingleUseJoin     ← 最先运行，简化IR
    ↓
EliminateAsync          ← 依赖简化后的IR
    ↓
Contification           ← 在join内联之后，更精确的尾调用分析
    ↓
RemoveLetAlias          ← 清理
    ↓
Stackalloc              ← 利用已简化的let结构
    ↓
UnboxLoopParams         ← 在stackalloc之后，循环参数可能已被展开
    ↓
PropagateConstr         ← 在线性化IR后进行构造器传播
    ↓
LambdaLift              ← 最后的大变换，准备link
    ↓
DCE                     ← 最后清理死代码
    ↓
(单态化)
    ↓
Layout                  ← Mcore上的布局优化
```

这个顺序经过精心设计：每个 Pass 都为后续 Pass 创造更好的优化条件。
