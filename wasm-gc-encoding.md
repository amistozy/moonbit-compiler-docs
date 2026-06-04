# Wasm GC 指令与二进制编码

MoonBit编译器后端将 Clam IR 编译为 Wasm GC 二进制格式。编码工作由两个构造器模块和一套 Dwarfsm 编码/解码模块完成。

---

## 1. 架构概览

```
Clam.prog (Clam IR)
    │
    ▼
wasm_of_clam_gc.ml         ← Clam→Dwarfsm AST 翻译（主后端）
    │
    ▼
Dwarfsm AST (dwarfsm_ast.ml)  ← Wasm GC 指令的OCaml表示
    │
    ├── wasmgc_constr.ml    ← 高层构造器（带类型信息，用于Clam后端）
    ├── wasmlinear_constr.ml ← 底层构造器（字符串索引，用于链接器）
    │
    ▼
dwarfsm_encode.ml / dwarfsm_encode_wasm.ml  ← 二进制编码器
    │
    ▼
.wasm 文件
```

---

## 2. Dwarfsm AST：Wasm GC 的内存表示

`dwarfsm_ast.ml` 定义了与 Wasm GC 规范一一对应的 OCaml 类型：

### 2.1 模块结构

```ocaml
(* 顶层模块 *)
type modul = {
  version : int;                    (* 1 *)
  types : rectype list;             (* 递归类型组 *)
  memories : mem list;
  datas : data list; 
  tables : table list;
  elems : elem list;
  globals : global list;
  exports : export list;
  start : start option;
  funcs : func list;
  imports : import list;
  tags : tag list;
}

(* 指令（部分） *)
type instr =
  | Unreachable | Nop | Drop
  | Block of blocktype * instr list           (* block ... end *)
  | Loop of blocktype * instr list            (* loop ... end *)
  | If_else of blocktype * instr list * instr list (* if ... else ... end *)
  | Br of labelidx                             (* br $label *)
  | Br_if of labelidx                          (* br_if $label *)
  | Br_on_cast of labelidx * reftype * reftype (* br_on_cast *)
  | Br_on_non_null of labelidx                 (* br_on_non_null *)
  | Return
  | Call of funcidx                            (* call $func *)
  | Call_ref of typeidx                        (* call_ref $type *)
  | Return_call of funcidx
  | Local_get of localidx | Local_set of localidx | Local_tee of localidx
  | Global_get of globalidx | Global_set of globalidx
  | Struct_new of typeidx                      (* struct.new $type *)
  | Struct_new_default of typeidx
  | Struct_get of typeidx * fieldidx           (* struct.get $type $field *)
  | Struct_set of typeidx * fieldidx
  | Array_new of typeidx                       (* array.new $type *)
  | Array_new_default of typeidx
  | Array_new_fixed of typeidx * int32         (* array.new_fixed $type N *)
  | Array_new_data of typeidx * dataidx
  | Array_get of typeidx                       (* array.get $type *)
  | Array_get_s of typeidx | Array_get_u of typeidx
  | Array_set of typeidx
  | Array_len | Array_copy of typeidx * typeidx
  | Array_fill of typeidx
  | Ref_null of heaptype                       (* ref.null ht *)
  | Ref_func of funcidx                        (* ref.func $func *)
  | Ref_as_non_null                            (* ref.as_non_null *)
  | Ref_cast of reftype                        (* ref.cast *)
  | Ref_test of reftype                        (* ref.test *)
  | I32_const of int32 | I64_const of int64
  | F32_const of float | F64_const of float
  | I31_get_s | I31_get_u | I31_new
  | Try_table of blocktype * instr list * catch list  (* try_table *)
  | Throw of tagidx                            (* throw $tag *)
  | Throw_ref                                  (* throw_ref *)
  | Memory_grow | Memory_size
  | Memory_copy | Memory_fill | Memory_init of dataidx
  | I32_load of memarg | I32_store of memarg
  (* ... 更多数值指令 *)
```

### 2.2 关键类型

```ocaml
type valtype = Numtype of numtype | Reftype of reftype

type reftype = Ref of nullability * heaptype

type heaptype =
  | Absheaptype of absheaptype   (* any, eq, i31, struct, array, func, extern等 *)
  | Type of typeidx              (* 具体类型索引 *)

type nullability = Nullable | NonNull

type blocktype = Valtype of valtype list option  (* 可选结果类型 *)
                | Typeidx of typeidx              (* 通过类型索引指定 *)
```

---

## 3. `wasmgc_constr.ml`：高层指令构造器

`wasmgc_constr.ml`（310行）是Clam后端的指令构造层，接受`Ltype.t`和`Tid.t`类型参数：

### 3.1 Ltype → Wasm Valtype 映射

```ocaml
let ltype_to_valtype (typ : Ltype.t) =
  match typ with
  | Ref_extern -> externref
  | F32 -> Numtype F32  | F64 -> Numtype F64
  | I64 -> Numtype I64  | I32 _ -> Numtype I32
  | Ref_bytes -> ref_ ~id_:"$moonbit.bytes" ()
  | Ref_string -> ref_ ~id_:"$moonbit.string" ()  (* JS模式下=externref *)
  | Ref { tid } -> ref_ ~id:tid ()                (* 非空引用 *)
  | Ref_nullable { tid } -> ref_ ~null ~id:tid () (* 可空引用 *)
  | Ref_lazy_init { tid } -> ref_ ~null ~id:tid ()
  | Ref_func -> funcref
  | Ref_any -> anyref
```

### 3.2 指令构造函数

```ocaml
(* 类型构造 *)
let ref_ ?null ?id () = ...           (* (ref null? $type) *)
let sub ?final ?super subtype = ...   (* sub final? $super* ... *)
let typedef ?id subtype = ...         (* (type $id (sub ...)) *)
let comp_struct fields = ...          (* (struct field*) *)
let comp_func params results = ...    (* (func (param...)(result...)) *)
let comp_array fieldtype = ...        (* (array fieldtype) *)

(* 内存指令 *)
let struct_new tid = ...              (* struct.new $tid *)
let struct_get tid index = ...        (* struct.get $tid $field *)
let struct_set tid index = ...        (* struct.set $tid $field *)
let array_new tid = ...               (* array.new $tid *)
let array_new_fixed ~id ~size = ...   (* array.new_fixed $id size *)
let array_new_data ?id data = ...     (* array.new_data $id $data *)
let array_get tid = ...               (* array.get $tid *)
let array_set tid = ...               (* array.set $tid *)
let array_copy ~dst_tid ~src_tid = ...(* array.copy $dst $src *)
let array_fill tid = ...              (* array.fill $tid *)
let call_ref tid = ...                (* call_ref $tid *)

(* 引用指令 *)
let ref_func addr = ...               (* ref.func $addr *)
let ref_null str = ...                (* ref.null $ht *)
let ref_cast valtype = ...            (* ref.cast *)
let ref_cast_ tid = ...               (* ref.cast (ref $tid) *)

(* 局部/全局变量 *)
let local_get id = ...                (* local.get $id *)
let local_set id = ...                (* local.set $id *)
let global_get id = ...               (* global.get $id *)
let global_set id = ...               (* global.set $id *)

(* 控制流 *)
let br_if id = ...                    (* br_if $id *)
```

### 3.3 调试支持

在`--debug`模式下，指令携带`source_type`和`source_name`：

```ocaml
let source_type_ (typ : Ltype.t) =
  if Config.debug then
    match typ with
    | I32 { kind = I32_Int } -> Some Int
    | I32 { kind = I32_Char } -> Some Char
    | I32 { kind = I32_Bool } -> Some Bool
    ...
    else None

let source_name_ source_name =
  if Config.debug then Some source_name else None
```

---

## 4. `wasmlinear_constr.ml`：底层指令构造器

`wasmlinear_constr.ml` 是更低层的构造器，用于链接器和需要直接操控字符串索引的场景，接受字符串而非强类型Id/Tid：

### 4.1 关键差异

| 特性 | wasmgc_constr | wasmlinear_constr |
|---|---|---|
| 标识符类型 | `Tid.t`, `Ident.t`, `Fn_address.t` | `string` |
| 类型引用 | `Ltype.t` → Wasm类型 | 直接`Ast.valtype` |
| 使用场景 | Clam→Dwarfsm翻译 | 链接器、数据段构建 |

### 4.2 指令序列优化

`wasmlinear_constr` 提供两个组合器：

```ocaml
(* 单指令添加到序列：自动优化 *)
let (@:) (instr : instr) (rest : instr list) =
  match (instr, rest) with
  | (F32_const _ | I64_const _ | I32_const _ | Local_get _), Drop :: rest ->
      rest                          (* 消除常量→立即Drop *)
  | (Local_set id1, Local_get id2 :: rest) when id1 = id2 ->
      Local_tee id1 :: rest        (* set+get→tee *)
  | Br _, _ -> [ instr ]            (* Br后忽略死代码 *)
  | _ -> instr :: rest

(* 批量序列连接：递归应用@: *)
let rec (@>) (instrs1 : instr list) (instrs2 : instr list) =
  match instrs1 with
  | [] -> instrs2
  | instr :: [] -> instr @: instrs2
  | instr :: rest -> instr @: rest @> instrs2
```

### 4.3 模块构造

```ocaml
let func ?name ?source_name ?export ?type_ ?params ?results ?locals body =
  (* 构造function segment *)
  [ Ast.Func { id; source_name; type_; locals; code = body; aux = {...} } ]
  @ export

let memory ~name ~export ~import ~shared limits = ...
let data ~offset data_str = ...
let table ~id ~type_ ~fn_names () = ...
let import ~module_ ~name desc = ...
let export ~name ?func () = ...
let global id type_ init = ...
let start ~init = ...
```

---

## 5. 二进制编码管道

### 5.1 解码/编码流程

```
Dwarfsm AST (OCaml结构)
    │
    ▼
dwarfsm_encode.ml          ← 高层编码抽象（LEB128、索引解析）
    │
    ▼
dwarfsm_encode_wasm.ml     ← Wasm特定二进制编码
    │
    ▼
二进制字节序列
    │
    ▼
.wasm 文件
```

### 5.2 文本格式（可选）

```
Dwarfsm AST (OCaml结构)
    │
    ▼
dwarfsm_sexp_encode.ml     ← S-表达式文本输出
    │
    ▼
.wat / .debug格式 文本
```

支持模块的双向转换：

```
.wasm 文件 → dwarfsm_parse.ml → Dwarfsm AST → dwarfsm_sexp_encode.ml → .wat
.wat 文件 → dwarfsm_sexp_parse.ml → Dwarfsm AST → dwarfsm_encode_wasm.ml → .wasm
```

---

## 6. Clam → Dwarfsm 翻译：`wasm_of_clam_gc.ml`

这是编译器后端的主翻译模块（注：该文件较大，此处聚焦其如何调用wasmgc_constr）：

### 6.1 类型段生成

```
Clam.prog.type_defs
    │ 遍历Ltype.type_defs
    ▼
Wasm GC 类型声明：
  - Ref_closure_abstract → (type $T (sub (struct (field (ref null $fn_sig)))))
  - Ref_closure → (type $T (sub (struct (field ref) captures...)))
  - Ref_struct → (type $T (sub (struct (field ... mut?)...)))
  - Ref_concrete_object → (type $T (sub $abstract (struct (field $self_ty))))
```

### 6.2 Clam Lambda → Wasm 指令映射

| Clam IR 指令 | Wasm GC 指令 |
|---|---|
| `Lconst (C_int {v})` | `i32.const v` |
| `Lconst (C_int64 {v})` | `i64.const v` |
| `Lconst (C_double f)` | `f64.const f` |
| `Lconst (C_string s)` | 数据段+`array.new_data` |
| `Lvar { var }` | `local.get $var` / `global.get $var` |
| `Lassign { var; e }` | `local.set $var` / `global.set $var` |
| `Llet { name; e; body }` | `local.set $name` + body |
| `Lsequence { exprs; last_expr }` | 顺序组合（丢弃中间值） |
| `Lif { pred; ifso; ifnot; type_ }` | `if (result $type) ... else ... end` |
| `Lswitch { obj; cases; default; type_ }` | `block ... br_on_cast + br_on_non_null链` |
| `Lloop { params; body; args; label }` | `loop (param ...) ... end` |
| `Lbreak { arg; label }` | `br $label` |
| `Lcontinue { args; label }` | `br $label` |
| `Lapply { fn=StaticFn addr }` | `call $addr` |
| `Lapply { fn=Dynamic var }` | `call_ref $sig_type` |
| `Lapply { fn=Object {obj; method_index} }` | `struct.get + call_ref` |
| `Lreturn expr` | `return` |
| `Lallocate { kind=Struct }` | `struct.new $type` + `struct.set`链 |
| `Lallocate { kind=Enum {tag} }` | `struct.new $type` + `struct.set 0`（tag字段） |
| `Lallocate { kind=Tuple }` | `struct.new $type` |
| `Lallocate { kind=Object {methods} }` | `struct.new $type` + 方法表填充 |
| `Lget_field { kind=Struct; index }` | `struct.get $type $field` |
| `Lset_field { kind=Struct; index }` | `struct.set $type $field` |
| `Lmake_array { elems }` | `array.new_fixed $type N` (编译期已知) 或 `array.new` + 循环 |
| `Larray_get_item` | `array.get $type` / `array.get_s` / `array.get_u` |
| `Larray_set_item` | `array.set $type` |
| `Lclosure { captures; address }` | 闭包对象构造(见下文) |
| `Lclosure_field { obj; index }` | `struct.get $closure_type $field` |
| `Lprim { fn }` | 直接映射为对应Wasm指令 |
| `Ljoinlet` | `block` + `br`链 |
| `Ljoinapply` | `br $join_label` |
| `Lcatch { body; on_exception }` | `try_table ... catch ... end` |
| `Lcast { expr; target_type }` | `ref.cast (ref $target)` |

### 6.3 闭包编码

```
(* 无捕获闭包 *)
let closure_instance = struct.new $closure_tid
closure_instance.struct_set 0 = ref.func $code_addr

(* 有捕获闭包 *)
let closure_instance = struct.new $closure_tid
closure_instance.struct_set 0 = ref.func $code_addr
closure_instance.struct_set 1 = captured_val_1
closure_instance.struct_set 2 = captured_val_2
...

(* 调用闭包 *)
let env = struct.get closure_instance 0  (* code pointer *)
call_ref $fn_sig_type  (* env是隐含参数 *)
```

---

## 7. LEB128 与索引解析

`dwarfsm_encode.ml` 负责将 `Unresolve "name"` 索引解析为数字索引：

```ocaml
(* 编码前阶段：resolve_indexes *)
(* 将所有 Unresolve "name" 替换为 Resolved { index; var_name } *)
(* 使用已收集的类型/函数/局部/全局/标签/数据索引表 *)
```

LEB128编码用于所有整数和索引值，支持变长压缩编码。

---

## 8. 两套构造器的分工

| 场景 | 使用的构造器 |
|---|---|
| Clam→Dwarfsm主翻译 | `wasmgc_constr`（高层，类型安全） |
| 链接器（core_link） | `wasmlinear_constr`（底层，字符串参数） |
| 运行时桩（stub） | `wasmlinear_constr` |
| 数据段构建 | `wasmlinear_constr` |
| 导入/导出声明 | `wasmlinear_constr` |
| 测试/调试工具 | `wasmlinear_constr` |

---

## 与 `codegen.md` 的关系

`codegen.md` 聚焦 Clam→Dwarfsm 翻译的高层算法和流程。本文档深入 Wasm GC 指令的**OCaml表示**（Dwarfsm AST）和**构造器API**（wasmgc_constr/wasmlinear_constr），覆盖指令映射表和二进制编码管道。
