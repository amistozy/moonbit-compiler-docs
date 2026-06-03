# FFI 外部函数接口详解

MoonBit 通过 `extern` 关键字支持与外部代码的互操作。支持三种 FFI 模式：导入（Import）、嵌入式代码（Embedded）、内部函数（Internal）。相关模块：`stub_type.ml`（桩类型定义）、`primitive.ml`（基元操作）、`moon_intrinsic.ml`（编译器内建）。

---

## FFI 语法

```mbt
// 1. 导入外部函数
extern "c" fn puts(s : String) -> Int = "puts"

// 2. 嵌入式 Wasm/GVM 代码
extern "wasm" fn add(x : Int, y : Int) -> Int =
  #|(func (param i32 i32) (result i32)
  #|  local.get 0
  #|  local.get 1
  #|  i32.add)
  #|

// 3. 纯 MoonBit 内部函数
extern fn debug_write(s : String) = "$moonbit.debug_write"

// 4. 多行嵌入式代码
extern "c" fn complex_op(x : Int) -> Int =
  #|...multi-line embedded code...

// 5. 外部类型
extern type ExternalHandle
```

---

## 1. 桩类型 (`stub_type.ml`)

### 定义

```ocaml
type t =
  | Import of { module_name : string; func_name : string }
  | Internal of { func_name : string }
  | Inline_code_sexp of {
      language : string;        (* "wasm" / "c" 等 *)
      func_body : W.t;         (* S-Expression 格式 *)
    }
  | Inline_code_text of {
      language : string;
      func_body : string;      (* 文本格式 *)
    }
```

### 四种 FFI 模式对比

| 模式 | 语法 | 编译方式 |
|------|------|----------|
| Import | `extern "c" fn ... = "name"` | Wasm import 段 |
| Internal | `extern fn ... = "$func"` | 内部函数调用 |
| Inline (Wasm) | `extern "wasm" fn ... = #\|...` | 内联 Wasm 指令 |
| Inline (其他) | `extern "c" fn ... = "..."` | 依赖外部链接器 |

---

## 2. Import 模式

```mbt
extern "c" fn malloc(size : Int) -> Int = "malloc"
```

编译为 Wasm 的 import 段：
```wasm
(import "c" "malloc" (func $malloc (param i32) (result i32)))
```

在 `stub_type.ml` 中表示为：
```ocaml
Import { module_name = "c"; func_name = "malloc" }
```

注意：`Import` 模式有两个字符串参数 —— 第一个是语言/模块名，第二个（`= "name"` 后面）是函数名。

---

## 3. Internal 模式

```mbt
extern fn println(s : String) = "$moonbit.println"
```

内部函数是编译器运行时提供的函数，名称以 `$` 开头。它们不是 Wasm import，而是由编译器特殊处理。

```ocaml
Internal { func_name = "$moonbit.println" }
```

常见内部函数：
- `$moonbit.malloc` — 内存分配
- `$moonbit.gc.malloc` — GC 分配
- `$moonbit.decref` — 引用计数减少
- `$moonbit.debug_write` — 调试输出

---

## 4. 内联 Wasm 代码

```mbt
extern "wasm" fn i32_add(x : Int, y : Int) -> Int =
  #|(func (param i32 i32) (result i32)
  #|  local.get 0
  #|  local.get 1
  #|  i32.add)
```

这种模式允许在 MoonBit 代码中直接嵌入 Wasm 指令。

### 验证 (`stub_type.ml` 的 `from_syntax`)

嵌入式 Wasm 在编译时会被解析和验证：

```ocaml
| "wasm" ->
    match Dwarfsm_sexp_parse.parse s with
    | (List (Atom "func" :: _) as func_body) :: [] ->
        (* 解析为 Dwarfsm AST *)
        match Dwarfsm_parse.modulefield func_body with
        | Dwarfsm_ast.Func func :: [] ->
            (* 白名单检查 *)
            check_inlinewasm func;  (* 防止调用任意 Wasm 函数 *)
            Ok (Inline_code_sexp { language = "wasm"; func_body })
```

### 白名单机制

内联 Wasm 代码只能调用白名单中的函数：

```ocaml
let wasm_gc_whitelist = []
(* 在 Wasm GC 后端，白名单为空 —— 不能 Call 任意函数 *)

let check_inlinewasm func =
  iter func.code (fun instr ->
    match instr with
    | Call { var = Unresolve name } ->
        if not (mem_string whitelist name) then
          raise (Invalid_wasm_function_call name)
    | _ -> ())
```

只能使用纯指令（无外部调用），确保安全性。

---

## 5. 内联 C/其他语言代码

```mbt
extern "c" fn custom_init() = "custom_setup()"
```

对于非 Wasm 的语言，编译器将代码字符串传递给外部工具链：

```ocaml
| "C" | "c" ->
    (* 检查是否为有效C标识符 *)
    let all_chars_valid =
      String.for_all (fun c -> ('0' <= c && c <= '9') || ...) s in
    ...
    Ok (Inline_code_text { language; func_body = s })
| language ->
    Ok (Inline_code_text { language; func_body = s })
```

这种模式在当前 Wasm GC 后端中通常不会生成有效代码 —— 它依赖于 Native/LLVM 后端。

---

## 6. 编译器内建 (`moon_intrinsic.ml`)

编译器内部识别一组特殊函数，将其映射为高效的基元操作：

```ocaml
type t =
  | Char_to_string        (* %char.to_string *)
  | F64_to_string         (* %f64.to_string *)
  | String_substring      (* %string.substring *)
  | FixedArray_join       (* %fixedarray.join *)
  | FixedArray_iter | FixedArray_iteri | FixedArray_map
  | FixedArray_fold_left | FixedArray_copy | FixedArray_fill
  | Iter_map | Iter_iter | Iter_from_array | Iter_take
  | Iter_reduce | Iter_flat_map | Iter_repeat | Iter_filter | Iter_concat
  | Array_length | Array_get | Array_unsafe_get
  | Array_set | Array_unsafe_set
  | ArrayView_length | ArrayView_unsafe_get | ArrayView_unsafe_set
  | ArrayView_unsafe_as_view
  | BytesView_length | BytesView_unsafe_get | BytesView_unsafe_as_view
```

这些函数在 Core IR 中标记为 `prim = Some (Pintrinsic ...)`，允许编译后期进行特化优化。

---

## 7. 编译流水线中的 FFI 处理

### 解析阶段

`parsing_main.ml` 中，`extern` 声明被解析为：
```ocaml
Ptop_funcdef {
  fun_decl = ...;
  decl_body = Decl_stubs (Embedded { language; code })
}
```

### 语义分析

`stub_type.from_syntax` 将语法桩转换为 `Stub_type.t`：
```ocaml
Import { module_name; func_name }    →  Wasm import
Internal { func_name }               →  内部调用
Inline_code_sexp { language; body }  →  内联Wasm指令
Inline_code_text { language; body }  →  嵌入文本
```

### Core IR

在 Core IR 中，FFI 函数为：
```ocaml
Ctop_stub {
  binder = func_name;
  func_stubs : Stub_type.t;    (* FFI 桩类型 *)
  params_ty : typ list;
  return_ty : typ option;
  is_pub_ : bool;
}
```

### Clam IR

在 Clam IR 中，FFI 调用为：
```ocaml
Lstub_call {
  fn : func_stubs;
  args : lambda list;
  params_ty : ltype list;
  return_ty : ltype option;
}
```

### Wasm 生成

- `Import` → Wasm `(import ...)` 段
- `Internal` → Wasm `(call $internal_func)`
- `Inline_code_sexp` → Wasm 指令直接嵌入

---

## 8. 外部类型

```mbt
extern type ExternalHandle
```

语法上的 `extern type` 声明一个抽象的外部类型：

```ocaml
(* parsing_syntax.ml *)
| Ptd_extern    (* extern type 的类型体 *)
```

在 Ltype_gc 层：
```ocaml
| Ref_extern    (* externref *)
```

在 Wasm GC 中，`extern type` 直接映射为 `externref`，可以安全地与宿主环境交换引用。

---

## 9. 基元操作 (`primitive.ml`)

编译器内置了一套基元操作，直接映射到 Wasm 指令或运行时调用：

```ocaml
type prim =
  | Pignore                  (* ignore(x) *)
  | Pidentity                (* 类型强转 *)
  | Pnot                     (* not *)
  | Ppanic                   (* panic *)
  | Punreachable             (* unreachable *)
  | Pnull                    (* null *)
  | Pis_null                 (* is_null *)
  | Pas_non_null             (* as_non_null *)
  | Pprintln                 (* println *)
  | Pgetstringitem           (* 字符串索引 *)
  | Pstringlength            (* 字符串长度 *)
  | Pcompare                 (* 比较 *)
  | Parith                   (* 算术 *)
  | Pbitwise                 (* 位操作 *)
  | Pcomparison              (* 比较操作 *)
  | Pconvert                 (* 类型转换 *)
  | Pcast                    (* 构造器/enum转换 *)
  | Pgetstringitem           (* 字符串取字符 *)
  | Pclosure_to_extern_ref   (* 闭包→externref *)
  | Praw_func_to_func_ref    (* 原始函数→funcref *)
  | Pcall_object_method      (* trait方法调用 *)
  | Pget_current_continuation (* 获取当前continuation *)
  | Prun_async               (* 运行async任务 *)
  | Penum_field              (* 枚举字段读写 *)
  | ...
```

基元操作在编译后期直接展开为一条或多条 Wasm 指令，无需函数调用开销。
