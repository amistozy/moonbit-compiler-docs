# 中间表示（IR）详解

MoonBit 编译器使用四层中间表示，每层逐步降低抽象级别，最终映射到 Wasm GC 指令集。

---

## IR 层次总览

```
源文件 (.mbt)
  → Syntax AST (parsing_syntax.ml)    语法层，未类型化
  → Typedtree  (typedtree.ml)         类型化语法层
  → Core       (core.ml)              高级IR，泛型
  → Mcore      (mcore.ml)            单态化IR
  → Clam       (clam.ml)              低级IR，接近Wasm GC
  → Dwarfsm    (dwarfsm_ast.ml)       Wasm IR 编码层
  → .wasm                              二进制输出
```

---

## 1. Core IR (`core.ml`, 4,854行)

### 定位

Core IR 是类型检查之后的第一层中间表示，由 `core_of_tast.ml` 从 Typedtree 翻译而来。它保留了高级语言结构（如闭包、async、trait对象），但消除了语法糖，是大多数优化 Pass 的工作对象。**仍然包含泛型参数。**

### 顶层结构

```ocaml
type program = top_item list

type top_item =
  | Ctop_expr     (* 顶层表达式（如main） *)
  | Ctop_let      (* 顶层let绑定 *)
  | Ctop_fn       (* 顶层函数声明 *)
  | Ctop_stub     (* FFI外部桩函数 *)
```

### 核心表达式

```ocaml
type expr =
  (* 字面量 *)
  | Cexpr_const      (* 常量 *)
  | Cexpr_unit       (* unit *)

  (* 变量与引用 *)
  | Cexpr_var        (* 变量引用（携带可选prim标记） *)
  | Cexpr_assign     (* 变量赋值 *)

  (* 函数 *)
  | Cexpr_function   (* 函数字面量（闭包） *)
  | Cexpr_apply      (* 函数调用 *)
  | Cexpr_letfn      (* let绑定函数 *)
  | Cexpr_letrec     (* 相互递归letrec *)

  (* 数据构造 *)
  | Cexpr_constr     (* 枚举构造器 *)
  | Cexpr_tuple      (* 元组 *)
  | Cexpr_record     (* 记录/结构体 *)
  | Cexpr_record_update  (* 记录更新 *)
  | Cexpr_array      (* 数组字面量 *)

  (* 字段访问 *)
  | Cexpr_field      (* 取字段 *)
  | Cexpr_mutate     (* 写可变字段 *)

  (* 控制流 *)
  | Cexpr_sequence   (* 顺序执行 *)
  | Cexpr_if         (* 条件 *)
  | Cexpr_switch_constr    (* 构造器switch *)
  | Cexpr_switch_constant  (* 常量switch *)
  | Cexpr_loop       (* 循环 *)
  | Cexpr_break      (* break *)
  | Cexpr_continue   (* continue *)
  | Cexpr_return     (* return *)

  (* 错误处理 *)
  | Cexpr_handle_error    (* 错误处理 *)
  | Cexpr_as              (* trait对象转型 *)

  (* 短路逻辑 *)
  | Cexpr_and        (* 逻辑与 *)
  | Cexpr_or         (* 逻辑或 *)

  (* 基元操作 *)
  | Cexpr_prim       (* 算术/位操作等 *)
```

### 类型系统

Core IR 的类型使用 `Stype.t`（结构类型），这是一个带有类型变量链接的类型系统：

```ocaml
type t =
  | Tarrow         (* 函数类型: params → ret ?err *)
  | T_constr       (* 命名类型 *)
  | Tvar of tlink ref  (* 类型变量（unification用） *)
  | Tparam         (* 泛型参数 *)
  | T_trait        (* trait 类型 *)
  | T_builtin      (* 内建类型 *)
  | T_blackhole    (* 占位 *)

and tlink = Tnolink | Tlink of t
```

### 关键特点

- **保留泛型**：`top_fun_decl` 包含 `ty_params_: tvar_env`
- **异步支持**：`fn.is_async` 标记，`Cexpr_apply` 的 `Async` 调用约定
- **Join点**：`letfn_kind = Tail_join | Nontail_join` 用于 continuation 传递
- **异常/错误结果**：`return_kind = Error_result | Single_value`，`handle_kind`

---

## 2. Mcore IR (`mcore.ml`, 4,893行)

### 定位

Mcore 是**单态化后的 Core IR**，由 `monofy.ml` 从多个 Core 模块链接并展开泛型而来。所有泛型参数已被消除，每个函数调用都有具体的类型签名。类型系统从 `Stype.t` 变为 `Mtype.t`。

### 与 Core 的主要差异

| 方面 | Core | Mcore |
|------|------|-------|
| 类型 | `Stype.t`（含Tvar/Tparam） | `Mtype.t`（全具体类型） |
| 泛型 | 保留 `ty_params_` | 无泛型参数 |
| async | `is_async: bool` | 已消除（转为状态机） |
| apply_kind | `Normal \| Async \| Join` | `Normal \| Join` |
| fn | `is_async: bool` | 无此字段 |
| 新增 | — | `Cexpr_object`（trait对象创建） |
| ty_args_ | `typ array`（泛型实参） | 不需要 |

### 新增表达式

```ocaml
| Cexpr_object of {
    methods_key : Object_util.object_key;
    self : expr;
    ty : typ;
    loc_ : location;
  }
```

Trait 对象在单态化后变为具体的方法表对象。

### Mtype 类型

```ocaml
type t =
  | T_int | T_char | T_bool | T_unit | T_byte
  | T_int16 | T_uint16 | T_int64 | T_uint | T_uint64
  | T_float | T_double | T_string | T_bytes
  | T_optimized_option of { elem : t }   (* 优化Option表示 *)
  | T_func of { params : t list; return : t }
  | T_raw_func of { params : t list; return : t }
  | T_tuple of { tys : t list }
  | T_fixedarray of { elem : t }
  | T_constr of id         (* 具名构造器类型 *)
  | T_trait of id          (* 具名trait类型 *)
  | T_any of { name : id } (* 存在类型 *)
  | T_maybe_uninit of t    (* 可能未初始化 *)
  | T_error_value_result of { ok : t; err : t; id : id }  (* Result!类型 *)
```

---

## 3. Clam IR (`clam.ml`, 3,332行)

### 定位

Clam 是**低级中间表示**，由 `clam_of_core.ml` 从 Mcore 翻译而来。它显式地管理内存分配、闭包捕获、控制流，紧密对应 Wasm GC 的能力。类型系统为 `Ltype_gc.t`。

### 程序结构

```ocaml
type prog = {
  fns      : top_func_item list;  (* 所有顶层函数 *)
  main     : lambda option;       (* main入口 *)
  init     : lambda;              (* 初始化代码 *)
  globals  : (binder * constant option) list;  (* 全局变量 *)
  type_defs : type_defs;          (* Wasm GC类型定义 *)
}

type top_func_item = {
  binder   : address;       (* 函数地址 *)
  fn_kind_ : fn_kind;       (* Top_pub name | Top_private *)
  fn       : fn;
  tid      : tid option;    (* Wasm类型索引 *)
}

type fn = {
  params       : binder list;
  body         : lambda;
  return_type_ : ltype list;
}
```

### lambda 表达式（Clam 的核心 AST）

```ocaml
type lambda =
  (* 内存分配 *)
  | Lallocate of aggregate               (* 分配数据结构 *)
  | Lclosure of closure                   (* 创建闭包 *)
  | Lget_raw_func of address              (* 获取原始函数指针 *)

  (* 字段访问 *)
  | Lget_field of { obj; tid; index; kind }     (* 读字段 *)
  | Lclosure_field of { obj; tid; index }       (* 读闭包捕获 *)
  | Lset_field of { obj; field; tid; index; kind }  (* 写字段 *)

  (* 数组操作 *)
  | Lmake_array of { tid; kind; elems }         (* 创建数组 *)
  | Larray_get_item of { tid; kind; arr; index; extra }  (* 读数组 *)
  | Larray_set_item of { tid; kind; arr; index; item }   (* 写数组 *)

  (* 函数调用 *)
  | Lapply of { fn : target; prim; args }     (* 函数调用 *)
  | Lstub_call                                (* FFI调用 *)

  (* 常量 *)
  | Lconst of constant

  (* 控制流 *)
  | Lloop of { params; body; args; label; type_ }    (* 循环 *)
  | Lif of { pred; ifso; ifnot; type_ }              (* 条件 *)
  | Lswitch of { obj; cases; default; type_ }        (* 构造器switch *)
  | Lswitchint of { obj; cases; default; type_ }     (* 整数switch *)
  | Lswitchstring of { obj; cases; default; type_ }  (* 字符串switch *)
  | Lbreak of { arg; label }
  | Lcontinue of { args; label }
  | Lreturn of lambda

  (* 变量 *)
  | Llet of { name; e; body }           (* let绑定 *)
  | Lletrec of { names; fns; body }     (* 递归绑定 *)
  | Lvar of { var }
  | Lassign of { var; e }

  (* Join点（continuation） *)
  | Ljoinlet of { name; params; e; body; kind; type_ }
  | Ljoinapply of { name; args }

  (* 顺序 *)
  | Lsequence of { exprs; last_expr }

  (* 基元操作 *)
  | Lprim of { fn; args }

  (* 异常 *)
  | Lcatch of { body; on_exception; type_ }

  (* 类型转换 *)
  | Lcast of { expr; target_type }

  (* 调试 *)
  | Levent of { expr; loc_ }            (* 位置标记 *)
```

### 调用目标

```ocaml
type target =
  | Dynamic of var              (* 动态调用（函数指针） *)
  | StaticFn of address         (* 静态调用 *)
  | Object of { obj; method_index; method_ty }  (* trait方法调用 *)
```

### 分配类型

```ocaml
type alloc_kind =
  | Tuple                          (* 元组 *)
  | Struct                         (* 结构体 *)
  | Enum of { tag : constr_tag }   (* 枚举 *)
  | Object of { methods }          (* 对象/方法表 *)
```

### Ltype_gc 类型

```ocaml
type t =
  | I32 of { kind : int_kind }   (* 32位整数，携带语义标记 *)
  | I64                          (* 64位整数 *)
  | F32                          (* 32位浮点 *)
  | F64                          (* 64位浮点 *)
  | Ref of { tid }               (* GC引用 *)
  | Ref_lazy_init of { tid }     (* 延迟初始化引用 *)
  | Ref_nullable of { tid }      (* 可空引用 *)
  | Ref_extern                   (* 外部引用 *)
  | Ref_string                   (* 字符串引用 *)
  | Ref_bytes                    (* 字节序列引用 *)
  | Ref_func                     (* 函数引用 *)
  | Ref_any                      (* 顶层any引用 *)
```

其中 `int_kind` 携带了紧凑的语义标签：
```ocaml
type int_kind =
  | I32_Int | I32_Char | I32_Bool | I32_Unit
  | I32_Byte | I32_Int16 | I32_UInt16
  | I32_Tag            (* 枚举构造器标签 *)
  | I32_Option_Char    (* Option<Char>的优化表示 *)
```

### Clam 的关键设计

1. **显式内存管理**：`Lallocate` 直接对应 Wasm GC 的 `struct.new`/`array.new`
2. **闭包展开**：`Lclosure { captures; address; tid }` 将闭包捕获的变量显式化
3. **无类型参数**：所有类型完全单态化，函数签名不含泛型
4. **Join点**：`Ljoinlet`/`Ljoinapply` 是对 Wasm `block`/`br` 的直接抽象
5. **trait对象**：`Object` 调用目标通过方法表索引进行动态分发

---

## 4. Dwarfsm IR (`dwarfsm_ast.ml`, 3,365行)

### 定位

Dwarfsm 是 Wasm 指令集的 OCaml AST 表示。它是 Clam → 二进制 Wasm 之间的编码层，由 `wasm_of_clam_gc.ml` 生成。此外，`dwarfsm_encode.ml` 负责将其序列化为二进制 `.wasm` 格式。

### 模块结构

```ocaml
type module_ = {
  id     : binder;
  fields : modulefield list;
}

type modulefield =
  | MType of rectype    (* 类型定义 *)
  | MFunc of func       (* 函数 *)
  | MTable of table     (* 表 *)
  | MMem of mem         (* 内存 *)
  | MGlobal of global   (* 全局变量 *)
  | MElem of elem       (* 元素段 *)
  | MData of data       (* 数据段 *)
  | MStart of funcidx   (* 入口 *)
  | MImport of import   (* 导入 *)
  | MExport of export   (* 导出 *)
```

### 指令集（部分）

Dwarfsm 指令直接对应 Wasm GC 的指令：

```ocaml
type instr =
  (* 控制流 *)
  | Block of { label; typeuse; instrs }
  | Br of labelidx | Br_if of labelidx
  | Br_table of labelidx list * labelidx
  | Return | Drop

  (* 函数调用 *)
  | Call of funcidx
  | Call_ref of typeidx
  | Call_indirect of tableidx * typeuse

  (* 整数操作 *)
  | I32_const of int32 | I64_const of int64
  | I32_add | I32_sub | I32_mul | I32_div_s | I32_div_u
  | I32_and | I32_or | I32_xor | I32_shl | I32_shr_s | I32_shr_u
  | I32_eq | I32_ne | I32_lt_s | I32_gt_s | I32_le_s | I32_ge_s
  | ...

  (* 浮点操作 *)
  | F64_add | F64_sub | F64_mul | F64_div | F64_const
  | ...

  (* GC操作 *)
  | Struct_new of typeidx | Struct_get of typeidx * fieldidx
  | Struct_set of typeidx * fieldidx
  | Array_new of typeidx | Array_new_fixed of typeidx * int32
  | Array_get of typeidx | Array_set of typeidx
  | Array_len
  | Ref_cast of reftype | Ref_test of reftype
  | Br_on_cast of labelidx * reftype * reftype
  | Br_on_cast_fail of labelidx * reftype * reftype

  (* 引用操作 *)
  | Ref_null of typeidx | Ref_is_null | Ref_func
  | Global_get of globalidx | Global_set of globalidx

  (* 内存操作 *)
  | I32_load of memarg | I64_load of memarg
  | I32_store of memarg | I64_store of memarg
  | ...

  (* externref *)
  | Any_convert_extern | Extern_convert_any
```

### Dwarfsm 的关键设计

1. **Wasm 语义直译**：每个 Dwarfsm 指令几乎都有一对一的 Wasm 指令对应
2. **可变引用**：`typeidx`、`funcidx` 等使用可变字段 `{ mutable var : var }` 以支持符号解析阶段的重写
3. **位置管理**：即使在此低级层仍保留 `Loc.t` 位置追踪用于调试
4. **优化模块**：`dwarfsm_elim_equivdefn.ml` 执行等价函数去重，`shrink_wasmir.ml` 执行 Wasm IR 瘦身

---

## IR 转换关系

```
Typedtree.output
    │ core_of_tast.ml
    ▼
Core.program ──── Core Passes ────→ Core.program (优化后)
    │ (DCE, LambdaLift, Contification, ...)
    │ core_link.ml (链接多个模块)
    ▼
Core_link.output
    │ monofy.ml (单态化)
    ▼
Mcore.t
    │ pass_layout.ml (布局优化)
    │ clam_of_core.ml
    ▼
Clam.prog
    │ pass_unused_let.ml
    │ wasm_of_clam_gc.ml
    ▼
Dwarfsm_ast.module_
    │ dwarfsm_elim_equivdefn.ml
    │ shrink_wasmir.ml
    │ dwarfsm_encode.ml
    ▼
.wasm 二进制
```

### 抽象级别递降表

| 特性 | Core | Mcore | Clam | Dwarfsm |
|------|------|-------|------|---------|
| 泛型 | ✓ | ✗ | ✗ | ✗ |
| 闭包 | 内隐 | 内隐 | 显式捕获 | 内联机器码 |
| 枚举 | 高阶构造器 | 高阶构造器 | 标签+分配 | GC struct |
| 循环 | for/while | for/while | loop+break | block+br |
| 内存 | GC自动 | GC自动 | GC显式分配 | GC指令 |
| 类型 | Stype | Mtype | Ltype_gc | rectype |
| 函数 | 闭包/函数名 | 闭包/函数名 | address/target | funcidx |
| FFI | stub | stub | Lstub_call | import |
