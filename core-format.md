# Core 文件格式详解

`.core` 文件是 MoonBit 编译器在 `build-package` 阶段产出的中间表示文件，包含优化后的 Core IR、类型信息和方法环境。它使用 S-Expression 格式存储，可以跨编译单元链接。核心模块：`core_format.ml`。

---

## 1. Core 文件结构

### 顶层格式

```lisp
;; .core 文件
(core
  (pkg_name "username/hello")
  (types ...)
  (traits ...)
  (methods ...)
  (ext_methods ...)
  (program ...))
```

### 类型信息（types）

```lisp
(types
  (("MyEnum" (Variant_type
    ((Variant "A" ((tag ...) (args ...)))
     (Variant "B" ((tag ...) (args ...)))))))
  (("MyStruct" (Record_type
    ((Field "x" ((ty ...) (pos 0) (mut false)))
     (Field "y" ((ty ...) (pos 1) (mut true))))))))
```

### Trait 声明（traits）

```lisp
(traits
  (("Show" ((name "Show")
            (supers ())
            (methods
              (("to_string" ((params ...) (return ...)))))))))
```

### 方法环境（methods）

```lisp
(methods
  (("to_string" ((id ...) (prim ...) (typ ...) (ty_params ...) ...))))
```

---

## 2. 序列化数据结构

```ocaml
type serialized = {
  program : Core.program;                        (* Core IR *)
  types : (string * Typedecl_info.t) array;       (* 类型定义 *)
  traits : (string * Trait_decl.t) array;          (* trait 声明 *)
  methods : method_array;                          (* 常规方法 *)
  ext_methods : ext_method_array;                  (* 扩展方法 *)
  pkg_name : string;                               (* 包全名 *)
}
```

---

## 3. 序列化/反序列化 API

```ocaml
(* 导出 Core 为文件 *)
val export :
  action:write_action ->
  pkg_name:string ->
  program:Core.program ->
  genv:Global_env.t ->
  unit

(* 从文件导入 Core *)
val import : path:string -> serialized array

(* 从字符串反序列化 *)
val of_string : string -> serialized array

(* Bundle 多个 .core 文件 *)
val bundle : inputs:string list -> path:string -> unit
```

### 导出流程

```
build-package:
  1. 类型检查 → Global_env
  2. Core IR 生成 + 优化 → Core.program
  3. 从 Global_env 提取:
     - all_local_types → types
     - 所有 traits → traits
     - method_env → methods
     - ext_method_env → ext_methods
  4. core_format.export → .core 文件
```

---

## 4. S-Expression 序列化

`.core` 文件使用 `moon_sexp_conv.ml` 和 `w.ml` 进行序列化：

```ocaml
(* w.ml - S-Expression 类型 *)
type t = Atom of string | List of t list

(* moon_sexp_conv.ml - 自动 derive sexp 转换 *)
```

Core IR 中每个类型都有 `sexp_of_*` 函数，通过 `ppx_base` 宏自动生成或手写。

---

## 5. Bundle 操作

`moonc bundle-core` 将多个 `.core` 文件合并为一个：

```ocaml
let bundle ~inputs ~path =
  let pkgs = Array.concat (List.map (fun input -> import ~path:input) inputs) in
  (* 合并所有程序项 *)
  let program = Array.fold_left (fun acc pkg -> 
    List.rev_append pkg.program acc) [] pkgs |> List.rev in
  (* 合并类型/方法等 *)
  ...
  (* 写出 *)
  export ~action:(Write_file path) ...
```

Bundle 后产出单个 `.core` 文件，可以用于后续的 `link-core`。

---

## 6. 与其他格式的对比

| 格式 | 阶段 | 包含 |
|------|------|------|
| `.mbt` | 源文件 | MoonBit 源代码 |
| `.mi` | 类型检查产出 | 公共 API 摘要（类型、函数签名、trait声明） |
| `.core` | build-package 产出 | Core IR + 类型 + 方法环境（完整实现） |
| `.wasm` | 最终产出 | Wasm GC 二进制 |

### `.mi` vs `.core`

| | .mi | .core |
|---|-----|-------|
| 包含 | 仅签名 | 完整 IR |
| 跨包引用 | 类型检查期使用 | 链接/单态化期使用 |
| 大小 | 小 | 大 |
| 是否必须 | 是（下游类型检查） | 是（下游链接/单态化） |

---

## 7. 读取和校验

### 格式校验

`.core` 文件在读取时进行基本的格式校验：

```ocaml
let of_string s : serialized array =
  let sexp = Moon_sexp_conv.parse_string s in
  match sexp with
  | List [Atom "core"; ...] -> parse_core_contents ...
  | _ -> raise (Invalid_core_format ...)
```

### 错误处理

格式损坏时提供有意义的错误信息，包括文件路径和具体问题。

---

## 8. Core 文件的用途

| 场景 | 说明 |
|------|------|
| 增量编译 | 只重新编译修改的包，复用其他包的 .core |
| 包分发 | 分发预编译的 .core（类似 .o/.a） |
| 跨语言链接 | 理论上任何后端都可以消费 .core |
| 缓存 | CI/CD 缓存中间产物 |
| 调试 | 导出 .core 检查优化效果 |

---

## 9. 完整编译流程中的 .core

```
开发:
  moon check      → .mi (类型检查 + 导出接口)
  moon build      → .core + .mi (每个包单独)

链接:
  moonc link-core  → 读取多个 .core → 链接 → 单态化 → .wasm

分发:
  moon publish     → 上传 .core + .mi 到 mooncakes.io
  下游 moon add    → 下载 .core + .mi
```
