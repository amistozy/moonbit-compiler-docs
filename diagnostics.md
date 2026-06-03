# 诊断与错误系统详解

MoonBit 编译器有一套完整的诊断系统，支持错误报告、警告管理、错误码系统和位置追踪。相关模块：`diagnostics.ml`、`errors.ml`、`error_code.ml`、`warnings.ml`、`local_diagnostics.ml`、`alerts.ml`。

---

## 1. 诊断报告 (`diagnostics.ml`)

### 数据结构

```ocaml
type report = {
  loc : Loc.t;                    (* 源代码位置 *)
  message : string;                (* 错误消息 *)
  error_code : Error_code.t;      (* 错误码 *)
}
```

### 诊断收集器

```ocaml
type t  (* 诊断报告集合 *)

val make : unit -> t               (* 创建空收集器 *)
val add_error : t -> report -> unit
val add_warning : t -> report -> unit
val has_fatal_errors : t -> bool
val reset : t -> unit
val check_diagnostics : t -> unit  (* 有错误时抛出 Fatal_error *)
```

### 工作流

```
1. 创建: Diagnostics.make()
2. 收集: 各阶段 add_error/add_warning
3. 检查: 
   - has_fatal_errors → 异常退出
   - check_diagnostics → 打印所有错误并退出
4. 重置: reset() 用于多文件编译
```

---

## 2. 错误码 (`error_code.ml`)

```ocaml
type t =
  | Internal            (* 编译器内部错误 *)
  | Lexing_error        (* 词法错误 *)
  | Parse_error         (* 语法错误 *)
  | Json_parse_error    (* JSON 解析错误 *)
  | Type_error          (* 类型错误 *)
  | Unbound_error       (* 未绑定符号 *)
  | ...
```

错误码用于分类和过滤错误，支持 `--explain` 查阅详细说明。

---

## 3. 错误定义 (`errors.ml`)

`errors.ml` 提供了 100+ 个错误构造器函数，覆盖编译器的各个阶段：

```ocaml
(* 词法/语法 *)
val lexing_error : loc_start -> loc_end -> string -> report
val parse_error : loc_start -> loc_end -> string -> report

(* 类型错误 *)
val type_mismatch : loc -> expected:string -> actual:string -> report
val unbond_variable : loc -> string -> report

(* 模式匹配 *)
val partial_match : loc -> hint_cases -> report
val unreachable_pattern : loc -> report

(* Trait *)
val trait_not_implemented : loc -> trait:string -> typ:string -> report

(* 其他 *)
val internal : string -> report    (* ICE *)
val invalid_init_or_main : kind -> loc -> report
val attribute_parse_error : loc -> string -> report
val json_parse_error : loc_start -> loc_end -> string -> report
```

---

## 4. 警告系统 (`warnings.ml`)

### 警告种类

MoonBit 定义了 45 个警告（编号 1-45），可单独启用/禁用：

```ocaml
type kind =
  | Unused_func of string              (* 1 *)
  | Unused_var of { ... }             (* 2 *)
  | Unused_type_declaration of string (* 3 *)
  | Partial_match of { ... }          (* 11 *)
  | Unreachable                       (* 12 *)
  | Deprecated_syntax of { ... }      (* 27 *)
  | Todo                               (* 28 *)
  | Unused_package of { ... }        (* 29 *)
  | Invalid_attribute of string       (* 42 *)
  | Invalid_inline_wasm of string     (* 44 *)
  | Implement_trait_with_method ...   (* 45 *)
  | ...
```

### 警告控制

```ocaml
(* 默认配置 *)
let default_warnings = "+a-31-32"        (* 启用所有，除31,32 *)
let default_warnings_as_errors = "-a+11+15+23+24+44"  (* 某些警告升级为错误 *)

(* 解析选项 *)
val parse_options : errflag:bool -> string -> unit
(* "+a" 启用所有, "-3" 禁用3号, "@11" 升级11号为错误 *)
```

### 警告编号表（部分）

| # | 名称 | 描述 |
|---|------|------|
| 1 | Unused_func | 未使用的函数 |
| 2 | Unused_var | 未使用的变量 |
| 11 | Partial_match | 不完全的模式匹配 |
| 12 | Unreachable | 不可达代码 |
| 27 | Deprecated_syntax | 废弃语法 |
| 28 | Todo | 未完成代码 |
| 31 | Optional_arg_never_supplied | 可选参数从未提供 |
| 32 | Optional_arg_always_supplied | 可选参数的默认值从未使用 |

---

## 5. 局部诊断 (`local_diagnostics.ml`)

局部诊断用于在类型检查阶段收集非致命错误（如类型不匹配）：

```ocaml
type error  (* 局部错误报告 *)

val add_warning : Diagnostics.t -> error -> unit
val add_error : Diagnostics.t -> error -> unit
```

与 `Diagnostics.report` 不同，局部错误可能被后续的类型推导修正（通过 error recovery）。

---

## 6. 告警系统 (`alerts.ml`)

告警用于 `moon.pkg.json` 中的 `"alert"` 配置：

```json
{
  "alert": [
    { "kind": "deprecated", "packages": ["old_package"] }
  ]
}
```

在 `pkg_config_util.ml` 中解析：

```ocaml
val parse_warn_alert : Json_types.t option -> unit
```

---

## 7. ICE 捕获 (`ice_catcher.ml`)

编译器入口包装在 ICE 保护器中：

```ocaml
val run_with_protection : (unit -> unit) -> unit
```

当编译器内部出错（ICE = Internal Compiler Error）时，捕获异常并生成带堆栈跟踪的用户友好报告。

---

## 8. 位置系统

两层位置模型：

### 相对位置 (`rloc.ml`)

```ocaml
type t  (* 文件内偏移区间 *)
(* 用于解析阶段，轻量级 *)
```

### 绝对位置 (`loc.ml`)

```ocaml
type t = {
  loc_start : Lexing.position;   (* (file, line, col) *)
  loc_end : Lexing.position;
}
(* 包含文件名，用于最终错误报告 *)
```

转换：`Loc.of_menhir (loc_start, loc_end)` 将 Menhir 词法位置转为 Loc。

---

## 9. 编译流程中的诊断

```
解析阶段:
  errors.ml: lexing_error, parse_error
  → Diagnostics.add_error

类型检查:
  local_diagnostics.ml: type_mismatch, unbound_variable
  → Diagnostics.add_error / add_warning

模式匹配:
  check_match.ml: partial_match, unreachable
  → Diagnostics.add_warning

FFI验证:
  stub_type.ml: invalid_inline_wasm
  → Diagnostics.add_warning

最终:
  Diagnostics.check_diagnostics → 打印所有错误 → Fatal_error (exit 2)
```

---

## 10. 错误消息示例

```
Error (Type Error): type mismatch
  expected: Int
  actual: String
  at file main.mbt, line 5, column 12

Warning [011]: Partial match, some hints:
  Some(_)
  at file main.mbt, line 3, column 1

Warning [028]: unfinished code
  at file main.mbt, line 10, column 4
```
