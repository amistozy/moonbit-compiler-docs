# 类型检查器内幕

MoonBit 的类型检查器采用**双向类型检查**（Bidirectional Type Checking）算法，分为自上而下（checking）和自下而上（inference）两种模式。核心模块：`typer.ml`（5,391行）、`typecheck_driver_util.ml`、`toplevel_typer.ml`、`local_typing_worklist.ml`。

---

## 类型检查流水线

```
Parsing_parse.output
    │
    ▼
┌───────────────────────────────────────────┐
│ 1. toplevel_typer.ml                      │
│    check_toplevel:                        │
│    - 注册所有顶层类型/函数/trait声明        │
│    - 构建 Global_env                       │
│    - 产出 Local_typing_worklist.t         │
└───────────────────────────────────────────┘
    │ Local_typing_worklist.t (含Global_env)
    ▼
┌───────────────────────────────────────────┐
│ 2. typer.ml (type_check)                  │
│    - 对每个顶层绑定进行类型检查             │
│    - 工作列表驱动的双向检查                 │
│    - 产出 Typedtree.output               │
└───────────────────────────────────────────┘
    │ Typedtree.output
    ▼
┌───────────────────────────────────────────┐
│ 3. check_match.ml + topo_sort.ml          │
│    - 模式匹配完整性检查                     │
│    - 拓扑排序                              │
│    - 死代码/未使用分析                     │
└───────────────────────────────────────────┘
    │
    ▼
Typedtree.output (最终)
```

---

## 1. 顶层类型检查 (`toplevel_typer.ml`)

### 第一遍：注册声明

```ocaml
val check_toplevel :
  pkgs -> build_context -> Syntax.impl list list ->
  diagnostics -> Local_typing_worklist.t
```

第一遍遍历所有顶层声明：
1. 注册 `type`/`struct`/`enum`/`typealias` → `Global_env.add_type`
2. 注册 `trait` → `Global_env.add_trait`
3. 注册函数签名（不检查体）→ `Global_env.add_fn_decl`
4. 注册 `let`/`const` 签名 → `Global_env.add_let_decl`

这一遍确保后续的类型检查可以引用同包内的任何符号（允许前向引用）。

### 工作列表

`Local_typing_worklist.t` 收集所有需要类型检查的顶层绑定（函数体、let表达式等），供第二遍处理。

---

## 2. 第二遍：类型检查 (`typer.ml`)

### 入口

```ocaml
let type_check ~diagnostics (input : Local_typing_worklist.t) =
  (* 对工作列表中的每个项进行类型检查 *)
  (* 产出 Typedtree.output *)
```

### 双向类型检查

类型检查器支持两种模式：

```ocaml
type expect_ty = Expect_type of Stype.t | Ignored
(* Expect_type → 自上而下：我们知道期望什么类型 *)
(* Ignored     → 自下而上：推断实际类型 *)
```

### 核心函数

```ocaml
(* 表达式类型检查 *)
and typing_expr ?(expect_ty : Stype.t option) 
    (env : Local_env.t) (expr : Syntax.expr)
    : Typedtree.expr

(* 模式类型检查 *)
and typing_pattern (env : Local_env.t) (pat : Syntax.pattern)
    : Typedtree.pat * Stype.t * ...

(* let 绑定类型检查 *)
and typing_let (env : Local_env.t) (pat : Syntax.pattern) 
    (expr : Syntax.expr) (body : Syntax.expr)
    : Typedtree.expr

(* 函数调用类型检查 *)
and typing_application ?(expect_ty) (env : Local_env.t)
    (func : Typedtree.expr) (args : Syntax.argument list) ...
```

---

## 3. Unification (`ctype.ml`)

类型检查器通过 unification 确保类型一致：

```ocaml
(* 核心 unification *)
val unify : Stype.t -> Stype.t -> unit   (* 失败抛出 Unify *)

(* 带位置信息的 unification *)
val unify_expr : expect_ty:Sttype.t -> actual_ty:Sttype.t
    -> loc:Rloc.t -> error option
```

### Unification 算法（简化）

```
unify(T1, T2):
  1. 如果物理相等，立即返回
  2. deref T1, T2 到链接目标
  3. 匹配:
     - Tvar + Tvar → 链接到任意一边
     - Tvar + any  → 检查occur → 链接Tvar到具体类型
     - Tarrow + Tarrow → unify(params) + unify(return) + unify(error)
     - T_constr + T_constr → 如果路径相同 → unify(类型参数)
     - T_builtin + T_builtin → 检查相等
     - 其他 → 失败
```

### 类型约束求解

```ocaml
(* 泛型约束 *)
type constraint_info = { trait : Type_path.t; loc_ : Rloc.t; src_ : constraint_src }

(* 添加约束到类型变量 *)
val add_constraint : constraint_env -> Stype.t -> Type_path.t -> unit

(* 验证所有约束被满足 *)
val check_constraints : constraint_env -> ...
```

---

## 4. 局部环境 (`local_env.ml`)

类型检查器维护一个作用域化的局部环境：

```ocaml
type t (* 名字 → Value_info 的映射，支持嵌套作用域 *)
```

操作：
- `find_by_name_opt` — 查找局部变量
- `add_var` — 添加变量绑定
- `push_scope` / `pop_scope` — 作用域管理
- `in_new_scope` — 在新作用域中执行操作

作用域用于：
- 函数参数 → 新作用域
- let 绑定的 body → 扩展作用域
- match 臂 → 新作用域（含模式变量）
- for 循环体 → 新作用域

---

## 5. 函数类型检查

### 泛型函数

```ocaml
(* fn f[T : Show](x : T) -> T { ... } *)

(* 1. 创建 tvar_env (T → Stype.Tparam{0}) *)
(* 2. 将约束 T : Show 添加到 constraint_env *)
(* 3. 在新作用域中检查函数体 *)
(* 4. 检查返回类型与声明一致 *)
(* 5. 验证所有 trait 约束被满足 *)
```

### 方法（含 self 类型）

```ocaml
and typing_self_method (ty_self : Stype.t) (method_name : Syntax.label)
    ~src:(Syntax.method_call_src) ...
```

Method 调用需要特殊处理 self 类型的分发（静态 vs 动态）和 trait 方法解析。

---

## 6. Trait约束检查

```ocaml
(* trait_not_implemented *)
let trait_not_implemented ~trait ~type_name ~failure_reasons ~loc =
  (* 检查 impl 是否满足 trait 的所有方法要求 *)
  (* 可能失败原因:
     - Method_missing: 方法未实现
     - Private_method: 方法不可见
     - Type_mismatch: 方法签名不匹配
     - Method_constraint_not_satisfied: 方法约束未满足
     - Method_type_params_unsolved: 无法推断类型参数 *)
```

---

## 7. 错误恢复

类型检查器通过 `Tvar_error` 实现优雅降级：

```ocaml
(* 类型变量标记为 error → 所有后续 unification 自动成功 *)
and tvar_kind = Tvar_normal | Tvar_error

(* 在类型错误时 *)
let store_error ~diagnostics error =
  (* 记录错误 *)
  Diagnostics.add_error ...;
  (* 将相关类型变量标记为 error *)
  set_tvar_error ...
```

这确保一个类型错误不会导致级联的虚假错误。

---

## 8. 类型遍历器 (`Typedtree_util`)

类型化 AST 的工具模块，提供类型查找：

```ocaml
(* 获取表达式的类型 *)
val type_of_typed_expr : Typedtree.expr -> Stype.t

(* 获取模式的类型 *)
val type_of_pat : Typedtree.pat -> Stype.t
```

---

## 9. 局部类型环境

```ocaml
(* local_type.ml - 局部类型声明（函数内定义的 type/local struct） *)
type t  (* local type → type_info 映射 *)
```

支持函数内定义的局部类型（`let type X = ...` 风格的局部类型别名）。

---

## 10. 类型检查驱动 (`typecheck_driver_util.ml`)

```ocaml
val tast_of_ast :
  diagnostics -> build_context -> quiet -> genv_callback -> tast_callback ->
  import_items -> pkgs -> import_kind ->
  Parsing_parse.output list ->
  Typedtree.output * Global_env.t
```

完整调用链：
```
1. Toplevel_typer.check_toplevel  → 注册符号，产出工作列表
2. Typer.type_check               → 类型检查所有绑定
3. Check_match.analyze            → 模式匹配完整性
4. Topo_sort.topo_sort            → 拓扑排序
5. Dead_code.analyze_unused       → 死代码/未使用分析
6. Global_env.report_unused_pkg   → 报告未使用的导入
```
