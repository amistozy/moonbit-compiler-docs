# 数据布局与内存模型详解

MoonBit 编译器在 Mcore IR 层通过 `pass_layout.ml` 优化数据布局，将 MoonBit 的代数数据类型映射到 Wasm GC 的结构体和数组。本章详细描述数据如何被布局和访问。

---

## 1. 数据布局流水线

```
Core IR (高级类型)
    │
    ▼
Monofy (单态化) → Mcore (Mtype.t)
    │
    ▼
Pass_layout.ml (布局优化)
    │ - Option 紧凑编码
    │ - Null 表示优化
    │ - 类型转换插入
    ▼
Clam IR (Ltype_gc.t)
    │
    ▼
Wasm GC (struct / array / i31ref)
```

---

## 2. 基本类型映射

| MoonBit 类型 | Mtype | Ltype_gc | Wasm GC |
|-------------|-------|----------|---------|
| `Int` | `T_int` | `I32 {I32_Int}` | `i32` |
| `Int64` | `T_int64` | `I64` | `i64` |
| `Double` | `T_double` | `F64` | `f64` |
| `Float` | `T_float` | `F32` | `f32` |
| `Bool` | `T_bool` | `I32 {I32_Bool}` | `i32` (0=false, 1=true) |
| `Char` | `T_char` | `I32 {I32_Char}` | `i32` (Unicode 码点) |
| `Unit` | `T_unit` | `I32 {I32_Unit}` | `i32` (常量0) |
| `Byte` | `T_byte` | `I32 {I32_Byte}` | `i32` (0-255) |
| `String` | `T_string` | `Ref_string` | `(ref string)` |
| `Bytes` | `T_bytes` | `Ref_bytes` | `(ref string)` |

---

## 3. 结构体布局

```mbt
struct Point {
  x : Int
  y : Int
}
```

### Wasm GC 布局

```wasm
(type $Point (struct (field i32) (field i32)))
```

每个字段按其类型映射：
- `Int` → `i32`
- `Int64` → `i64`
- `Double` → `f64`
- 其他结构体/枚举 → `(ref $Type)`

### 可变字段

```mbt
struct Counter {
  mut value : Int
}
```

```wasm
(type $Counter (struct (field (mut i32))))
```

可变字段使用 Wasm GC 的 `(mut ...)` 标记。

---

## 4. 枚举布局

```mbt
enum Option[T] {
  None
  Some(T)
}
```

### 布局策略

枚举 = tag字段 + payload字段：

```wasm
(type $Option (struct (field i32) (field (ref null $T))))
;;                  tag (0=None, 1=Some)    payload
```

- `None` → `struct.new $Option` with tag=0, payload=null
- `Some(x)` → `struct.new $Option` with tag=1, payload=x

### 枚举访问

枚举的 match 编译为：
```wasm
;; match opt { None => a; Some(v) => b }
block $exit (result ...)
  local.get $opt
  struct.get $Option 0    ;; 读 tag
  br_table 0 1            ;; None→0, Some→1
  ...
end
```

---

## 5. Option 紧凑编码 (`pass_layout.ml`)

### 优化原理

`Option<T>` 的 None/Some 用哨兵值表示，避免堆分配：

```ocaml
(* Option<T> 的紧凑表示 *)
let null ty = Mcore.prim ~ty Pnull []
let is_null e = Mcore.prim ~ty Mtype.T_bool Pis_null [e]
let as_non_null e ty = Mcore.prim ~ty Pas_non_null [e]
let upcast e ty = Mcore.prim ~ty Pidentity [e]
```

### Option<引用类型>

用 `ref null` 表示 `None`：

```
Option<Point>:
  None → ref null
  Some(p) → p (非null引用)
```

```ocaml
(* 构造 None → null *)
let none = Mcore.prim ~ty Pnull []

(* 构造 Some(x) → x *)
let some x = x    (* 直接传递 *)

(* match: is_null → null case, else → Some case *)
```

### Option<值类型>

用哨兵值表示 None：

```
Option<Int>:
  None → -1 (哨兵值)
  Some(5) → 5

Option<Char>:
  None → 0 (U+0000 不是有效 Unicode? → 不，用偏移)
  Some(c) → char_code + 1
```

```ocaml
(* Option<Char> 特殊编码 *)
let i32_to_i64 e = Mcore.prim ~ty:Mtype.T_int64 (Pconvert { ... }) [e]
let i64_to_i32 e = Mcore.prim ~ty:Mtype.T_int (Pconvert { ... }) [e]

(* None → max_i32+1 (0x100000000) *)
(* Some(c) → char_code *)
```

### Mtype 层的标记

```ocaml
type t =
  | T_optimized_option of { elem : t }
  (* 标记 Option 已被优化为紧凑表示 *)
```

---

## 6. 元组布局

```mbt
let p = (1, "hello", true)  // (Int, String, Bool)
```

### 布局

元组编译为固定长度的结构体：

```wasm
(type $Tuple_3 (struct (field i32) (field (ref string)) (field i32)))
```

元组内部的不可变性由 MoonBit 语言的不可变语义保证——所有字段都是不可变的（无 `(mut ...)`）。

---

## 7. 闭包布局

```mbt
let f = fn(x : Int) -> Int { x + captured_var }
```

闭包编译为包含捕获变量和函数指针的结构体：

```wasm
(type $Closure (struct
  (field (ref $Env))     ;; 环境：捕获的变量
  (field (ref func))     ;; 函数指针
))
```

### 调用闭包

```
1. struct.get $Closure 1 → 取 func_ref
2. call_ref              → 调用
```

函数的环境通过函数参数传递：
```
fn inner_lifted(env, x) { env.field + x }
```

---

## 8. Trait 对象布局

```mbt
let showable : Show = obj as Show
```

Trait 对象编译为包含方法表指针的结构体：

```wasm
(type $Show_vtable (struct
  (field (ref func))    ;; to_string 方法
))

(type $Show_object (struct
  (field (ref $Data))        ;; 原始对象
  (field (ref $Show_vtable)) ;; 方法表
))
```

### 方法调用

```
1. struct.get $Show_object 1  → 取 vtable
2. struct.get $Show_vtable 0  → 取方法 func_ref
3. call_ref                    → 调用
```

---

## 9. FixedArray 布局

```mbt
let arr : FixedArray[Int] = [1, 2, 3]
```

```wasm
(type $FixedArray_Int (array (mut i32)))
;; 或者引用包装
(type $FixedArray_Int_ref (struct (field (ref (array (mut i32))))))
```

创建：
```wasm
i32.const 3
array.new_default $FixedArray_Int
```

访问：
```wasm
local.get $arr
i32.const 0    ;; index
array.get $FixedArray_Int
```

---

## 10. 布局优化策略总结

| 优化 | 适用场景 | 效果 |
|------|---------|------|
| Option 紧凑编码 | `Option<T>` 值类型 | 无堆分配 |
| Null 哨兵 | `Option<RefType>` 引用类型 | 无额外结构体 |
| Char Option | `Option<Char>` | 零开销（一个 i32 搞定） |
| 单捕获闭包 | 单变量闭包 | 无环境结构体 |
| Tag + Payload | 枚举 | 一次分配 |
| 方法表 | Trait 对象 | 固定开销 |
