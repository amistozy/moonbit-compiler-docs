# 链接过程详解

MoonBit 的链接过程将多个包的 Core IR 合并为一个统一的程序，是单态化前的关键步骤。核心模块：`core_link.ml`（55行，非常紧凑）、`core_format.ml`。

---

## 链接在流水线中的位置

```
包A → build-package → A.core
包B → build-package → B.core
包C.mbt → compile → C (Core)
    │
    ▼
core_link.link(targets=[A.core, B.core, C])
    │
    ▼
Core_link.output { linked_program; types; methods; ext_methods }
    │
    ▼
monofy → Mcore.t
```

---

## 1. 链接输出结构

```ocaml
type output = {
  linked_program : Core.program;              (* 合并后的程序 *)
  methods : Method_env.t Hash_string.t;       (* 各包的方法环境 *)
  ext_methods : Ext_method_env.t Hash_string.t; (* 各包的扩展方法 *)
  types : Typing_info.types Hash_string.t;    (* 各包的类型定义 *)
}

type linking_target =
  | File_path of string           (* .core 文件路径 *)
  | Core_format of Core_format.t array  (* 内存中的 Core 数据 *)
```

---

## 2. 链接算法 (`core_link.ml`)

```ocaml
let link ~(targets : linking_target Vec.t) =
  (* 1. 创建空的类型/方法/扩展方法表 *)
  let types = Hash_string.create 17 in
  let methods = Hash_string.create 17 in
  let ext_methods = Hash_string.create 17 in

  (* 2. 对每个目标 *)
  let append_core (serialized : Core_format.t) acc =
    (* 注册该包的类型 *)    Hash_string.add types    pkg_name types;
    (* 注册该包的方法 *)    Hash_string.add methods   pkg_name methods;
    (* 注册该包的扩展方法 *) Hash_string.add ext_methods pkg_name ext_methods;
    (* 追加程序项 *)        List.rev_append serialized.program acc
  in

  (* 3. 遍历所有目标，累积 *)
  let items_rev = Vec.fold_left ~f:(fun items_acc target ->
    let pkgs = match target with
      | File_path path  → Core_format.import ~path
      | Core_format pkgs → pkgs in
    Array.fold_left append_core items_acc pkgs
  ) [] targets
  in

  (* 4. 产出 *)
  { linked_program = List.rev items_rev; types; methods; ext_methods }
```

### 算法复杂度

链接是线性的 O(n)：
- 按顺序遍历所有目标
- 每个目标的 program items 被追加到总列表
- 类型/方法表按包名索引

---

## 3. Core 格式 (`core_format.ml`)

### 序列化结构

```ocaml
type serialized = {
  program : Core.program;                        (* Core IR 程序 *)
  types : (string * Typedecl_info.t) array;       (* 类型定义 *)
  traits : (string * Trait_decl.t) array;          (* trait 声明 *)
  methods : method_array;                          (* 方法实现 *)
  ext_methods : ext_method_array;                  (* 扩展方法 *)
  pkg_name : string;                               (* 包名 *)
}
```

### 序列化/反序列化

```ocaml
(* 导出为文件 *)
val export : action:write_action -> pkg_name:string 
    -> program:Core.program -> genv:Global_env.t -> unit

(* 从文件导入 *)
val import : path:string -> serialized array

(* 从字符串反序列化 *)
val of_string : string -> serialized array
```

### Bundle 操作

```ocaml
(* moonc bundle-core: 合并多个 .core 文件 *)
val bundle : inputs:string list -> path:string -> unit
```

---

## 4. Core 二进制格式

`.core` 文件使用 S-Expression 格式存储：

```
(core
  (pkg_name "username/hello/lib")
  (types ...)
  (traits ...)
  (methods ...)
  (ext_methods ...)
  (program
    (Ctop_fn ...)
    (Ctop_let ...)
    ...
  ))
```

使用 `moon_sexp_conv.ml` 和 `w.ml` 进行序列化。

---

## 5. 链接后的单态化

链接产出 `Core_link.output` 后，立即进行单态化：

```ocaml
(* driver_util.ml *)
let monofy_core_link ~link_output ~exported_functions =
  let monofy_env = Monofy_env.make 
    ~regular_methods:link_output.methods
    ~extension_methods:link_output.ext_methods in
  let mono_core = Monofy.monofy ~monofy_env 
    ~stype_defs:link_output.types
    ~exported_functions link_output.linked_program in
  let mono_core = Pass_layout.optimize_layout mono_core in
  mono_core
```

单态化时需要访问所有链接包的类型和方法信息（通过 `Monofy_env`），以便解析跨包的泛型实例化。

---

## 6. 包名到符号的分辨

链接后的程序仍然使用带包名的限定标识符（`Pdot qual_name`）：

```ocaml
type ident = Pident of string | Pdot of Qual_ident.t | ...
```

在单态化阶段，`Pdot qual_name` 通过工作列表解析为具体的单态化标识符。

---

## 7. 链接顺序

链接顺序遵循目标输入顺序：

```
[main.core, libA.core, libB.core]
    → program = libB.items @ libA.items @ main.items (reverse)
    → linked_program = rev → main.items ++ libA.items ++ libB.items
```

这意味着 `main` 的顶层定义最先出现，然后是依赖库。对于全局初始化代码，这个顺序保证拓扑排序的正确性。

---

## 8. 编译场景的链接

### build-package（单包）

```
build-package → 产出单包的 .core + .mi
不需要跨包链接
```

### compile（单文件 + 依赖）

```ocaml
(* moon0_main.ml: compile 子命令 *)
let targets = Basic_vec.empty () in
(* 添加额外依赖的 .core *)
List.iter extra_deps ~f:(fun filename ->
  Basic_vec.push targets (Core_link.File_path filename));
(* 添加当前文件编译的 Core *)
Basic_vec.push targets (Core_link.Core_format [| current_core |]);
(* 链接 *)
let link_output = Core_link.link ~targets in
```

### link-core（多包链接）

```ocaml
(* moon0_main.ml: link-core 子命令 *)
let targets = Basic_vec.empty () in
(* 收集所有输入的 .core *)
List.iter input_files ~f:(fun file ->
  Basic_vec.push targets (Core_link.File_path file));
(* 链接 → 单态化 → 生成 .wasm *)
```

---

## 9. 跨包引用解析

链接后的类型查找路径：

```ocaml
type All_types = {
  toplevel : ...;   (* 当前包的类型 *)
  builtin : ...;    (* 内建类型 *)
  type_alias : ...; (* 类型别名 *)
  pkgs : Pkg.pkg_tbl; (* 依赖包 *)
}
```

链接时，各包的 types/methods/ext_methods 被存储为 `pkg_name → data` 的映射。单态化时通过 `Monofy_env` 查找跨包的 trait 实现和泛型实例。
