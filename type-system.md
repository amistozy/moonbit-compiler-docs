# 类型系统详解

MoonBit 的类型系统贯穿编译流水线的多个层次，从类型检查阶段的高级语义类型到代码生成阶段的低级 Wasm GC 类型，共有四层类型表示。

---

## 类型层次总览

```
类型检查层            IR层（高级）         IR层（低级）        代码生成层
   Stype.t  ──────→  Stype.t  ──────→  Mtype.t  ──────→  Ltype_gc.t
   (含Tvar,           (含Tvar,           (全具体,           (Wasm GC映射)
    Tparam,            封闭后)           无类型变量)
    双向检查用)
    ↓                   ↓
   ctype.ml           core.ml           mcore.ml           clam.ml
   (unification)      (Core IR)         (Mcore IR)         (Clam IR)
```

---

## 1. Stype.t — 结构类型（类型检查层）

**定义位置**：`stype.ml`

`Stype` 是 MoonBit 类型检查阶段的核心类型表示，支持类型变量（用于 unification）和泛型参数。

```ocaml
type builtin =
  | T_unit | T_bool | T_byte | T_char
  | T_int | T_int16 | T_int64
  | T_uint | T_uint16 | T_uint64
  | T_float | T_double
  | T_string | T_bytes

type t =
  | Tarrow of {           (* 函数类型 *)
      params_ty : t list;  (* 参数类型列表 *)
      ret_ty : t;          (* 返回类型 *)
      err_ty : t option;   (* 错误类型 *)
      is_async : bool;     (* 是否异步 *)
      generic_ : bool;     (* 是否泛型 *)
    }
  | T_constr of {              (* 命名构造器类型 *)
      type_constructor : Type_path.t;
      tys : t list;             (* 类型参数 *)
      generic_ : bool;
      is_suberror_ : bool;      (* 是否是某个error的子类型 *)
    }
  | Tvar of tlink ref           (* 类型变量（unification用） *)
  | Tparam of { index : int; name_ : string }  (* 泛型参数 *)
  | T_trait of Type_path.t      (* trait类型 *)
  | T_builtin of builtin         (* 内建基础类型 *)
  | T_blackhole                  (* 占位黑洞 *)
```

### 类型变量的 unfification

```ocaml
and tlink =
  | Tnolink of tvar_kind   (* 未绑定 *)
  | Tlink of t             (* 已绑定到某类型 *)

and tvar_kind =
  | Tvar_normal   (* 普通类型变量 *)
  | Tvar_error    (* 错误传播型变量 *)
```

`Tvar` 是 unification 的核心机制。类型变量初始为 `Tnolink`，通过 unification 被链接 (`Tlink`) 到具体类型。`Tvar_error` 标记用于在类型错误发生时优雅降级而非即刻失败。

### Stype 在 IR 层的复用

`Core.ml` 中 `type typ = Stype.t`，因此 Core IR 也使用 `Stype`。在类型检查完成后，`Stype` 中的 `Tvar` 应全部被链接（closed），但仍可能包含 `Tparam`（因为 Core 仍然支持泛型）。

---

## 2. Ctype — 约束类型（unification 引擎）

**定义位置**：`ctype.ml`

`ctype.ml` 定义了类型间的关系，核心是 `unify` 函数。它不是新的类型定义，而是对 `Stype.t` 的操作。

```ocaml
type typ = Stype.t

exception Unify

val unify : typ -> typ -> unit
```

### unification 规则

```ocaml
let rec unify ty1 ty2 =
  match type_repr ty1, type_repr ty2 with
  | Tvar link1, _ -> link1 := Tlink ty2'      (* 类型变量绑定 *)
  | _, Tvar link -> link := Tlink ty1'
  | T_blackhole, _ | _, T_blackhole -> ()      (* 黑洞忽略 *)
  | Tarrow a, Tarrow b ->                       (* 结构匹配 *)
      unify a.ret_ty b.ret_ty;
      unify_list a.params_ty b.params_ty;
      ...
  | T_constr a, T_constr b ->                   (* 命名匹配 *)
      if Type_path.equal c1 c2 then unify_list a.tys b.tys
  | T_builtin a, T_builtin b ->                 (* 内建匹配 *)
      if equal_builtin a b then ()
  | _ -> raise Unify                            (* 其他情况失败 *)
```

辅助模块：
- `constraint_cache.ml` — 缓存已解析的类型约束
- `type_constraint.ml` — 类型约束定义（泛型参数的 trait bound）
- `poly_type.ml` — 多态类型（泛型函数的类型方案）
- `type_subst.ml` — 类型替换（将 Tparam 替换为具体类型）
- `tvar_env.ml` — 类型变量环境

---

## 3. Mtype.t — 单态化类型（Mcore 层）

**定义位置**：`mtype.ml`

`Mtype` 是单态化后的类型系统，用于 Mcore IR。与 Stype 的关键区别是**不存在类型变量**——所有 `Tvar`/`Tparam` 已被消解为具体类型。

```ocaml
type t =
  | T_int | T_char | T_bool | T_unit | T_byte
  | T_int16 | T_uint16
  | T_int64 | T_uint | T_uint64
  | T_float | T_double
  | T_string | T_bytes
  | T_optimized_option of { elem : t }  (* Option的内联优化表示 *)
  | T_func of { params : t list; return : t }
  | T_raw_func of { params : t list; return : t }
  | T_tuple of { tys : t list }
  | T_fixedarray of { elem : t }
  | T_constr of id                         (* 具名构造器类型 *)
  | T_trait of id                          (* 具名trait *)
  | T_any of { name : id }                 (* 存在类型 *)
  | T_maybe_uninit of t                    (* 未初始化内存 *)
  | T_error_value_result of { ok : t; err : t; id : id }  (* Result! *)
```

### 关键新增类型

- **T_optimized_option**：MoonBit 将 `Option<T>` 用小整数范围优化表示（`None=0`, `Some`=1+），避免堆分配
- **T_error_value_result**：MoonBit 的错误类型（`Result!`），将 `ok`/`err` 视为带标签的 sum type
- **T_maybe_uninit**：标记延迟初始化的变量
- **T_raw_func**：与 `T_func` 区别在于 `T_raw_func` 表示未装箱的原始函数指针（用于 FFI）

### 类型信息

```ocaml
type info =
  | T_info_enum of { constrs : constr_info list }    (* 枚举 *)
  | T_info_record of { fields : field_info list }    (* 结构体 *)
  | T_info_newtype of t                                 (* newtype *)
  | T_info_abstract                                      (* 抽象类型 *)

type defs = { defs : info Id_hash.t; ext_tags : int Hash_string.t }
```

这些 `defs` 携带了每个具名类型的具体结构信息（字段布局、枚举标签等），是 `Mtype` → `Ltype_gc` 翻译的关键依据。

---

## 4. Ltype.t / Ltype_gc.t — 低级类型（Clam 层→Wasm GC）

**定义位置**：`ltype.ml`（非GC版本）和 `ltype_gc.ml`（GC版本）

`Ltype` 直接映射 Wasm 的数据类型。当前公开仓库使用 `Ltype_gc`。

```ocaml
(* ltype_gc.ml *)
type int_kind =
  | I32_Int | I32_Char | I32_Bool | I32_Unit
  | I32_Byte | I32_Int16 | I32_UInt16
  | I32_Tag            (* 枚举构造器标签 *)
  | I32_Option_Char    (* Option<Char>的紧凑编码 *)

type t =
  (* 值类型 *)
  | I32 of { kind : int_kind }    (* 32位整数，带语义标记 *)
  | I64                            (* 64位整数 *)
  | F32                            (* 32位浮点 *)
  | F64                            (* 64位浮点 *)

  (* 引用类型（GC管理） *)
  | Ref of { tid : Ty_ident.t }              (* 指向特定类型的GC引用 *)
  | Ref_lazy_init of { tid : Ty_ident.t }    (* 延迟初始化的引用 *)
  | Ref_nullable of { tid : Ty_ident.t }     (* 可空GC引用 *)
  | Ref_extern                              (* 外部引用 *)
  | Ref_string                              (* 字符串引用 *)
  | Ref_bytes                               (* 字节序列引用 *)
  | Ref_func                                (* 函数引用 *)
  | Ref_any                                 (* 顶层any引用 *)
```

### int_kind 的标签设计

Wasm GC 中 `i32` 是位级别的类型，不携带语义。`int_kind` 为编译器提供了额外的语义信息：

| 标记 | 对应的 MoonBit 类型 | 说明 |
|------|---------------------|------|
| `I32_Int` | `Int` | 有符号32位整数 |
| `I32_Char` | `Char` | Unicode码点 |
| `I32_Bool` | `Bool` | 布尔值 |
| `I32_Unit` | `Unit` | 单元类型 |
| `I32_Byte` | `Byte` | 无符号字节 |
| `I32_Int16` | `Int16` | 16位整数 |
| `I32_UInt16` | `UInt16` | 无符号16位整数 |
| `I32_Tag` | 构造器标签 | 枚举判别式 |
| `I32_Option_Char` | `Option<Char>?` | Char的特殊Option编码 |

### Ltype 函数签名

```ocaml
type fn_sig = {
  params : t list;
  ret : t list;   (* 多返回值（Wasm GC特性） *)
}

type def = ...
type type_defs = def Ty_ident.Hash.t
```

`fn_sig.ret` 是 `t list` 而非 `t`，支持 Wasm GC 的多返回值特性。

### ltype.ml vs ltype_gc.ml

| 特性 | ltype.ml | ltype_gc.ml |
|------|----------|-------------|
| U32/U64 | ✓ | ✗（用I32+I64+bounds check替代）|
| Raw_func | ✓ | ✗ |
| GC Ref类型 | 同 | 同 |
| 当前使用 | 非GC的Wasm后端（未公开） | **当前公共后端使用** |

---

## 类型翻译流程

### Stype → Mtype（单态化）

单态化过程消解所有类型变量：

```
Stype.t:
  Tarrow { params_ty=[Tparam{index=0}]; ret_ty=T_int; ... }
  → (给定实例化 Int64)
Mtype.t:
  T_func { params=[T_int64]; return=T_int }
```

所有 `Tvar` 在类型检查后已被全部链接。`Tparam` 在单态化时被具体类型替换。

### Mtype → Ltype_gc（低级化翻译）

`transl_mtype_gc.ml` 负责此转换：

```
Mtype.t 数值类型:
  T_int    → I32 { kind = I32_Int }
  T_int64  → I64
  T_double → F64
  T_char   → I32 { kind = I32_Char }
  ...

Mtype.t 引用类型:
  T_constr "User"  → Ref { tid = <generated_id> }
  T_string         → Ref_string
  T_trait "Show"   → Ref_func

Mtype.t 复合类型:
  T_tuple [A;B]    → (分配 + 字段类型)
  T_fixedarray A   → (Wasm GC array)
```

### int_kind 在指令选择中的作用

`int_kind` 影响整数操作的符号性选择：

```ocaml
(* 伪代码 *)
match int_kind with
| I32_Int | I32_Int16 →
    div → I32_div_s;  lt → I32_lt_s
| I32_UInt16 | I32_Byte →
    div → I32_div_u;  lt → I32_lt_u
| I32_Tag →
    eq → I32_eq
```

---

## 与 Wasm GC 类型系统的映射

| MoonBit 概念 | Mtype | Ltype_gc | Wasm GC |
|-------------|-------|----------|---------|
| Int | T_int | I32 {I32_Int} | i32 |
| Int64 | T_int64 | I64 | i64 |
| Double | T_double | F64 | f64 |
| struct | T_constr + T_info_record | Ref {tid} | (ref $tid) ⇒ struct |
| enum | T_constr + T_info_enum | Ref {tid} + I32_Tag | (ref $tid) ⇒ struct + i32 tag |
| 闭包 | T_func → T_constr | Ref {tid} | (ref $tid) ⇒ struct (captures+func) |
| String | T_string | Ref_string | (ref string) |
| Array | T_fixedarray | Ref + array | (ref (array ...)) |
| Unit | T_unit | I32 {I32_Unit} | i32 (常量0) |
| Option<T> | T_optimized_option | I32 {I32_Tag} + Ref_nullable | i32 + (ref null ...) |
| Result! | T_error_value_result | I32 {I32_Tag} + Ref | struct {tag; payload} |
