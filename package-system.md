# 包与模块系统详解

MoonBit 的文件组织采用类似 Go 的包（package）和模块（module）体系，由 `moon.pkg.json` 管理导入，`moon.mod.json` 管理依赖。相关模块：`pkg.ml`、`pkg_config_util.ml`、`pkg_path_tbl.ml`、`pkg_info.ml`、`mi_format.ml`、`parsing_import_path.ml`。

---

## 项目布局

```
my_module/                          ← 模块根（含 moon.mod.json）
├── moon.mod.json                   ← 模块元数据
├── moon.pkg.json                   ← 根包配置（可选）
├── main.mbt                        ← 根包源文件
├── main_test.mbt                   ← 黑盒测试
├── liba/                           ← 子包
│   ├── moon.pkg.json               ← 子包配置（必需）
│   └── liba.mbt
└── libb/
    ├── moon.pkg.json
    └── libb.mbt
```

- **Module**：由 `moon.mod.json` 定义，是一个可发布的代码集合
- **Package**：由 `moon.pkg.json` 定义，每个目录是一个编译单元
- 所有 `.mbt` 文件在同一包内共享所有声明（类似 Go）

---

## 1. moon.mod.json（模块配置）

```json
{
  "name": "username/hello",
  "version": "0.1.0",
  "source": ".",
  "deps": {
    "moonbitlang/x": "0.4.6"
  },
  "preferred-target": "native"
}
```

编译器通过 `Basic_config.current_package` 维护当前编译的包名。

---

## 2. moon.pkg.json（包配置）

```json
{
  "import": [
    { "path": "username/hello/liba", "alias": "a" },
    "moonbitlang/x/encoding"
  ],
  "wbtest-import": [...],
  "bbtest-import": [...],
  "options": {
    "is-main": true
  },
  "supported-targets": "+native"
}
```

### 导入配置解析 (`pkg_config_util.ml`)

```ocaml
type import_kind = Wbtest | Bbtest | Normal

(* 解析导入配置 *)
let parse_import_item ~import_kind pkg_config =
  (* 从 JSON 中提取 "import" / "wbtest-import" / "bbtest-import" *)
  (* 支持字符串格式和对象格式 *)
```

导入条目支持：
- **字符串**：`"username/hello/liba"` → 别名 = `liba`
- **对象**：`{"path": "...", "alias": "a"}` → 别名 = `a`
- **按需导入**：`{"path": "...", "value": ["foo", "bar"]}` → 只导入指定符号

### 导入种类

| Kind | 用途 |
|------|------|
| `Normal` | 常规导入，对所有 `.mbt` 文件可见 |
| `Wbtest` | 白盒测试导入（`*_wbtest.mbt`） |
| `Bbtest` | 黑盒测试导入（`*_test.mbt`） |

---

## 3. 包表 (`pkg.ml`)

### 数据结构

```ocaml
type pkg_tbl_entry = Pkg_info.mi_view option lazy_t
(* 懒加载的 .mi 文件内容 *)

type pkg_info = {
  mi_info : pkg_tbl_entry;     (* .mi 内容 *)
  usage : bool ref;             (* 是否被使用 *)
  alias_to : string option;     (* 别名重命名 *)
  direct_uses : direct_uses;    (* 按需导入的值列表 *)
  loc : Loc.t;                  (* 导入声明位置 *)
}

type direct_uses = Direct_use_all | Values of (string * Loc.t) list
```

### 包加载

```ocaml
(* 加载标准库 *)
val load_std : pkg_tbl -> std_path:string -> diagnostics:Diagnostics.t -> unit

(* 加载 .mi 文件 *)
val load_mi : pkg_tbl -> import_path -> string -> diagnostics:Diagnostics.t -> unit
```

标准库（`moonbitlang/core`）自动对当前包可见，不需要显式导入。未来会转为需要显式导入。

---

## 4. 包路径解析

### 路径格式

```
username/module_name/path/to/package
```

引用方式：`@alias.symbol`。别名默认为路径的最后一段。

```mbt
// moon.pkg.json: import "username/hello/liba"
// 使用: @liba.some_function()

// moon.pkg.json: import {"path": "username/hello/liba", "alias": "a"}
// 使用: @a.some_function()
```

### 内部解析 (`parsing_import_path.ml`)

```ocaml
type t = { pkg : string; path : string }
(* "username/hello/liba" → {pkg="username/hello"; path="liba"} *)
```

---

## 5. .mi 接口文件 (`mi_format.ml`)

`.mi` 文件是 MoonBit 的编译产物，包含包的公共 API 摘要：

```ocaml
module Serialize = struct
  type serialized = {
    export_values : (string * Value_info.toplevel) array;
    export_types : (string * Typedecl_info.t) array;
    export_traits : (string * Trait_decl.t) array;
    export_type_alias : (string * Type_alias.t) array;
    export_method_env : method_array;       (* 常规方法 *)
    export_ext_method_env : ext_method_array; (* 扩展方法 *)
    export_trait_impls : Trait_impl.impl array;
    name : string;
  }
end
```

`.mi` 文件通过 `Global_env.export_mi` 生成，通过 `Pkg.load_mi` 加载。这使得跨包编译只需要 `.mi` 接口而非完整源码。

### `.mi` 的生命周期

```
编译 check:
  源码 → 类型检查 → .mi

编译 build-package:
  源码 + .mi(依赖) → 类型检查 → Core IR + .mi

下游编译:
  .mi(上游) + 源码 → 类型检查 → ...
```

---

## 6. 编译流程中的包处理

### check 子命令

```ocaml
let check () =
  let std_import = Std_Path !Basic_config.std_path in
  let imports = mi_files |> List.map (fun imp -> Import_Path imp) in
  process_config_and_input             (* 解析 moon.pkg.json *)
    ~pkg_config_file ~imports ~std_import;
  tast_of_ast ~import_items ~pkgs ...  (* 类型检查 *)
  → 产出 .mi
```

### build-package 子命令

```ocaml
let build_package () =
  (* 1. 加载 std 和各依赖的 .mi *)
  (* 2. 解析源码 *)
  (* 3. 类型检查 *)
  (* 4. 生成 Core IR *)
  (* 5. 导出 .core + .mi *)
```

### link-core 子命令

```ocaml
let link_core () =
  (* 1. 读取各包的 .core 文件 *)
  (* 2. core_link.link 链接 *)
  (* 3. monofy 单态化 *)
  (* 4. 生成 .wasm *)
```

---

## 7. 全局环境 (`global_env.ml`)

### 类型存储

```ocaml
module All_types = struct
  type t = {
    toplevel : Typing_info.types;     (* 当前包的类型的 *)
    builtin : Typing_info.types;      (* 内建类型（Option, Ref, Result等） *)
    type_alias : Type_alias.t Hash_string.t;
    pkgs : Pkg.pkg_tbl;               (* 依赖包的 .mi 信息 *)
  }
end
```

类型查找路径：
1. 当前包的顶层类型
2. 内建类型（`builtin.ml` 注册的）
3. 类型别名
4. 依赖包（通过 `pkgs` 包表查找）

---

## 8. 符号可见性

| 修饰符 | 含义 |
|--------|------|
| 无（私有） | 仅包内可见 |
| `pub` | 包外可读，外部不可构造 |
| `pub(all)` | 包外可读写构造 |
| `pub(readonly)` | 包外只读 |

这些通过 `Syntax.visibility` 类型建模：
```ocaml
type visibility =
  | Vis_default        (* 私有 *)
  | Vis_priv           (* 显式私有 *)
  | Vis_pub of { attr : string option; loc_ }  (* pub / pub(all) *)
```

---

## 9. is-main 与入口点

`moon.pkg.json` 中的 `"is-main": true` 将当前包标记为可执行入口：

```ocaml
(* pkg_config_util.ml *)
type is_main_result = Is_main of Loc.t | Not_main

(* 解析后用于 build_context *)
match parse_is_main pkg_config with
| Is_main loc -> Exec { is_main_loc = loc }
| Not_main -> if is_main then Exec ... else Lib
```

`is-main` 包的 `pub fn main`（或 `async fn main`）成为 Wasm 模块的入口函数。

---

## 10. supported-targets

```json
{
  "supported-targets": "+native"
}
```

限制包只能在特定后端编译。`moon.pkg.json` 中的 `supported_targets = "+native"` 表示此包仅支持 native 目标，在 Wasm GC 目标上会报错。这对于依赖平台特定功能的包（如 async IO）很重要。
