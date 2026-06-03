# 单态化（Monomorphization）详解

单态化是 MoonBit 编译器从泛型 Core IR 到具体 Mcore IR 的关键转换。相关模块：`monofy.ml`（437行）、`monofy_analyze.ml`、`monofy_env.ml`、`monofy_worklist.ml`、`monofy_instances.ml`。

---

## 什么是单态化

单态化将泛型/多态代码展开为具体类型的特化版本。MoonBit 没有运行时泛型——所有泛型在编译期完全展开。

```mbt
// 泛型函数
pub fn id[T](x : T) -> T { x }

// 使用
let a = id(42)       // id[Int]
let b = id("hello")  // id[String]
```

编译后被展开为两个独立的单态函数：
```
fn id_Int(x: Int) -> Int { x }
fn id_String(x: String) -> String { x }
```

---

## 单态化流水线

```
Core.program (含泛型)
    │
    ▼
┌──────────────────────────────────────┐
│ 1. monofy_analyze.ml                 │
│    分析阶段：遍历 Core IR，收集        │
│    所有需要的泛型实例                  │
│    产出: Worklist (实例集合)          │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│ 2. monofy.ml (monofy_generate)       │
│    生成阶段：为每个实例化生成单态代码    │
│    产出: Mcore.top_item list          │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│ 3. Type_subst (type_subst.ml)        │
│    类型替换引擎                         │
│    Tparam → 具体 Mtype.t              │
└──────────────────────────────────────┘
    │
    ▼
Mcore.t
```

---

## 1. 分析阶段 (`monofy_analyze.ml`)

### 工作列表算法

```ocaml
val monofy_analyze :
  Core.program ->
  Monofy_env.t ->
  stype_defs ->
  mtype_defs ->
  exported_functions ->
  Worklist.t
```

### 分析过程

1. **初始种子**：从 `exported_functions` 和 `main` 函数开始
2. **遍历 Core IR**：对每个表达式：
   - `Cexpr_var { id = Pdot qual_name; ty_args }` → 记录实例化 `qual_name[ty_args]`
   - `Cexpr_apply { func = Pdot name; ty_args }` → 记录实例化 `name[ty_args]`
   - `Plocal_method` → 记录 trait 方法实例化
3. **依赖跟踪**：`worklist.add_dependency(src, tgt)` 建立调用图
4. **闭包传播**：遇到新实例时，递归分析其引用到的其他泛型符号

### 工作列表项

```ocaml
type analyzed_item = {
  qual_name : Qual_ident.t;    (* 泛型符号的限定名 *)
  types : Type_args.t;          (* 具体类型参数 *)
  binder : Ident.t;             (* 生成的单态化标识符 *)
}
```

---

## 2. 生成阶段 (`monofy.ml`)

### 类型替换

```ocaml
let monofy_typ (env : Type_subst.t) (ty : Stype.t) : Stype.t =
  (* 将 Tparam index 替换为 env 中的具体类型 *)
  ...
```

### 单态化上下文

```ocaml
type monofy_ctx = {
  subst_env : Type_subst.t;            (* 类型替换映射 *)
  stype_defs : Typing_info.stype_defs; (* 原始类型定义 *)
  mtype_defs : Mtype.defs;             (* 目标类型定义（累积） *)
  rename_ctx : rename_ctx;             (* 标识符重命名表 *)
  worklist : Worklist.t;               (* 工作列表 *)
  monofy_env : Monofy_env.t;           (* 方法查找环境 *)
}
```

### 表达式单态化

`monofy_obj` 是一个 visitor，对每个 Core.expr 执行：
1. 将所有 `Stype.t` 类型替换为具体类型
2. 将泛型函数引用 `Pdot qual_name` 替换为具体实例的标识符
3. 将 trait 方法调用 `Plocal_method` 解析为具体方法
4. 将 `Cexpr_as` (trait object 构造) 翻译为 `Cexpr_object`

### 关键代码

```ocaml
method visit_Cexpr_var ctx id ty ty_args prim loc_ =
  match id with
  | Pdot qual_name when not (Array.is_empty ty_args) ->
      (* 泛型函数引用 → 查找或创建具体实例 *)
      let tys = Array.map ty_args (monofy_typ ctx.subst_env) in
      let new_binder = Worklist.find_new_binder_exn ctx.worklist qual_name tys in
      Mcore.var ~ty:mtype new_binder ~prim
  | Plocal_method { index; trait; method_name } ->
      (* Trait 方法调用 → 解析到具体方法 *)
      let type_name = Type_subst.monofy_param ctx.subst_env ~index in
      match generate_trait_method type_name ... with
      | `Prim prim → Mcore.prim prim args
      | `Method (func, prim) → Mcore.apply func args
  ...
```

### 函数声明单态化

```ocaml
let generate_fun_decl (fd : Core.top_fun_decl) =
  match fd.binder with
  | Pdot qual_name ->
      match Worklist.find_analyzed_items wl qual_name with
      | [] -> []  (* 未使用，移除 *)
      | items ->
          List.map items ~f:(fun item ->
            let env = Type_subst.make item.types in
            let func = generate_fn fd.func env in    (* 生成单态函数体 *)
            Ctop_fn { binder = item.binder; func; ... })
```

每个泛型函数的每个实例化组合都生成一个独立的顶层函数。

---

## 3. 类型替换引擎 (`type_subst.ml`)

```ocaml
type t  (* 替换映射：Tparam index → Stype.t *)

val make : Type_args.t -> t            (* 构造替换 *)
val monofy_typ : t -> Stype.t -> Stype.t    (* 替换类型 *)
val monofy_param : t -> index:int -> Type_path.t  (* 替换类型参数 *)
```

---

## 4. 工作列表 (`monofy_worklist.ml`)

```ocaml
type t  (* 实例的DAG *)

(* 添加/查找实例 *)
val add_value_if_not_exist : t -> Qual_ident.t -> Type_args.t -> Ident.t

(* 查找已分析实例 *)
val find_analyzed_items : t -> Qual_ident.t -> analyzed_item list

(* 获取新binder *)
val find_new_binder_exn : t -> Qual_ident.t -> Type_args.t -> Ident.t

(* 依赖管理 *)
val add_dependency : t -> Ident.t -> Ident.t -> unit

(* 拓扑排序全局变量初始化 *)
val order_globals : t -> globals -> ordered_globals
```

---

## 5. 全局变量处理

单态化后的全局变量需要按依赖顺序初始化：

```ocaml
(* 拓扑排序确保初始化顺序 *)
val order_globals : Worklist.t -> globals -> ordered_globals

(* let x = f[Int]()  →  f_Int 必须先生成 *)
```

---

## 6. error_to_string 的特殊处理

MoonBit 编译器为 `Error.to_string()` 自动生成一个 switch 函数：

```ocaml
let body =
  if Worklist.get_used_error_to_string wl then
    Worklist.make_error_to_string wl ~tags:mtype_defs.ext_tags :: body
  else body
```

该函数根据错误类型的 tag 生成字符串表示。

---

## 7. 完整示例

```mbt
pub fn pair[T](x : T, y : T) -> (T, T) {
  (x, y)
}

fn main {
  let a = pair(1, 2)       // pair[Int]
  let b = pair("a", "b")   // pair[String]
}
```

### 分析阶段产出

```
Worklist {
  pair[Int]    → binder: pair_Int_0
  pair[String] → binder: pair_String_1
}
```

### 生成阶段产出 (Mcore)

```ocaml
(* pair_Int_0 *)
Ctop_fn {
  binder = pair_Int_0;
  func = {
    params = [{binder=x; ty=T_int}, {binder=y; ty=T_int}];
    body = Cexpr_tuple([x, y], ty=T_tuple[T_int, T_int])
  }
}

(* pair_String_1 *)
Ctop_fn {
  binder = pair_String_1;
  func = {
    params = [{binder=x; ty=T_string}, {binder=y; ty=T_string}];
    body = Cexpr_tuple([x, y], ty=T_tuple[T_string, T_string])
  }
}

(* main *)
Ctop_fn {
  binder = main;
  func = {
    body =
      let a = pair_Int_0(1, 2) in
      let b = pair_String_1("a", "b") in
      ...
  }
}
```

---

## 8. Trait方法的单态化

```mbt
pub fn show_it[T : Show](x : T) -> String {
  @Show.to_string(x)
}
```

类型检查后 Core IR 中为：
```
Plocal_method { index=0; trait=Show; method_name="to_string" }
```

单态化时：
1. 从 `Monofy_env` 查找 `Show.to_string` 对具体类型 `T` 的实现
2. 如果实现有 `prim` 实现 → 直接生成 prim 调用
3. 否则 → 生成对具体方法的静态调用

---

## 9. 代码膨胀控制

单态化可能导致代码膨胀（每个实例化组合都生成一份代码）。MoonBit 的控制策略：

1. **死代码消除**：未使用的泛型实例不生成（在分析阶段过滤）
2. **Small types 内联**：小类型（如 Int、Bool）的泛型参数在后续 Pass 中可能被优化合并
3. **Trait prim**：有直接 Wasm 基元实现的 trait 方法不生成额外代码
