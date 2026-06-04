# Core IR 生成：从 Typedtree 到 Core IR

`core_of_tast.ml`（2170行）是将类型化 AST（Typedtree）翻译为 Core IR 的核心模块。这是编译器流水线中承上启下的关键转换步骤：上游是类型检查器产出的 Typedtree，下游是 Core IR 优化流水线和后续编译阶段。

---

## 在流水线中的位置

```
类型检查器 (typer.ml, toplevel_typer.ml)
    │
    ▼
Typedtree.output  (类型化AST，含完整类型信息)
    │
    ▼
core_of_tast.ml  ← 【本模块】
    │
    ▼
Core.program  (Core IR 顶层项列表)
    │
    ▼
Core IR 优化Pass → 链接 → 单态化 → Clam IR → Wasm GC
```

---

## 1. 核心数据结构

### 1.1 翻译上下文 `transl_context`

翻译上下文跟踪翻译过程中的状态：

```ocaml
type transl_context = {
  return_ctx : return_context;     (* 返回控制流上下文 *)
  error_ctx : error_context option; (* 错误处理上下文 *)
  loop_ctx : labeled_loop_context; (* 循环上下文栈 *)
  error_ty : Stype.t option;       (* 当前函数的错误类型 *)
  return_ty : Stype.t;             (* 当前函数的返回类型 *)
  wrapper_info : Stype.t option;   (* Result包装类型，用于错误处理 *)
  base : Loc.t;                    (* 基准位置，用于源映射 *)
}
```

### 1.2 返回上下文 `return_context`

```ocaml
type return_context =
  | No_return           (* 函数体顶层：直接使用Core.return *)
  | Normal_return of {  (* 非函数位置：通过join点返回 *)
      return_join : Ident.t;
      mutable need_return_join : bool
    }
  | Foreach_return of foreach_context  (* for..in循环中的return *)
```

- `No_return`：在函数体内，`return`直接生成`Cexpr_return`节点
- `Normal_return`：在非函数位置（如`let`表达式的RHS中），`return`编译为跳转到join点
- `Foreach_return`：`for..in`循环内，需要将返回值包装为`Foreach_util.Return`构造器并写入可变字段

### 1.3 错误上下文 `error_context`

```ocaml
type error_context = {
  raise_join : Ident.t;
  mutable need_raise_join : bool
}
```

当函数体中有`try-catch`时，`raise`翻译为跳转到`raise_join`。否则直接生成`Cexpr_return`（`Error_result`类型）。

### 1.4 循环上下文 `loop_context`

```ocaml
type loop_context =
  | Loop_label of Label.t           (* while/loop *)
  | For_loop_info of {              (* 函数式for循环 *)
      label : Label.t;
      continue_join : Ident.t;
      mutable need_for_loop_join : bool;
    }
  | Foreach of foreach_context      (* for..in循环 *)
```

---

## 2. 翻译入口：`transl`

```ocaml
let transl ~global_env (output : Typedtree.output) =
  let (Output { value_defs; _ }) = output in
  Lst.concat_map value_defs (transl_impl ~global_env)
```

`Typedtree.output`包含一个`value_defs`列表，每个`value_def`是一个`Typedtree.impl`：

### 2.1 `transl_impl` 分发

```ocaml
let transl_impl ~global_env (impl : Typedtree.impl) =
  match impl with
  | Timpl_expr { expr; is_main; loc_ } ->
      (* 顶层表达式（如test块、main表达式） *)
      [ Ctop_expr { expr = transl_top_expr expr; is_main; loc_ } ]
  | Timpl_letdef { binder; expr; is_pub; loc_; _ } ->
      (* let绑定 *)
      [ Ctop_let { binder; expr; is_pub_; loc_ } ]
  | Timpl_fun_decl { fun_decl; arity_; loc_ } ->
      (* 函数声明，包括主函数和默认参数函数 *)
      Ctop_fn { binder; func; subtops; ty_params_; is_pub_; loc_ }
      :: generate_default_exprs (...)
  | Timpl_stub_decl { func_stubs; binder; ... } ->
      (* 外部函数桩（FFI声明） *)
      Ctop_stub { binder; func_stubs; ... } :: default_exprs
```

### 2.2 Core IR 顶层项类型

```ocaml
type top_item =
  | Ctop_expr of { expr; is_main; loc_ }        (* 顶层表达式 *)
  | Ctop_let of { binder; expr; is_pub_; loc_ } (* 全局let绑定 *)
  | Ctop_fn of top_fun_decl                      (* 函数定义 *)
  | Ctop_stub of { binder; func_stubs; ... }    (* 外部函数桩 *)
```

---

## 3. 表达式翻译：`transl_expr`

`transl_expr` 是表达式翻译的核心函数（约1200行），负责将 `Typedtree.expr` 翻译为 `Core.expr`。

```ocaml
let rec transl_expr ~is_tail ~need_wrap_ok ~global_env ~tvar_env ctx texpr =
```

参数说明：
- `is_tail`：处于尾调用位置则为`true`（影响`return`和异常处理的翻译）
- `need_wrap_ok`：是否需要将结果用`Ok`包装（用于有错误类型的函数）
- `ctx`：翻译上下文
- `tvar_env`：类型变量环境

### 3.1 翻译对照表

| Typedtree 表达式 | Core IR 表达式 | 翻译要点 |
|---|---|---|
| `Texpr_constant` | `Cexpr_const` | 大整数特殊处理：转为`BigInt::from_string`调用 |
| `Texpr_unit` | `Cexpr_unit` | 简单映射 |
| `Texpr_ident (Normal)` | `Cexpr_var` | 直接变量引用 |
| `Texpr_ident (Mutable)` | `Cexpr_field (Ref, pos=0)` | 可变变量包装为单字段Ref record |
| `Texpr_ident (Value_constr)` | `Pcast (Constr_to_enum)` | 构造器值转为整数 |
| `Texpr_ident (Prim)` | `Cexpr_var` (带prim标记) | 带primitive注解的变量 |
| `Texpr_method` | `Cexpr_var` | 方法转为变量引用 |
| `Texpr_unresolved_method` | `Cexpr_var` 或 `Cexpr_prim` | Trait方法解析 |
| `Texpr_as` | `Cexpr_as` | 类型转换（to trait） |
| `Texpr_let` | `Cexpr_let` + 模式匹配翻译 | 通过`Transl_match.transl_let`展开 |
| `Texpr_letmut` | `Cexpr_let` + `Cexpr_record` | 可变变量=Ref record |
| `Texpr_function` | `Cexpr_function` | 匿名函数 |
| `Texpr_apply` | `Cexpr_apply` / `Cexpr_prim` / `Cexpr_constr` | 函数调用（见下文详述） |
| `Texpr_constr` | `Cexpr_constr` / `Cexpr_function` | 构造器值或构造器函数 |
| `Texpr_tuple` | `Cexpr_tuple` | 元组 |
| `Texpr_record` | `Cexpr_record` | 记录构造 |
| `Texpr_record_update` | `Cexpr_record` / `Cexpr_record_update` | 记录更新（字段≤6用复制，>6用更新） |
| `Texpr_field` | `Cexpr_field` | 字段访问 |
| `Texpr_mutate` | `Cexpr_mutate` | 可变字段赋值 |
| `Texpr_array (fixed)` | `Cexpr_array` | 定长数组 |
| `Texpr_array (resizable)` | `Core_util.make_array_make` | 可变数组用`Array::make` |
| `Texpr_assign` | `Cexpr_mutate (Ref)` + `Cexpr_field` | 可变变量赋值 |
| `Texpr_sequence` | `Cexpr_sequence` | 语句序列 |
| `Texpr_if` | `Cexpr_if` | 条件分支 |
| `Texpr_is` + `is`模式 | `Transl_match.transl_match` | `is`模式编译为match |
| `Texpr_match` | `Transl_match.transl_match` | 模式匹配 |
| `Texpr_try` | `Cexpr_handle_error` + join | try-catch |
| `Texpr_exclamation` | `Cexpr_handle_error` | `!`操作符（try!） |
| `Texpr_while` | `Cexpr_loop` | while循环 |
| `Texpr_for` | `Cexpr_loop` | 函数式for循环 |
| `Texpr_foreach` | `Cexpr_loop` + 迭代器调用 | for..in循环 |
| `Texpr_return` | `Cexpr_return` / `Cexpr_join_apply` | return语句 |
| `Texpr_raise` | `Cexpr_return (Error_result)` / `Cexpr_join_apply` | raise语句 |
| `Texpr_break` | `Cexpr_break` | break |
| `Texpr_continue` | `Cexpr_continue` / `Cexpr_join_apply` | continue |
| `Texpr_loop` | `Cexpr_loop` | 无标签循环 |
| `Texpr_pipe` | `Cexpr_apply` | 管道运算符`|>` |
| `Texpr_interp` | `Cexpr_sequence` (write调用链) | 字符串插值展开 |
| `Texpr_guard` | `Cexpr_if` | guard语句翻译为if |
| `Texpr_constraint` | (透传内层expr) | 类型约束已消解 |
| `Texpr_hole` | (编译错误) | 类型化hole |

---

## 4. 函数调用翻译：`transl_apply` 和 `make_apply`

### 4.1 `transl_apply`

```ocaml
and transl_apply ~global_env ~tvar_env ~ctx ~kind ~ty ~loc_ func args =
```

函数调用是翻译中最复杂的部分，需要处理：
1. **带标签参数的重排**：`process_labelled_args`解析`Fn_arity`，将位置参数和标签参数映射到正确位置
2. **可选参数**：未传入时调用默认值函数`name.default`
3. **Question可选参数**：`arg? : T`翻译为`Option[T]`，未传入→`None`，传入普通值→`Some(val)`
4. **Autofill参数**：`SourceLoc`类型自动填入源码位置，`Argsloc`填入所有参数的SourceLoc数组
5. **错误类型处理**：如果函数类型包含`err_ty`，结果类型自动包装为`MultiValueResult`

### 4.2 `make_apply`

```ocaml
and make_apply ~global_env ~tvar_env ctx func args ~loc ~ty =
```

实际生成Core IR调用节点的函数：

1. **Primitive识别**：若被调用者是`Pintrinsic`，首先尝试`Core_util.try_apply_intrinsic`进行常量折叠/优化
2. **特殊化（specialize）**：`Core_util.specialize`将限定名调用转换为专用prim（如`@int.add` → `Padd_int`）
3. **Normal vs Async**：根据函数类型判断`is_async`标记
4. **计算型func**：若`func`不是简单变量（如返回值是函数的调用），先`let`绑定再调用

### 4.3 级联调用（Cascade）脱糖

`Texpr_apply { kind_ = Dot_return_self }` 编译为链式调用：

```mbt
// MoonBit源码
self.f()?.g(a)?.h(b)

// 翻译结果（伪代码）
let self = self
self.f()?
self.g(a)?
self.h(b)
self
```

级联脱糖通过`desugar_cascade`递归实现，遇到`Dot_return_self`调用时，将self作为第一个参数传递并保留为链中下一个调用的self。

---

## 5. 函数翻译：`transl_fn`

```ocaml
and transl_fn ~global_env ~tvar_env ~base (fn : Typedtree.fn) =
```

1. 提取参数信息（名称、类型、位置）
2. 根据返回类型和错误类型确定`wrapper_info`：
   - 无错误类型：`wrapper_info = None`
   - 有错误类型：`wrapper_info = Some (result_ty)`，结果自动包装为`Result[ok_ty, err_ty]`
3. 设置`return_ctx = No_return`，函数体顶层可直接`return`
4. `need_wrap_ok`根据是否有错误类型自动确定

---

## 6. 错误处理翻译

### 6.1 `wrap_ok_prim`

有错误类型的函数中，正常返回值需要包装为`Ok(value)`：

```ocaml
let wrap_ok_prim expr ~result_ty =
  (* tail_is_optimizable: 判断是否可以将Ok构造器推到尾部 *)
  (* 如果尾部可优化，递归push_ok_to_tail避免冗余包装 *)
  (* 否则在最外层wrap_ok_expr *)
```

尾部优化（`push_ok_to_tail`）：当表达式的最后一个动作是`return`/`join apply`/`handle_error(Joinapply|Return_err)`时，将`Ok`包装推到动作内部。

### 6.2 `transl_return`

```ocaml
and transl_return ctx ~is_tail return_value ~ty ~loc_ =
```

三种情况：
- `Foreach_return`：包装为`Foreach_util.Return(value)`并跳转到退出点
- `No_return` + 无错误类型：`is_tail`为true时直接返回，false时生成`Cexpr_return(Single_value)`
- `No_return` + 有错误类型：生成`Cexpr_return(Error_result { is_error = false })`，即`Ok(value)`
- `Normal_return`：跳转到return_join点

### 6.3 `transl_raise`

```ocaml
and transl_raise ctx error_value ~ty ~loc_ =
```

两种情况：
- 有`error_ctx`：跳转到`raise_join`（在`try`块内）
- 无`error_ctx`：直接生成`Cexpr_return(Error_result { is_error = true })`

### 6.4 Try-Catch翻译

```
Texpr_try { body; catch; catch_all; try_else; err_ty; ... }

翻译为：
let try_join(err_val) = match_constr err_val { catch_cases } in
let body = transl_expr(body) in  (* 其中raise跳转到try_join *)
Cexpr_handle_error { obj = body; handle_kind = Joinapply try_join }
```

---

## 7. 控制流翻译

### 7.1 `transl_break` 和 `transl_continue`

通过`find_loop_ctx`查找最近的循环上下文：
- `Loop_label` / `For_loop_info`：直接生成`Cexpr_break`/`Cexpr_continue`
- `Foreach`：通过`mutable result_var`写入`Break(value)`/`Continue`构造器并跳转到`exit_join`
- `Jump_out_foreach`：包裹标签的`break`/`continue`，包装为`JumpOuter`载荷

### 7.2 `transl_cond_contain_is`

处理包含`is`模式的条件表达式（`expr is Pat`、`&&`、`||`）：

- `Texpr_is`：编译为`Transl_match.transl_match`（单分支match）
- `Texpr_and`：短路求值，为假分支创建join点
- `Texpr_or`：短路求值，为真分支创建join点

`in_pattern_guard`标记：用于match guard中的`is`，需要内联action以避免跳转。

---

## 8. 辅助翻译函数

### 8.1 `transl_top_expr`

翻译顶层表达式（test块、main表达式等）：

```ocaml
let transl_top_expr expr ~global_env ~base =
```

- 创建`Normal_return`返回上下文
- 表达式译文后，如果`need_return_join`为true，添加一个`Nontail_join`绑定

### 8.2 `generate_default_exprs`

为带可选参数的函数生成默认值函数：

```mbt
// MoonBit源码
fn f(x: Int, y: Int = x + 1) { ... }

// 生成的默认值函数（伪Core IR）
fn f.y.default(x: Int) -> Int { x + 1 }
```

默认参数函数的命名规则：`{函数名}.{参数标签}.default`

### 8.3 `transl_trait_method`

解析trait方法调用：

- 如果self类型是`Tparam`（类型参数）：生成`local_method`标识符（由单态化时解析）
- 否则查找`Global_env.find_trait_method`确定具体实现
- 返回`Prim`（内建trait方法）或`Regular`（普通方法引用）

---

## 9. Newtype 处理

MoonBit的newtype是零开销类型包装：

```mbt
type Email String    // newtype
```

翻译时的特殊处理：
- **构造**：`Email("test")` → `Pcast { kind = Make_newtype }`（恒等映射）
- **解构**：`email._` → `Pcast { kind = Unfold_rec_newtype }`（递归newtype需要）
- **字段访问**：`.Newtype`访问器 → 直接`Cexpr_field(pos=0, Newtype)`

非递归newtype：解构等同于字段访问`pos=0`。

---

## 10. 可变变量处理

MoonBit的`let mut x = 1`翻译为：

```mbt
let mut x = 1
x = 2
```

转化为：

```ocaml
let x = { val: 1 }    (* Ref record *)
x.val = 2             (* mutate Ref field *)
```

- `mutable_var_label = { label_name = "val" }`
- 类型：`mutable_var_type ty = Builtin.type_ref ty`
- 读取：`Cexpr_field(Ref_var, pos=0, Label("val"))`
- 赋值：`Cexpr_mutate(Ref_var, pos=0, Label("val"), new_val)`

---

## 11. 优化机会

`make_apply`中的关键优化：

1. **Intrinsic内联**：`Core_util.try_apply_intrinsic`在编译时对已知intrinsic进行常量折叠
2. **运算符特化**：`Core_util.specialize`将`@int.add`等通用调用转为专用prim指令
3. **Ok包装优化**：`push_ok_to_tail`将`Ok(value)`推入return/apply内部，避免中间包装

---

## 总结

`core_of_tast.ml` 将类型丰富的 Typedtree 系统地转换为结构化的 Core IR：

| 方面 | Typedtree | Core IR |
|---|---|---|
| 类型 | `Stype.t`（完整类型） | `Stype.t`（保留类型信息） |
| 变量 | 有`kind`标记（Normal/Mutable/Prim/Value_constr） | 统一`Ident.t` |
| 函数调用 | 灵活的标签/可选/autofill参数 | 位置参数列表（已展开） |
| 模式匹配 | 完整模式语法 | `Cexpr_switch_constr`/`Cexpr_switch_constant` |
| 错误处理 | `raise`/`try-catch`/`!` | `Cexpr_return(Error_result)`/`Cexpr_handle_error` |
| 循环 | `while`/`for`/`foreach`/`loop` | 统一的`Cexpr_loop`（循环）或展开的迭代器调用 |
