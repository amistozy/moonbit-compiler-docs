# Derive 宏机制详解

MoonBit 的 `derive` 是一种编译时元编程机制，允许自动为类型生成 trait 实现。与 Rust 的 `#[derive(...)]` 类似但使用函数调用语法。相关模块：`derive.ml`、`derive_args.ml`、`ast_derive.ml`。

---

## Derive 语法

```mbt
enum Color {
  Red
  Green
  Blue
} derive(Show, Eq, Hash, ToJson, Debug)

struct Point {
  x : Int
  y : Int
} derive(Show, Debug)
```

语法：`derive(Trait1, Trait2, ...)` 在类型声明之后。

---

## 1. Derive 解析

### 语法 AST

```ocaml
(* parsing_syntax.ml *)
type deriving_directive = {
  type_name_ : type_name;       (* trait 名 *)
  args : argument list;          (* derive 参数 *)
  loc_ : Rloc.t;
}
```

### 解析过程

在 `parsing_main.ml` 中，`parse_deriving_directive_list` 解析 `derive(...)` 块：

```ocaml
(* derive(Show, Eq, Hash(foo)) *)
→ [
  { type_name_ = Show; args = [] };
  { type_name_ = Eq; args = [] };
  { type_name_ = Hash; args = [foo] };
]
```

---

## 2. Derive 类型检查 (`derive.ml`)

### 主入口

```ocaml
val generate_signatures :
  types:Global_env.All_types.t ->
  ext_method_env:Ext_method_env.t ->
  trait_impls:Trait_impl.t ->
  Typedecl_info.t ->              (* 宿主类型 *)
  Syntax.type_name ->              (* trait 名 *)
  Trait_decl.t ->                  (* trait 的声明 *)
  diagnostics:Local_diagnostics.t ->
  loc:Rloc.t ->
  unit
```

### 处理流程

1. **查找 trait 声明**：通过 `trait_name` 在 `Global_env` 中查找对应的 `Trait_decl`
2. **计算约束闭包**：`Trait_closure.compute_closure` 获取所有父 trait
3. **生成方法签名**：为 trait 的每个方法生成对应的扩展方法注册
4. **注册 impl**：`Trait_impl.add_impl` 将实现注册到 `trait_impls`
5. **检查重复**：如果已有显式 impl，报错误

### 排除内建方法

```ocaml
(* 不自动生成 Hash 的 hash 方法和 Show 的 to_string *)
if not (trait = Builtin.trait_hash && method_name = "hash")
   && not (trait = Builtin.trait_show && method_name = "to_string")
then add_method ...
```

`Hash` 和 `Show` 的某些方法有特殊的编译器内建实现（通过 `Pintrinsic`），因此不生成 derive 签名。

### 生成的方法

```ocaml
add_method {
  id = Qual_ident.ext_meth ~trait ~self_typ:decl.ty_constr ~name:method_name;
  prim = None;               (* derive的方法没有prim实现 *)
  typ = impl_ty;              (* 实例化后的方法类型 *)
  pub;
  doc_ = Docstring.make [ "automatically derived" ];
  ty_params_ = impl_params;
  kind_ = Method_explicit_self { self_ty };
  ...
}
```

---

## 3. Derive 参数 (`derive_args.ml`)

部分 derive 接受参数：

```mbt
enum Status {
  Active
  Inactive
} derive(ToJson(content="active", content="inactive"))
```

### 参数提取

```ocaml
(* 提取字符串参数 *)
val extract_string : Syntax.expr -> (string, string) result

(* 提取常量参数 *)
val extract_constant : Syntax.expr -> (Constant.t, string) result

(* 拒绝所有参数 *)
val deny_all_args : host_type:string -> trait_name:Longident.t
    -> diag -> Syntax.deriving_directive -> bool
```

### 错误处理

```ocaml
val mk_diag_emitter :
  host_type:string -> trait_name:Basic_longident.t ->
  Local_diagnostics.t -> string -> Rloc.t -> unit
```

当 derive 参数不合法时，生成诊断：
```
Error: Cannot derive `MyTrait` for type `MyType`:
  MyTrait does not accept any arguments.
```

---

## 4. AST Derive (`ast_derive.ml`)

处理与语法树推导相关的 declare 机制（用于 spec-driven development）：

```mbt
// spec.mbt: 声明 trait 和类型
declare pub trait Eq {
  eq(Self, Self) -> Bool
}

// 后续：derive 自动实现
```

---

## 5. Derive 的执行时机

Derive 在类型检查的**第一遍**（`toplevel_typer.ml` 的符号注册阶段）执行：

```
类型声明解析:
  struct Point { x: Int; y: Int } derive(Show)
    ↓
1. 注册 Point 类型到 Global_env
2. 检查 derive(Show) 指令
3. 调用 derive.generate_signatures:
   - 查找 Show trait 声明
   - 为 to_string 方法注册扩展方法签名
   - 注册 Point 的 Show impl
```

此后，`Show::to_string` 对 `Point` 的调用在类型检查的第二遍中被正常处理。

---

## 6. 支持的 Derive

| Derive | trait | 说明 |
|--------|-------|------|
| `Show` | 自动生成 `to_string` 实现 | 类型名 + 字段 |
| `Eq` | 自动生成 `eq` 实现 | 结构相等 |
| `Hash` | 自动生成 `hash` 实现 | 哈希值 |
| `Debug` | 自动生成调试输出 | 含字段名 |
| `ToJson` | 自动生成 JSON 序列化 | 递归 |
| `Default` | 自动生成默认值 | 零值 |

---

## 7. Derive 的实现在哪里？

Derive `generate_signatures` 只是生成了**方法签名**（即声明了 "这个类型实现了这个trait"），实际的方法**实现**是在标准库中通过编译器内建（`Pintrinsic`）提供的。

例如，`derive(Show)` 生成的方法签名指向 `%char.to_string` 等 intrinsic 函数，编译器在代码生成阶段将这些 intrinsic 映射为高效的 Wasm 指令序列。

---

## 8. 与 Rust derive 对比

| | MoonBit | Rust |
|---|--------|------|
| 语法 | `derive(Trait1, Trait2)` | `#[derive(Trait1, Trait2)]` |
| 参数 | `derive(ToJson(content="n"))` | 属性参数 |
| 实现 | 编译器内建 + 方法签名 | 宏展开为完整实现 |
| 时机 | 类型检查第一遍 | 宏展开阶段 |
| 自定义derive | 不可自定义 | 可自定义 proc-macro |
