# 子类型检查与类型等价判定

MoonBit的类型系统基于**结构等价**和**Unification**，而非传统面向对象语言的标称子类型。子类型关系主要通过Trait系统实现，类型检查器使用Robinson风格的合一算法。

---

## 1. 类型表示：`Stype.t`

`stype.ml` 定义了编译器内部的类型表示：

```ocaml
type t =
  | Tarrow of {
      params_ty : t list;       (* 参数类型列表 *)
      ret_ty : t;               (* 返回类型 *)
      err_ty : t option;        (* 错误类型（None=无错误） *)
      is_async : bool;          (* 是否为异步函数 *)
      generic_ : bool;          (* 是否包含泛型参数 *)
    }
  | T_constr of {
      type_constructor : Type_path.t;  (* 类型构造函数路径 *)
      tys : t list;                    (* 类型参数 *)
      generic_ : bool;
      is_suberror_ : bool;             (* 是否为Error的子错误 *)
    }
  | Tvar of tlink ref          (* 类型变量（统一变量） *)
  | Tparam of { index : int; name_ : string }  (* 泛型参数 *)
  | T_trait of Type_path.t     (* Trait对象类型 *)
  | T_builtin of builtin       (* 内建类型 *)
  | T_blackhole                (* 类型黑洞（错误恢复） *)

and tlink = Tnolink of tvar_kind | Tlink of t  (* Union-Find链 *)
and tvar_kind = Tvar_normal | Tvar_error

type builtin =
  | T_unit | T_bool | T_byte | T_char | T_int | T_int16 | T_int64
  | T_uint | T_uint16 | T_uint64 | T_float | T_double | T_string | T_bytes
```

### 1.1 Tvar与Union-Find

类型变量使用经典的**Union-Find**（并查集）结构：

```ocaml
type tlink =
  | Tnolink of tvar_kind    (* 未被赋值的变量 *)
  | Tlink of t              (* 已链接到目标类型 *)

let rec type_repr (ty : t) =
  match ty with
  | Tvar { contents = Tlink ty } -> type_repr ty  (* 跟随链 *)
  | Tvar { contents = Tnolink _ } | Tarrow _ | T_constr _ | ... -> ty
```

`type_repr` 是类型系统的核心操作：它跟随 Union-Find 链找到变量的最终值。几乎所有类型操作都先用它来消除间接引用。

**路径压缩优化**：`type_repr`支持双重Tlink的压缩：

```ocaml
(* Tvar(link1) → Tvar(link2) → 实际类型 *)
| Tvar ({ contents = Tlink (Tvar ({ contents = Tlink ty } as a0)) } as a1) ->
    (* 压缩：a1直接指向最终类型 *)
    a0 := Tlink ty;
    a1 := Tlink ty;
    ty
```

---

## 2. 类型等价性：`Type.same_type`

MoonBit的类型等价是**结构等价**（structural equality）：

```ocaml
let rec same_type (ty1 : typ) (ty2 : typ) =
  let ty1 = type_repr ty1 in
  let ty2 = type_repr ty2 in
  phys_equal ty1 ty2  (* 快速路径：物理相等 *)
  ||
  match ty1 with
  | Tvar _ -> false    (* 未赋值的类型变量不相等 *)
  | Tarrow { params_ty = ps1; ret_ty = r1; err_ty = e1; is_async = a1 } ->
      (* 函数类型：参数、返回、错误、async标记都相等 *)
      match ty2 with
      | Tarrow { params_ty = ps2; ret_ty = r2; err_ty = e2; is_async = a2 } ->
          Lst.for_all2_no_exn ps1 ps2 same_type
          && same_type r1 r2 && a1 = a2
          && (match (e1, e2) with
              | Some e1, Some e2 -> same_type e1 e2
              | None, None -> true
              | _ -> false)
      | _ -> false
  | T_constr { type_constructor = p1; tys = tys1 } ->
      (* 构造器类型：路径相等 + 类型参数结构等价 *)
      match ty2 with
      | T_constr { type_constructor = p2; tys = tys2 } ->
          Type_path.equal p1 p2 && Lst.for_all2_no_exn tys1 tys2 same_type
      | _ -> false
  | Tparam { index = i1 } ->
      match ty2 with Tparam { index = i2 } -> i1 = i2 | _ -> false
  | T_trait t1 ->
      match ty2 with T_trait t2 -> Type_path.equal t1 t2 | _ -> false
  | T_builtin x ->
      match ty2 with T_builtin y -> Stype.equal_builtin x y | _ -> false
  | T_blackhole ->
      match ty2 with T_blackhole -> true | _ -> false
```

**关键特性**：
- `phys_equal`快速路径：编译过程中相同类型常被共享（通过hash-consing效果）
- 未赋值的`Tvar`不与任何类型相等（它是"待确定"的）
- **无标称等价**：两个不同名但结构相同的struct类型不相等
- `T_trait`通过`Type_path.equal`判定等价（标称等价）

---

## 3. Unification算法：`Ctype.unify`

`ctype.ml`实现了经典的Robinson合一算法：

```ocaml
exception Unify  (* 合一失败异常 *)

let rec unify (ty1 : typ) (ty2 : typ) =
  let ty1' = type_repr ty1 and ty2' = type_repr ty2 in
  if phys_not_equal ty1' ty2' then (
    match (ty1', ty2') with
    | Tvar link1, _ ->
        (* 变量统一：如果ty2中含有ty1则失败（occurs check） *)
        if check_occur ty1' ty2' then raise_notrace Unify
        else link1 := Tlink ty2'
    | _, Tvar link ->
        if check_occur ty2' ty1' then raise_notrace Unify
        else link := Tlink ty1'
    | T_blackhole, _ | _, T_blackhole -> ()
    | Tarrow { params_ty = ps1; ret_ty = r1; err_ty = e1; is_async = a1 },
      Tarrow { params_ty = ps2; ret_ty = r2; err_ty = e2; is_async = a2 } ->
        if a1 <> a2 then raise_notrace Unify;
        unify_list ps1 ps2;
        unify r1 r2;
        (match (e1, e2) with
         | None, None -> ()
         | Some e1, Some e2 -> unify e1 e2
         | _ -> raise_notrace Unify)
    | T_constr { type_constructor = c1; tys = tys1 },
      T_constr { type_constructor = c2; tys = tys2 } ->
        if Type_path.equal c1 c2 then unify_list tys1 tys2
        else raise_notrace Unify
    | Tparam { index = i1 }, Tparam { index = i2 } ->
        if i1 <> i2 then raise_notrace Unify
    | T_trait t1, T_trait t2 ->
        if not (Type_path.equal t1 t2) then raise_notrace Unify
    | T_builtin a, T_builtin b ->
        if not (Stype.equal_builtin a b) then raise_notrace Unify
    | _ -> raise_notrace Unify  (* 不同类型构造器 = 合一失败 *)
  )
```

### 3.1 Occurs Check

```ocaml
let check_occur (v : typ) (ty : typ) =
  let rec go ty =
    match ty with
    | Tarrow { params_ty; ret_ty; err_ty; _ } ->
        List.exists go params_ty || go ret_ty
        || (match err_ty with Some e -> go e | None -> false)
    | T_constr { tys; _ } -> List.exists go tys
    | Tvar { contents = Tlink ty' } -> go ty'
    | Tvar _ -> phys_equal v ty  (* v出现在ty中 *)
    | _ -> false
  in go ty
```

防止无限类型（如 `X = X -> X`）。

### 3.2 API层

Unification被包装为带诊断的API：

```ocaml
(* 表达式上下文的合一 *)
let unify_expr ~expect_ty ~actual_ty loc =
  try unify expect_ty actual_ty; None
  with Unify -> Some (Errors.expr_unify ~expected ~actual ~loc)

(* 模式匹配上下文的合一 *)
let unify_pat ~expect_ty ~actual_ty loc =
  try unify expect_ty actual_ty; None
  with Unify -> Some (Errors.pat_unify ~expected ~actual ~loc)
```

对Tvar_error变量做特殊处理：错误变量**优先合一到对方**（减少级联错误）：

```ocaml
| Tvar link1, Tvar link2 ->
    match !link1 with
    | Tnolink Tvar_error -> link2 := Tlink ty1'  (* 错误变量被覆盖 *)
    | _ -> link1 := Tlink ty2'
```

---

## 4. Trait子类型

MoonBit的**唯一子类型关系**通过Trait实现：

### 4.1 Trait作为上界

```mbt
trait Show {
  show(Self) -> String
}

// 任何实现Show的类型都可以看作Show
fn display(x : Show) { ... }  // x的类型是 T_trait(Show)
```

内部表示：`T_trait(Type_path)` 代表该trait的对象类型。

### 4.2 显式转换：`as`

```mbt
let x : @json.Json = ...
let s : Show = x as Show  // 构造Trait对象
```

在Typedtree中表示为`Texpr_as { expr; trait; ... }`，在Core IR中为`Cexpr_as { expr; trait; obj_type; ... }`。

### 4.3 Trait方法解析

```ocaml
(* type.ml *)
let is_error_type ~tvar_env (ty : Stype.t) =
  match type_repr ty with
  | T_constr { type_constructor = T_error; _ } -> true
  | T_constr { is_suberror_; _ } -> is_suberror_
  | Tparam { index } ->
      let info = Tvar_env.find_by_index_exn tvar_env index in
      Lst.exists info.constraints (fun c ->
          Type_path.equal c.trait Type_path.Builtin.type_path_error)
  | Tvar _ -> true      (* 未赋值的类型变量可能是任何类型 *)
  | T_blackhole -> true  (* 黑洞用于错误恢复 *)
  | _ -> false
```

类型参数可以带上trait约束（如`T: Error`），在编译时检查。

---

## 5. Error层次结构（Supererror）

MoonBit的`Error`类型使用**supererror**机制实现错误层次：

```mbt
suberror ParseError { ... }
suberror IOError { ... }
```

内部实现：

```ocaml
(* Stype.t中： *)
| T_constr { is_suberror_ : bool; ... }

(* type.ml: *)
let is_suberror (ty : Stype.t) =
  match type_repr ty with
  | T_constr { is_suberror_; _ } -> is_suberror_
  | _ -> false

let is_super_error (ty : Stype.t) =
  match type_repr ty with
  | T_constr { type_constructor = Basic_type_path.T_error; _ } -> true
  | _ -> false
```

`ParseError`可以隐式转换为`Error`（supererror），因为`is_suberror_ = true`。

---

## 6. 类型相关辅助函数

### 6.1 类型分类

```ocaml
let classify_as_builtin (t : Stype.t) =
  match type_repr t with
  | T_builtin T_int -> `Int    | T_builtin T_int64 -> `Int64
  | T_builtin T_uint -> `UInt  | T_builtin T_uint64 -> `UInt64
  | T_builtin T_float -> `Float | T_builtin T_double -> `Double
  | T_builtin T_char -> `Char  | T_builtin T_byte -> `Byte
  | T_builtin T_int16 -> `Int16 | T_builtin T_uint16 -> `UInt16
  | _ -> `Other
```

用于字面量重载（确定无类型整数字面量的具体类型）。

### 6.2 结构类型解构

```ocaml
(* 解构构造器类型：Option(Tag) → (Option, Some tag/None) *)
let deref_constr_type (t : Stype.t) =
  match type_repr t with
  | T_constr { type_constructor = Constr { ty; tag }; tys; ... } ->
      (T_constr { type_constructor = ty; tys; ... }, Some tag)
  | ty -> (ty, None)

(* 构造构造器类型 *)
let make_constr_type (t : Stype.t) ~tag =
  match type_repr t with
  | T_constr { type_constructor; tys; ... } ->
      T_constr { type_constructor = Type_path.constr ~ty:type_constructor ~tag; ... }
  | ty -> ty
```

用于模式匹配中构造器的类型推导。

### 6.3 数组类型判断

```ocaml
let is_array_like ty =
  match type_repr ty with
  | T_constr { type_constructor; _ } ->
      Type_path.equal type_constructor Type_path.Builtin.type_path_fixedarray
      || Type_path.equal type_constructor Type_path.Builtin.type_path_array
      || Type_path.equal type_constructor Type_path.Builtin.type_path_arrayview
  | T_builtin T_bytes -> true
  | ...
```

---

## 7. 类型系统的整体流程

```
源码中的类型标注 + 表达式
    │
    ▼
typer.ml（类型推断/检查）
    │
    ├── 创建Tvar (类型变量)
    ├── 收集约束（expect_ty vs actual_ty）
    ├── Ctype.unify（合一）
    │   ├── check_occur（防止无限类型）
    │   ├── Tvar赋值（Union-Find链）
    │   └── 失败时生成类型错误诊断
    │
    ├── 约束求解（trait bounds检查）
    │
    ▼
Typedtree（完整类型化AST，所有类型确定）
    │
    ▼
后续编译阶段使用 type_repr + same_type 判定类型
```

---

## 8. 与Wasm GC类型系统的关系

MoonBit的Stpye系统最终映射到Wasm GC的线性类型：

```
Stype.t                      Wasm GC type
─────────────────────────────────────────
T_builtin T_int       →      i32
T_builtin T_int64     →      i64
T_builtin T_double    →      f64
T_builtin T_float     →      f32
T_builtin T_unit      →      i32 (编码为0)
T_builtin T_bool      →      i32
T_builtin T_string    →      (ref $moonbit.string)
T_constr { tid }      →      (ref $tid)    (* struct/array类型 *)
T_trait trait         →      (ref any)     (* 运行时trait对象 *)
T_trait trait         →      i31/i32       (* 单方法trait优化 *)
```

在Wasm GC层面，子类型由`ref.cast`/`br_on_cast`指令实现，利用Wasm GC的标称子类型（struct/array继承链）。

---

## 与 `type-system.md` 的关系

`type-system.md` 聚焦于**Stype/Mtype/Ltype四层类型架构**和它们在编译器各阶段的表现形式。本文档聚焦于类型的**等价性判定**（same_type）、**合一算法**（unify）和**子类型关系**（trait/supererror），覆盖类型检查器的核心算法。
