# Wasm GC 代码生成详解

MoonBit 编译器的当前公开后端将 Clam IR 编译为 Wasm GC 二进制。涉及的核心模块：`wasm_of_clam_gc.ml` → `dwarfsm_*.ml` → `.wasm`。

---

## 代码生成流水线

```
Clam.prog
    │
    ▼
┌──────────────────────────────────────────────┐
│ wasm_of_clam_gc.ml                            │
│   Clam lambda → Dwarfsm_ast.instr 序列        │
│   - 表达式编译                                  │
│   - 类型定义生成                                │
│   - 闭包布局                                    │
└──────────────────────────────────────────────┘
    │ Dwarfsm_ast.module_
    ▼
┌──────────────────────────────────────────────┐
│ dwarfsm_elim_equivdefn.ml (可选)              │
│   消除等价函数定义（去重）                        │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│ shrink_wasmir.ml (可选)                       │
│   Wasm IR 瘦身优化                              │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│ dwarfsm_local_resolve.ml                      │
│   局部变量索引解析                               │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│ dwarfsm_encode.ml / dwarfsm_encode_wasm.ml    │
│   序列化为二进制 .wasm 或文本 .wat               │
│   - LEB128编码                                  │
│   - 类型段/函数段/代码段/导出段等                   │
└──────────────────────────────────────────────┘
    │ .wasm
```

---

## 1. Clam → Dwarfsm 转换 (`wasm_of_clam_gc.ml`, 993行)

这是代码生成的核心。它将 Clam 的函数式 IR 转换为扁平的 Wasm 指令序列。

### 编译顶层程序

```ocaml
val compile : Clam.prog -> Dwarfsm_ast.module_
```

对 `Clam.prog` 的处理：
1. **生成 Wasm GC 类型定义**（`type_defs` → `rectype` 段）
2. **编译全局初始化**（`init` lambda）
3. **编译每个顶层函数**（`fns` → `func` 段）
4. **编译 main 入口**（`main` lambda）
5. **生成导出段**

### 编译 Lambda 到指令序列

`wasm_of_clam_gc.ml` 将每个 Clam lambda 表达式编译为 Dwarfsm 指令：

| Clam Lambda | Dwarfsm 指令 | 说明 |
|-------------|-------------|------|
| `Lconst c` | `I32_const` / `F64_const` / `Ref_null` 等 | 常量加载 |
| `Lvar {var}` | `Local_get` | 局部变量 |
| `Lassign {var; e}` | 编译e → `Local_set` | 变量赋值 |
| `Llet {name; e; body}` | 编译e → `Local_set` → 编译body | let绑定 |
| `Lif {pred; ifso; ifnot}` | 编译pred → `If [block(ifso)] [block(ifnot)]` | 条件 |
| `Lloop {params; body; args}` | `Block [Loop [body]]` + br_table | 循环 |
| `Lapply {fn=StaticFn addr; args}` | `Call` | 静态调用 |
| `Lapply {fn=Dynamic var; args}` | `Call_ref` | 动态调用 |
| `Lapply {fn=Object ...; args}` | `Struct_get` → `Call_ref` | 方法调用 |
| `Lallocate {kind=Struct; fields}` | `Struct_new` + `Struct_set` | 结构分配 |
| `Lallocate {kind=Enum {tag}; fields}` | `Struct_new` + tag设置 + `Struct_set` | 枚举分配 |
| `Lget_field {obj; index}` | `Struct_get` | 读字段 |
| `Lset_field {obj; field; index}` | `Struct_set` | 写字段 |
| `Lclosure {captures; address}` | `Struct_new` + 捕获 → `Struct_set` | 闭包创建 |
| `Lmake_array {elems}` | `Array_new_fixed` | 数组创建 |
| `Larray_get_item {arr; index}` | `Array_get` | 读数组 |
| `Larray_set_item {arr; index; item}` | `Array_set` | 写数组 |
| `Lswitch {obj; cases; default}` | `Br_table` + I32_tag提取 | 构造器switch |
| `Ljoinlet` / `Ljoinapply` | `Block` + `Br` | continuation |
| `Lreturn e` | 编译e → `Return` | 返回 |
| `Lprim {fn; args}` | 对应Wasm算术/位指令 | 基元操作 |
| `Lstub_call` | `Call` (import) | FFI调用 |
| `Lcatch` | `Try_table` + `Catch` | 异常处理 |
| `Lcast` | `Ref_cast` | 类型转换 |

### 指令构造器

`wasm_of_clam_gc.ml` 依赖两个构造器模块：

**`wasmlinear_constr.ml`** — Wasm 线性指令构造器：
```ocaml
val block : label -> typeuse -> instr list -> instr
val loop : label -> typeuse -> instr list -> instr
val if_ : typeuse -> instr list -> instr list -> instr
val br_ : labelidx -> instr
val func : ... -> func
val call : funcidx -> instr
val i32_const : int32 -> instr
val f64_const : float -> instr
...
```

**`wasmgc_constr.ml`** — Wasm GC 特定构造器：
```ocaml
val struct_new : typeidx -> instr
val struct_get : typeidx -> fieldidx -> instr
val struct_set : typeidx -> fieldidx -> instr
val array_new : typeidx -> instr
val array_get : typeidx -> instr
val array_set : typeidx -> instr
val ref_cast : ... -> instr
val ref_null : typeidx -> instr
...
```

---

## 2. Dwarfsm IR (`dwarfsm_ast.ml`, 3,365行)

Dwarfsm 是 Wasm 的 OCaml AST 表示，位于 `wasm_of_clam_gc` 和最终编码之间。

### 核心数据结构

```ocaml
type module_ = {
  id     : binder;
  fields : modulefield list;
}

type modulefield =
  | MType of rectype          (* 递归类型定义 *)
  | MFunc of func             (* 函数定义 *)
  | MTable of table           (* 间接调用表 *)
  | MMem of mem               (* 线性内存 *)
  | MGlobal of global         (* 全局变量 *)
  | MElem of elem             (* 元素段 *)
  | MData of data             (* 数据段 *)
  | MStart of funcidx         (* 启动函数 *)
  | MImport of import         (* 导入 *)
  | MExport of export         (* 导出 *)

type func = {
  typeuse : typeuse;          (* 函数类型索引 *)
  locals  : (int * numtype) list;      (* 局部变量 *)
  body    : instr list;                (* 函数体指令 *)
}

type label = string option     (* 标签名（用于block/loop/if） *)
```

### 指令集（完整）

```ocaml
type instr =
  (* === 控制流 === *)
  | Block of { label : label; typeuse : typeuse; instrs : instr list }
  | Loop of { label : label; typeuse : typeuse; instrs : instr list }
  | If of { label : label; typeuse : typeuse; instrs1 : instr list; instrs2 : instr list }
  | Br of labelidx
  | Br_if of labelidx
  | Br_table of labelidx list * labelidx
  | Return
  | Unreachable
  | Nop
  | Drop

  (* === 调用 === *)
  | Call of funcidx
  | Call_ref of typeidx
  | Call_indirect of tableidx * typeuse
  | Return_call of funcidx
  | Return_call_ref of typeidx

  (* === 局部变量 === *)
  | Local_get of localidx
  | Local_set of localidx
  | Local_tee of localidx

  (* === 全局变量 === *)
  | Global_get of globalidx
  | Global_set of globalidx

  (* === 整数常量 === *)
  | I32_const of int32
  | I64_const of int64

  (* === 整数运算 (i32) === *)
  | I32_add | I32_sub | I32_mul
  | I32_div_s | I32_div_u | I32_rem_s | I32_rem_u
  | I32_and | I32_or | I32_xor
  | I32_shl | I32_shr_s | I32_shr_u | I32_rotl | I32_rotr
  | I32_eq | I32_ne
  | I32_lt_s | I32_lt_u | I32_gt_s | I32_gt_u
  | I32_le_s | I32_le_u | I32_ge_s | I32_ge_u
  | I32_eqz
  | I32_clz | I32_ctz | I32_popcnt
  | I32_extend8_s | I32_extend16_s
  | I32_wrap_i64

  (* === 整数运算 (i64) === *)
  | I64_add | I64_sub | I64_mul
  | I64_div_s | I64_div_u | I64_rem_s | I64_rem_u
  | I64_and | I64_or | I64_xor
  | I64_shl | I64_shr_s | I64_shr_u | I64_rotl | I64_rotr
  | I64_eq | I64_ne
  | I64_lt_s | I64_lt_u | I64_gt_s | I64_gt_u
  | I64_le_s | I64_le_u | I64_ge_s | I64_ge_u
  | I64_eqz
  | I64_clz | I64_ctz
  | I64_extend_i32_s | I64_extend_i32_u

  (* === 浮点运算 (f32) === *)
  | F32_add | F32_sub | F32_mul | F32_div
  | F32_sqrt | F32_abs | F32_neg
  | F32_ceil | F32_floor | F32_trunc | F32_nearest
  | F32_min | F32_max
  | F32_eq | F32_ne | F32_lt | F32_gt | F32_le | F32_ge
  | F32_const of string * float  (* (repr, value) *)
  | F32_convert_i32_s | F32_convert_i32_u
  | F32_convert_i64_s | F32_convert_i64_u
  | F32_demote_f64
  | F32_reinterpret_i32

  (* === 浮点运算 (f64) === *)
  | F64_add | F64_sub | F64_mul | F64_div
  | F64_sqrt | F64_abs | F64_neg
  | F64_ceil | F64_floor | F64_trunc | F64_nearest
  | F64_min | F64_max
  | F64_eq | F64_ne | F64_lt | F64_gt | F64_le | F64_ge
  | F64_const of string * float
  | F64_convert_i32_s | F64_convert_i32_u
  | F64_convert_i64_s | F64_convert_i64_u
  | F64_promote_f32
  | F64_reinterpret_i64

  (* === i32/f32 互转 === *)
  | I32_reinterpret_f32
  | I32_trunc_f32_s | I32_trunc_f32_u
  | I32_trunc_f64_s | I32_trunc_f64_u
  | I64_reinterpret_f64
  | I64_trunc_f32_s | I64_trunc_f32_u
  | I64_trunc_f64_s | I64_trunc_f64_u

  (* === GC 结构体 === *)
  | Struct_new of typeidx
  | Struct_new_default of typeidx
  | Struct_get of typeidx * fieldidx
  | Struct_get_s of typeidx * fieldidx
  | Struct_get_u of typeidx * fieldidx
  | Struct_set of typeidx * fieldidx

  (* === GC 数组 === *)
  | Array_new of typeidx
  | Array_new_default of typeidx
  | Array_new_fixed of typeidx * int32
  | Array_new_data of typeidx * dataidx
  | Array_new_elem of typeidx * elemidx
  | Array_get of typeidx
  | Array_get_s of typeidx
  | Array_get_u of typeidx
  | Array_set of typeidx
  | Array_len
  | Array_fill of typeidx
  | Array_copy of typeidx * typeidx
  | Array_init_data of typeidx * dataidx
  | Array_init_elem of typeidx * elemidx

  (* === 引用操作 === *)
  | Ref_null of typeidx
  | Ref_is_null
  | Ref_func of funcidx
  | Ref_eq
  | Ref_as_non_null
  | Ref_cast of reftype * reftype
  | Ref_test of reftype * reftype
  | Br_on_cast of labelidx * reftype * reftype
  | Br_on_cast_fail of labelidx * reftype * reftype
  | Any_convert_extern
  | Extern_convert_any

  (* === i31 引用 === *)
  | Ref_i31
  | I31_get_s | I31_get_u

  (* === 内存操作 === *)
  | I32_load of memarg | I64_load of memarg
  | F32_load of memarg | F64_load of memarg
  | I32_store of memarg | I64_store of memarg
  | F32_store of memarg | F64_store of memarg
  | I32_load8_s of memarg | I32_load8_u of memarg
  | I32_load16_s of memarg | I32_load16_u of memarg
  | I64_load8_s of memarg | I64_load8_u of memarg
  | I64_load16_s of memarg | I64_load16_u of memarg
  | I64_load32_s of memarg | I64_load32_u of memarg
  | I32_store8 of memarg | I32_store16 of memarg
  | I64_store8 of memarg | I64_store16 of memarg | I64_store32 of memarg
  | Memory_size | Memory_grow
  | Memory_fill | Memory_copy | Memory_init of dataidx
  | Data_drop of dataidx

  (* === 表操作 === *)
  | Table_get of tableidx | Table_set of tableidx
  | Table_size of tableidx | Table_grow of tableidx
  | Table_fill of tableidx | Table_copy of tableidx * tableidx
  | Table_init of tableidx * elemidx
  | Elem_drop of elemidx

  (* === try/catch (异常处理) === *)
  | Try_table of typeuse * (instr list) * (catch list)
  | Throw of tagidx
  | Throw_ref
```

---

## 3. 类型定义生成

Clam 的类型定义 (`Ltype_gc.type_defs`) 在代码生成阶段被转换为 Wasm GC 的 `rectype`（递归类型定义）。

### 映射规则

| Ltype_gc 类型 | Wasm GC rectype |
|---------------|-----------------|
| `I32 {kind=_}` | 不需要定义（Wasm值类型） |
| `I64` | 不需要定义 |
| `F32` | 不需要定义 |
| `F64` | 不需要定义 |
| `Ref {tid}` | `(type $tid (struct ...))` |
| `Ref_lazy_init {tid}` | `(type $tid (struct ...))` |
| `Ref_nullable {tid}` | `(type $tid (struct ...))` |
| `Ref_extern` | `externref`（Wasm内建） |
| `Ref_string` | `stringref`（Wasm内建） |
| `Ref_bytes` | `stringref`（Wasm内建） |
| `Ref_func` | `funcref`（Wasm内建） |
| `Ref_any` | `anyref`（Wasm内建） |

### Wasm GC 结构体布局

MoonBit 的类型映射为 Wasm GC struct：

```
(* MoonBit struct Point { x : Int; y : Int } *)
(type $Point (struct (field i32) (field i32)))

(* MoonBit enum Option<T> { None; Some(T) } *)
(type $Option (struct (field i32) (field (ref null $T))))
(* 字段0 = tag (0=None, 1=Some), 字段1 = payload *)

(* MoonBit 闭包 fn(x: Int) -> Int *)
(type $closure_N (struct (field (ref $captured_1)) (field (ref func))))
```

### 数据布局优化 (`pass_layout.ml`)

在 Mcore → Clam 转换之前，`pass_layout.ml` 对数据布局进行优化：
- 字段重排序（减少padding）
- 小字段合并（多个 i8/i16 → i32）
- Option<Char> 的特殊紧凑编码（用i32范围的0/非0表示）

---

## 4. 优化 Pass

### Shrink Wasmir (`shrink_wasmir.ml`)

在 Dwarfsm AST 上的优化：
- 消除冗余的 `Local_get`/`Local_set`
- 合并连续的 `Drop`
- 简化 `Block` 嵌套
- 消除 `Nop`

### 等价函数去重 (`dwarfsm_elim_equivdefn.ml`)

检测并合并相同函数体的函数定义，减小 Wasm 二进制体积：
- 函数体哈希
- 等价定义合并
- 引用重定向

### 未使用 let 消除 (`pass_unused_let.ml`)

在 Clam 层消除从未被引用的变量绑定，减少 Clam → Dwarfsm 翻译的指令数。

---

## 5. 二进制编码 (`dwarfsm_encode.ml`)

将 Dwarfsm AST 序列化为 Wasm 二进制格式：

### 编码项

```
Wasm Binary Format:
  ┌─ Magic Number (0x00 0x61 0x73 0x6D) ─┐
  ├─ Version (0x01 0x00 0x00 0x00)       ─┤
  ├─ Type Section (section id 1)          ─┤
  ├─ Import Section (section id 2)        ─┤
  ├─ Function Section (section id 3)      ─┤
  ├─ Table Section (section id 4)         ─┤
  ├─ Memory Section (section id 5)        ─┤
  ├─ Global Section (section id 6)        ─┤
  ├─ Export Section (section id 7)        ─┤
  ├─ Start Section (section id 8)         ─┤
  ├─ Element Section (section id 9)       ─┤
  ├─ Code Section (section id 10)         ─┤
  ├─ Data Section (section id 11)         ─┤
  └─ ...                                  ─┘
```

### LEB128 编码

Dwarfsm 使用 `basic_vlq64.ml` 进行可变长度量编码：

```ocaml
(* LEB128 无符号编码 *)
val encode_u32 : int32 -> string  (* 1-5字节 *)
val encode_u64 : int64 -> string  (* 1-10字节 *)

(* LEB128 有符号编码 *)
val encode_s32 : int32 -> string
val encode_s64 : int64 -> string
```

### 符号解析 (`dwarfsm_local_resolve.ml`)

在编码前解析局部变量索引。Dwarfsm 的 `localidx` 使用可变引用设计：
```ocaml
type localidx = { mutable var : var }
```
其 `var` 字段在 `dwarfsm_local_resolve.ml` 中被赋值为确定的整数索引。

---

## 6. 辅助工具模块

| 模块 | 功能 |
|------|------|
| `wasm_lex.ml` | Wasm 文本格式词法分析（用于测试/调试） |
| `dwarfsm_parse.ml` | Wasm 文本格式解析（用于读取手写.wat） |
| `dwarfsm_encode_wasm.ml` | 将 Dwarfsm AST 输出为可读的 Wasm 文本（调试用） |
| `dwarfsm_instr_utils.ml` | 指令工具函数 |
| `dwarfsm_basic.ml` | Dwarfsm 基础工具 |
| `dwarfsm_itype.ml` | Wasm 类型表示 |
| `wasmir_util.ml` | Wasm IR 工具函数 |
| `dwarfsm_sexp_*.ml` | Dwarfsm 的 S-Expression 解析/格式化 |

---

## 7. 关键设计决策

### 为什么有自己的 Wasm AST (Dwarfsm) 而非直接用库？

1. **可控性**：需要对所有 Wasm GC 扩展指令的精确支持
2. **符号解析**：可变引用的设计支持 two-pass 解析（先收集再赋值）
3. **优化**：在 Dwarfsm 层进行 `shrink_wasmir` 和 `elim_equivdefn` 可能比在 Wasm 二进制上操作更容易
4. **调试**：可以从 Dwarfsm 输出文本格式进行 human-readable 检查

### GC 与内存管理

MoonBit 完全依赖 Wasm GC 进行内存管理：
- **无需线性内存**进行对象分配（除非 FFI 需要）
- 所有 MoonBit 堆对象（struct、enum、闭包、数组）都是 Wasm GC 管理的引用
- `externref` 用于 FFI 与宿主环境交互

### 性能考虑

1. **Option 优化**：`Option<Char>` 被优化为 I32（无堆分配）
2. **枚举标签**：枚举判别式存储在 struct 的 `i32` 字段中，而非单独的 tag
3. **闭包布局**：闭包对象 = [captured_vars..., func_ref]，方法表同样
4. **多返回值**：利用 Wasm GC 的 multi-value 返回支持 MoonBit 的多返回值
