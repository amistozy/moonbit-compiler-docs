# 源码映射与调试信息详解

MoonBit 支持生成 Wasm 的 source map 和 DWARF 格式的调试信息。相关模块：`dwarfsm_encode.ml`、`dwarfsm_encode_context.ml`、`dwarfsm_encode_resolve.ml`、`value_tracing.ml`。

---

## 1. 源码映射 (Source Map)

### 生成流程

```ocaml
(* dwarfsm_encode.ml *)
let module_with_source_map ~file ?source_map_url ?source_loader m =
  let ctx = Encode_context.make_context () in
  Encode_resolve.resolve ctx m;
  let code_pos = Vec.empty () in
  let wasm = Encode_wasm.encode ~add_code_pos
    ~custom_sections:(fun () ->
      [( "sourceMappingURL",
         with_length_preceded ~f:int_uleb128
           (match source_map_url with
            | Some url -> string url
            | None -> string ("./" ^ file ^ ".map"))) ])
    ~emit_names:true
    ctx
  in
  (* 生成 source map *)
  ...
```

### 处理步骤

1. **编码**：将 Dwarfsm AST 编码为 Wasm 二进制，同时记录每个指令的 Wasm 字节偏移 (`rel_pc`) 对应的源位置 (`source_pos`)
2. **生成映射**：将 `(rel_pc, source_pos)` 列表转换为标准 source map JSON
3. **URL 嵌入**：在 Wasm 的 custom section 中嵌入 `sourceMappingURL`，指向 `.map` 文件

### Source Map 数据

```ocaml
type source_pos = {
  pkg : string;    (* 包名 *)
  file : string;   (* 文件名 *)
  line : int;      (* 行号 *)
  col : int;       (* 列号 *)
}

type source_loader = pkg:string -> file:string -> (string * string option)
(* 返回 (path, content option) *)
```

### 编译器选项

```bash
# 启用 source map
moonc compile --source-map input.mbt

# 指定 source map URL
moonc compile --source-map --source-map-url "https://example.com/maps/"
```

相关配置（`driver_config.ml`）：
```ocaml
val source_map : bool ref
val source_map_url : string ref
```

---

## 2. DWARF 调试信息

### 模块

- `dwarfsm_ast.ml` — Dwarfsm AST（已包含 DWARF 相关的 type definitions）
- `dwarfsm_encode.ml` — 二进制编码
- `dwarfsm_encode_context.ml` — 编码上下文（符号表）
- `dwarfsm_encode_resolve.ml` — 符号解析

### Wasm 名称段

```ocaml
let module_ ~emit_names m =
  let ctx = Encode_context.make_context () in
  Encode_resolve.resolve ctx m;
  let wasm = Encode_wasm.encode ~emit_names ctx in
  Byteseq.to_string wasm
```

当 `emit_names=true` 时，生成的 Wasm 包含 `name` custom section，存储函数名和局部变量名。

---

## 3. 值追踪 (`value_tracing.ml`)

值追踪是一种运行时调试机制，允许在编译时注入追踪代码：

```ocaml
(* driver_util.ml 中的回调 *)
let tracing_callback asts =
  if !enable_value_tracing then Value_tracing.instrument asts
  else asts
```

当 `--enable-value-tracing` 标志启用时：
- 在关键表达式处注入 `Pprintln` 或其他追踪调用
- 支持运行时值的检查

---

## 4. 源位置在 IR 中的保留

MoonBit 编译器在 IR 的每个节点保留源位置信息：

### Core IR

```ocaml
type location = Rloc.t    (* 解析阶段的相对位置 *)

(* 大多数 Core.expr 节点包含 loc_ : location *)
| Cexpr_const of { c : constant; ty : typ; loc_ : location }
| Cexpr_let of { name : binder; rhs : expr; body : expr; ty : typ; loc_ : location }
```

### Clam IR

```ocaml
| Levent of { expr : lambda; loc_ : location }
(* Levent 是纯位置标记，不产生任何代码 *)
```

`Levent` 节点在 Wasm 生成时被转换为位置注释，用于 source map。

### Dwarfsm

```ocaml
(* 二进制编码时，每个指令的 PC 映射到源位置 *)
let add_code_pos rel_pc pos = Vec.push code_pos (rel_pc, pos)
```

---

## 5. 编码解析器

### 解析上下文

```ocaml
(* dwarfsm_encode_context.ml *)
type context  (* 符号表：typeidx→Wasm types, funcidx→funcs, ... *)

val make_context : unit -> context
```

### 符号解析前

Dwarfsm 使用可变引用操作：
```ocaml
type typeidx = { mutable var : var }
type funcidx = { mutable var : var }
```

在 `Encode_resolve.resolve` 之前，索引字段是 `Unresolve "name"`（字符串）；resolve 之后变为 `Resolved { index }`（整数索引）。

---

## 6. 调试信息的优化

### 优化 vs 调试

编译选项控制调试信息级别：

```ocaml
if !Config.debug then
  if !source_map then
    (* 生成 source map + wasm *)
    let wasm, source_map = module_with_source_map ... in
    ...
  else
    (* 仅生成带名称的 wasm *)
    module_ ~emit_names:true m
else
  (* 无调试信息 *)
  module_ ~emit_names:false m
```

当 `Config.debug=false` 时：
- `emit_names=false` → 无 name section → 更小的二进制
- 无 source map

---

## 7. Code Section 偏移计算

```ocaml
(* dwarfsm_encode.ml - 找到 code section 的字节偏移 *)
let codesec_offset =
  let read_int_leb128 data ofs = ... in
  let rec find_codesec_offset ofs =
    let id, ofs = read_int_leb128 wasm ofs in
    let size, ofs = read_int_leb128 wasm ofs in
    if id = 10 (* code section id *) then ofs
    else find_codesec_offset (ofs + size)
  in
  find_codesec_offset 8  (* 跳过 magic+version *)
```

这会遍历 Wasm sections 找到 code section（id=10）的起始位置，用于 source map 的偏移计算。

---

## 8. 调试体验

完整的调试工具链：

```
源码 (.mbt)
  → moonc compile --source-map --debug
  → main.wasm + main.wasm.map
  → 浏览器 DevTools / wasm 调试器
  → 显示原始 MoonBit 源文件和断点
```
