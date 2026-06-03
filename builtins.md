# 内建类型与基元操作详解

MoonBit 编译器在 `builtin.ml` 中定义了一组语言级内建类型，在 `primitive.ml` 中定义了一组基元操作。这些是编译器的"硬编码"知识，不依赖标准库。

---

## 1. 内建类型 (`builtin.ml`)

### 类型列表

```ocaml
let builtin_types = [
  ty_constr_option;       (* Option[T] *)
  ty_constr_fixedarray;   (* FixedArray[T] *)
  ty_constr_ref;          (* Ref[T] *)
  ty_constr_result;       (* Result[T, E] *)
  ty_constr_error;        (* Error *)
  ty_constr_func_ref;     (* FuncRef *)
]
```

### Option[T]

```ocaml
(* None 构造器 *)
let constr_none = {
  constr_name = "None";
  cs_args = [];              (* 无参数 *)
  cs_tag = { index = 0; total = {0,1} };  (* tag=0 *)
  cs_vis = Read_write;
}

(* Some 构造器 *)
let constr_some = {
  constr_name = "Some";
  cs_args = [generic_var];   (* 一个泛型参数 *)
  cs_tag = { index = 1 };
}

let ty_constr_option = {
  ty_constr = type_path_option;   (* @builtin.Option *)
  ty_arity = 1;
  ty_desc = Variant_type [constr_none; constr_some];
}
```

Option 在 Mtype 层有特殊优化：`T_optimized_option { elem }` 表示紧凑编码（无堆分配）。

### Ref[T]

```ocaml
(* Ref 是一个含 mut val 字段的单字段记录 *)
let field_val = {
  field_name = "val";
  pos = 0;
  ty_field = generic_var;
  mut = true;               (* 可变字段 *)
  vis = Read_write;
}

let ty_constr_ref = {
  ty_constr = type_path_ref;   (* @builtin.Ref *)
  ty_arity = 1;
  ty_desc = Record_type { fields = [field_val] };
}
```

`Ref[T]` 提供了 ML 风格的引用单元。在 Wasm GC 中编译为一个包含可变字段的结构体。

### Result[T, E]

```ocaml
(* Result 是二值枚举: Ok(T) | Err(E) *)
let constr_ok = {
  constr_name = "Ok";
  cs_args = [generic_var_1];   (* T *)
  cs_tag = { index = 1 };      (* tag=1 *)
}

let constr_err = {
  constr_name = "Err";
  cs_args = [generic_var_2];   (* E *)
  cs_tag = { index = 0 };      (* tag=0 *)
}
```

`Result[T, E]` 在 Mtype 层有特殊类型：`T_error_value_result { ok; err; id }`。

### Error

```ocaml
let ty_constr_error = {
  ty_constr = type_path_error;   (* @builtin.Error *)
  ty_arity = 0;
  ty_desc = Abstract_type;
}
```

`Error` 是顶层错误类型。所有 `type!` 声明产生的 suberror 自动继承 `Error`。

### FixedArray[T]

```ocaml
let ty_constr_fixedarray = {
  ty_constr = type_path_fixedarray;
  ty_arity = 1;
  ty_desc = Abstract_type;
}
```

`FixedArray` 在 Mtype 中有对应类型 `T_fixedarray`，映射为 Wasm GC 的 `(array ...)`。

### 其他内建

```ocaml
(* 元组类型 *)
let type_product ts = T_constr { type_constructor = Type_path.tuple n; tys = ts }

(* 数组类型 *)
let type_array = f type_path_array

(* 函数引用 *)
let type_arrow t1 t2 ~err_ty ~is_async = Tarrow { ... }
```

---

## 2. 内建值

```ocaml
let builtin_values = Typing_info.make_values ()

(* 注册构造器和字段 *)
let _ =
  List.iter [constr_none; constr_some; constr_err; constr_ok]
    ~f:(Typing_info.add_constructor builtin_values);
  Typing_info.add_field builtin_values ~field:field_val
```

---

## 3. 基元操作 (`primitive.ml`)

### 操作数类型

```ocaml
type operand_type = I32 | I64 | U32 | U64 | F64 | F32 | U8 | U16 | I16
```

对应 MoonBit 的各种整数和浮点类型。

### 完整基元列表

```ocaml
type prim =
  (* 控制流 *)
  | Pignore              (* 忽略值 *)
  | Pidentity            (* 类型强转（无运行时开销） *)
  | Pnot                 (* 布尔取反 *)
  | Ppanic               (* 运行时 panic *)
  | Punreachable         (* 不可达代码 *)

  (* 转换 *)
  | Pconvert of { kind; from; to_ }  (* 数值类型转换 *)
  | Pcast of { kind }                (* 构造器/enum/类型转换 *)

  (* 算术 *)
  | Parith of { operand_type; operator }      (* + - * / % *)
  | Pbitwise of { operand_type; operator }    (* & | ^ << >> ~ *)
  | Pcomparison of { operand_type; operator }  (* == != < > <= >= *)
  | Pcompare of operand_type                  (* 三路比较 *)

  (* 字符串 *)
  | Pgetstringitem of { safe }    (* s[i] *)
  | Pstringlength                 (* s.length() *)
  | Pstringequal                  (* s1 == s2 *)

  (* 字节 *)
  | Pmakebytes                    (* Bytes::make *)
  | Pbyteslength                  (* b.length() *)
  | Pgetbytesitem of { safe }     (* b[i] *)
  | Psetbytesitem of { safe }     (* b[i] = x *)
  | Pbytesequal                   (* b1 == b2 *)

  (* FixedArray *)
  | Pfixedarray_length                   (* a.length() *)
  | Pfixedarray_make of { kind }         (* 创建 fixedarray *)
  | Pfixedarray_get_item of { kind }     (* a[i] *)
  | Pfixedarray_set_item of { set_kind } (* a[i] = x *)

  (* Array (可增长) *)
  | Parray_make                          (* Array::make *)

  (* Null/引用 *)
  | Pnull                    (* null 值 *)
  | Pnull_string_extern      (* null 字符串/外部引用 *)
  | Pis_null                 (* 判断是否为 null *)
  | Pas_non_null             (* 断言非 null *)
  | Prefeq                   (* 引用相等 *)
  | Pclosure_to_extern_ref   (* 闭包转 externref *)
  | Praw_func_to_func_ref    (* 原始函数转 funcref *)

  (* Enum 操作 *)
  | Penum_field of { index; tag }       (* 读取 enum 字段 *)
  | Pset_enum_field of { index; tag }   (* 写入 enum 字段 *)

  (* Error 操作 *)
  | Pmake_value_or_error of { tag }     (* 构造 Ok/Err *)
  | Perror_to_string                    (* Error→String *)

  (* Trait 方法调用 *)
  | Pcall_object_method of { method_index; method_name }

  (* Async *)
  | Pget_current_continuation   (* 获取当前 continuation *)
  | Prun_async                  (* 运行 async 任务 *)

  (* 调试 *)
  | Pprintln                    (* println *)
  | Pany_to_string              (* any→String *)

  (* FFI *)
  | Pccall of { arity; func_name }    (* C 函数调用 *)

  (* 内建 intrinsic *)
  | Pintrinsic of Moon_intrinsic.t
```

### Make Array Kind

```ocaml
type make_array_kind = LenAndInit | EverySingleElem | Uninit
(* 创建 FixedArray 的不同方式 *)
```

### Array Get/Set Kind

```ocaml
type array_get_kind = Safe | Unsafe | Rev_unsafe
type array_set_kind = Null | Default | Value | Unsafe
```

### Convert Kind

```ocaml
type convert_kind = Convert | Saturate | Reinterpret
(* i32→i64 的不同模式 *)
```

---

## 4. 编译器内建函数 (`moon_intrinsic.ml`)

编译器将一组标准库函数识别为 intrinsic，允许在编译时期进行特殊优化：

```ocaml
type t =
  (* 字符串 *)
  | Char_to_string         (* %char.to_string *)
  | F64_to_string          (* %f64.to_string *)
  | String_substring       (* %string.substring *)

  (* FixedArray 操作 *)
  | FixedArray_join        (* %fixedarray.join *)
  | FixedArray_iter | FixedArray_iteri | FixedArray_map
  | FixedArray_fold_left | FixedArray_copy | FixedArray_fill

  (* Iter 操作 *)
  | Iter_map | Iter_iter | Iter_from_array | Iter_take
  | Iter_reduce | Iter_flat_map | Iter_repeat
  | Iter_filter | Iter_concat

  (* Array 操作 *)
  | Array_length | Array_get | Array_unsafe_get
  | Array_set | Array_unsafe_set

  (* View 操作 *)
  | ArrayView_length | ArrayView_unsafe_get | ArrayView_unsafe_set
  | ArrayView_unsafe_as_view
  | BytesView_length | BytesView_unsafe_get | BytesView_unsafe_as_view
```

命名约定：`%module.function`。

---

## 5. 名称修饰 (`name_mangle.ml`)

MoonBit 编译器对生成的低级符号进行名称修饰（name mangling），确保唯一性并包含类型信息：

```ocaml
(* 类型名生成 *)
let make_type_name (t : Mtype.t) =
  match ty with
  | T_int -> "Int"
  | T_optimized_option { elem = T_char } -> "Option<Char>"
  | T_func { params; return } -> "<Int*String>=>Bool"
  | T_constr id -> id
  | ...

(* 构造器名生成 *)
let make_constr_name (enum_name : Tid.t) (tag : Tag.t) =
  Tid.of_string (enum_name ^ "." ^ tag.name_)
```

修饰后的名称如：
- `Option<Int>` — Option 在 Int 上的单态实例
- `MyEnum.VariantA` — 枚举构造器
- `<Int*String>=>Bool` — 函数类型

---

## 6. 常量 (`constant.ml`)

编译器内部使用的常量表示，支持编译期运算：

```ocaml
type t =
  | C_bool of bool
  | C_char of Uchar.t
  | C_int of { v : int32; repr : string option }
  | C_byte of { v : int; repr : string option }
  | C_int64 of { v : int64; repr : string option }
  | C_uint of { v : UInt32.t; repr : string option }
  | C_uint64 of { v : UInt64.t; repr : string option }
  | C_float of { v : float; repr : string option }
  | C_double of { v : float; repr : string option }
  | C_string of string
  | C_bytes of { v : string; repr : string option }
  | C_bigint of { v : BigInt.t; repr : string option }
```

还支持编译期常量的算术、比较、位运算等求值（`eval_arith`、`eval_comparison`、`eval_bitwise` 等）。
