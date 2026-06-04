# Clam IR 生成：从 Core IR 到 Clam IR

`clam_of_core.ml`（1410行）将 Mcore（单态化后的 Core IR）翻译为 Clam IR——一种接近 Wasm GC 虚拟机模型的低级中间表示。这是编译器后端的第一阶段，将高级控制流降级到可直接映射为 Wasm GC 指令的形态。

---

## 在流水线中的位置

```
Core.program (多包链接后)
    │
    ▼
单态化 (monofy)
    │
    ▼
Mcore.t (单态化后的Core IR，已消除泛型)
    │
    ▼
clam_of_core.ml  ← 【本模块】
    │
    ▼
Clam.prog (Clam IR)
    │
    ▼
Clam优化Pass → Wasm GC Dwarfsm编码 → .wasm
```

---

## 1. Clam IR 核心数据结构

### 1.1 程序结构

```ocaml
type prog = {
  fns : top_func_item list;           (* 顶层函数定义 *)
  main : lambda option;               (* main入口 *)
  init : lambda;                      (* 初始化代码 *)
  globals : (binder * constant option) list;  (* 全局变量 *)
  type_defs : type_defs;              (* 类型定义表 *)
}
```

### 1.2 顶层函数

```ocaml
type top_func_item = {
  binder : address;         (* Fn_address.t 函数地址 *)
  fn_kind_ : fn_kind;       (* Top_pub "name" | Top_private *)
  fn : fn;                  (* 函数签名+体 *)
  tid : tid option;         (* 类型标识符 *)
}

type fn = {
  params : binder list;          (* 参数列表 *)
  body : lambda;                 (* 函数体 *)
  return_type_ : ltype list;     (* 返回类型 *)
}
```

### 1.3 Lambda 表达式（Clam IR 指令集）

Clam IR 的 `lambda` 类型包含 30+ 种指令变体，可直接映射为 Wasm GC 指令：

```ocaml
type lambda =
  (* 基本操作 *)
  | Lconst of constant                                  (* 常量 *)
  | Lvar of { var : var }                               (* 变量引用 *)
  | Lassign of { var : var; e : lambda }                (* 变量赋值 *)
  | Llet of { name : binder; e : lambda; body : lambda } (* let绑定 *)
  | Lsequence of { exprs : lambda list; last_expr : lambda } (* 顺序执行 *)

  (* 控制流 *)
  | Lif of { pred : lambda; ifso : lambda; ifnot : lambda; type_ : ltype }
  | Lswitch of { obj : var; cases : (tag * lambda) list; default : lambda; type_ : ltype }
  | Lswitchint of { obj : var; cases : (int * lambda) list; default : lambda; type_ : ltype }
  | Lswitchstring of { obj : lambda; cases : (string * lambda) list; default : lambda; type_ : ltype }
  | Lloop of { params : binder list; body : lambda; args : lambda list; label : label; type_ : ltype }
  | Lbreak of { arg : lambda option; label : label }
  | Lcontinue of { args : lambda list; label : label }

  (* 函数调用 *)
  | Lapply of { fn : target; prim : intrinsic option; args : lambda list }
  | Lstub_call of { fn : func_stubs; args : lambda list; params_ty : ltype list; return_ty : ltype option }
  | Lreturn of lambda

  (* Join点（类似continuation） *)
  | Ljoinlet of { name : join; params : binder list; e : lambda; body : lambda; kind : join_kind; type_ : ltype list }
  | Ljoinapply of { name : join; args : lambda list }

  (* 闭包支持 *)
  | Lclosure of closure          (* 创建闭包 *)
  | Lclosure_field of { ... }    (* 访问闭包捕获字段 *)
  | Lget_raw_func of address     (* 获取原始函数指针 *)

  (* 内存操作 *)
  | Lallocate of aggregate       (* 分配对象 *)
  | Lget_field of { obj : lambda; tid : tid; index : int; kind : get_field_kind }
  | Lset_field of { obj : lambda; field : lambda; tid : tid; index : int; kind : set_field_kind }
  | Lmake_array of { tid : tid; kind : make_array_kind; elems : lambda list }
  | Larray_get_item of { ... }   (* 数组索引读取 *)
  | Larray_set_item of { ... }   (* 数组索引写入 *)

  (* 其他 *)
  | Lprim of { fn : prim; args : lambda list }  (* 原生操作 *)
  | Lcatch of { body : lambda; on_exception : lambda; type_ : ltype }  (* 异常捕获 *)
  | Lcast of { expr : lambda; target_type : ltype }  (* 类型转换 *)
  | Levent of { expr : lambda; loc_ : location }      (* 调试源位置 *)
```

### 1.4 调用目标

```ocaml
type target =
  | Dynamic of var           (* 动态调用（通过变量） *)
  | StaticFn of address      (* 静态调用（已知函数地址） *)
  | Object of { obj : var; method_index : int; method_ty : ltype }  (* 对象方法调用 *)
```

### 1.5 聚合类型分配

```ocaml
type alloc_kind =
  | Tuple                    (* 元组 *)
  | Struct                   (* 结构体/记录 *)
  | Enum of { tag : constr_tag }  (* 枚举/构造器 *)
  | Object of { methods : address list }  (* 对象（含方法表） *)
```

---

## 2. 两遍翻译架构

`transl_prog` 采用两遍策略：

### 第一遍：预收集

```ocaml
let transl_prog prog =
  (* 0. 收集非well-known局部函数（被动态引用的闭包） *)
  collect_local_non_well_knowns local_non_well_knowns prog;

  (* 1. 翻译类型定义：Mtype.defs → Ltype.type_defs *)
  let type_defs = Transl_mtype.transl_mtype_defs types in

  (* 2. 收集地址信息：为每个顶层函数分配Fn_address和参数/返回类型 *)
  let addr_tbl = Addr_table.create 17 in
  Lst.iter body ~f:(collect_top_func ~type_defs ~addr_tbl);

  (* 3. 构建对象方法包装器 *)
  let object_methods = Object_util.Hash.create 17 in
  ...
```

### 第二遍：翻译

```ocaml
  (* 4. 翻译main *)
  let main = Option.map (transl_expr ...) main in

  (* 5. 翻译top_item列表 *)
  let result = Lst.fold_right body acc (transl_top_item ...) in

  (* 6. 生成闭包包装器和初始化代码 *)
  Addr_table.fold addr_tbl (...) (...)
```

---

## 3. 表达式翻译：`transl_expr`

```ocaml
let rec transl_expr ~name_hint ~mtype_defs ~addr_tbl ~type_defs ~object_methods x =
```

### 3.1 翻译对照表

| Mcore 表达式 | Clam IR 指令 | 翻译要点 |
|---|---|---|
| `Cexpr_const` | `Lconst` | 常量直接映射 |
| `Cexpr_unit` | `Lconst (C_int 0)` | unit=32位0 |
| `Cexpr_var` | `Lvar { var }` | 变量：查addr_tbl转换标识符 |
| `Cexpr_and` | `Lif { pred=lhs; ifso=rhs; ifnot=false }` | 短路与 |
| `Cexpr_or` | `Lif { pred=lhs; ifso=true; ifnot=rhs }` | 短路或 |
| `Cexpr_let` | `Llet { name; e; body }` | let绑定直接映射 |
| `Cexpr_letfn Nonrec` | `Llet + Lclosure` 或 `Lalloc(Struct)` | 非递归局部函数=闭包 |
| `Cexpr_letfn Rec` | `Llet + Lclosure` 或 `Lalloc(Struct)` | 递归：自身作为捕获 |
| `Cexpr_letfn (Tail_join\|Nontail_join)` | `Ljoinlet` | Join点转换为Clam joinlet |
| `Cexpr_letrec` | `Lletrec { names; fns; body }` | 互递归let |
| `Cexpr_function (is_raw=true)` | `Lget_raw_func` + 新建顶层函数 | 原始函数指针 |
| `Cexpr_function (is_raw=false)` | `Lclosure` | 匿名函数=闭包 |
| `Cexpr_apply (Join)` | `Ljoinapply` | join调用 |
| `Cexpr_apply (Normal)` | `Lapply (StaticFn\|Dynamic)` | 函数调用 |
| `Cexpr_object` | `Lallocate { kind=Object { methods } }` | 对象构造 |
| `Cexpr_constr` | `Lallocate { kind=Enum { tag } }` | 构造器=分配枚举 |
| `Cexpr_tuple` | `Lallocate { kind=Tuple }` | 元组=分配元组 |
| `Cexpr_record` | `Lallocate { kind=Struct }` | 记录=分配结构体 |
| `Cexpr_record_update` | `Lallocate { kind=Struct }` + `Lget_field` | 记录更新=复制+修改 |
| `Cexpr_field` | `Lget_field` | 字段访问 |
| `Cexpr_mutate` | `Lset_field` | 可变字段写入 |
| `Cexpr_array` | `Lmake_array` | 定长数组构造 |
| `Cexpr_assign` | `Lassign` | 变量赋值 |
| `Cexpr_sequence` | `Lsequence` | 语句序列 |
| `Cexpr_if` | `Lif` | 条件分支 |
| `Cexpr_switch_constr` | `Lswitch` | 构造器匹配（编译为Wasm br_on_cast） |
| `Cexpr_switch_constant` | `Lswitchint` / `Lswitchstring` / `Lif`链 | 常量匹配 |
| `Cexpr_loop` | `Lloop` | 循环节点 |
| `Cexpr_break` | `Lbreak` | 循环退出 |
| `Cexpr_continue` | `Lcontinue` | 循环继续 |
| `Cexpr_return (Single_value)` | `Lreturn` | 简单返回 |
| `Cexpr_return (Error_result)` | `Lreturn (Lalloc(Enum { tag }))` | Result返回=分配+返回 |
| `Cexpr_handle_error` | `Lswitch` (解构Result) | 错误处理=枚举switch |

---

## 4. Primitive 翻译

`Cexpr_prim`中的primitive在翻译时进行语义降级：

| Primitive | Clam IR 指令 | 说明 |
|---|---|---|
| `Pmake_value_or_error { tag }` | `Lallocate { kind=Enum { tag } }` | Result构造 |
| `Prefeq` | `Lprim(Pcomparison\|Pstringequal\|Prefeq)` | 根据类型选择相等比较 |
| `Pcast (Constr_to_enum\|Make_newtype)` | 直接使用参数（恒等） | 零成本转换：消除 |
| `Pcast (Unfold_rec_newtype\|Enum_to_constr)` | `Lcast` | 需要实际类型转换 |
| `Penum_field` | `Lget_field { kind=Enum }` | 枚举字段访问 |
| `Pset_enum_field` | `Lset_field { kind=Enum }` | 枚举字段设置 |
| `Pcatch` | `Lcatch` | Wasm try-catch |
| `Pnull` | `Lprim(Pnull\|Pnull_string_extern)` | JS模式下字符串null特化 |
| `Parray_make` | `Lmake_array` / 展开为循环 | 数组构造 |
| `Pfixedarray_make` | `Lmake_array` | 定长数组构造 |
| `Pfixedarray_get_item` | `Larray_get_item` / `Lprim(Pgetbytesitem)` | 字节数组特化 |
| `Pfixedarray_set_item` | `Larray_set_item` | 数组写入 |
| `Pcall_object_method` | `Lapply { fn=Object { ... } }` | 对象方法调用 |

---

## 5. 闭包翻译策略

Clam IR 中的闭包翻译是区分不同场景的核心：

### 5.1 非Well-Known闭包（`local_non_well_knowns`）

被动态引用的局部函数，使用**抽象闭包类型**：

```
Mcore:
  let f = fn(x) { x + captured_var }
  f(1)

Clam:
  let f = closure { captures = [captured_var]; address = fn_addr }
  Lapply(Dynamic f, 1)

顶层函数 fn_addr(env, x):
  Lclosure_field(env, 0)  // 获取captured_var
  Padd(x, captured_var)
```

使用 `closure_of_fn` 函数处理：
- **无自由变量**：环境参数为`*env : Ref_closure_abstract`，函数体忽略
- **有自由变量**：创建具体闭包类型 `Ref_closure{ fn_sig_tid; captures }`，用`Lclosure_field`读取捕获变量

### 5.2 Well-Known闭包

编译器知道所有引用点、不会被外部看到的局部函数。无需独立闭包对象：

```
Mcore:
  let f = fn(x) { x + captured_var }
  f(1)

Clam:
  // 单自由变量：
  let f = captured_var           // 直接就是捕获值
  let fn_addr(captured_var, x) = 
    Padd(x, captured_var)
  Lapply(StaticFn fn_addr, f, 1)  // 静态调用

  // 多自由变量：
  let f = Lalloc(Struct, [v1, v2])  // 打包为结构体
  let fn_addr(env, x) = 
    Lget_field(env, 0)  // v1
    Lget_field(env, 1)  // v2
    ...
```

`well_known_closure_of_fn` 按自由变量数分三种情况：
1. **0个FV**：环境为`unit`，自由变量不必传递
2. **1个FV**：直接将自由变量作为闭包的值，自身替换为该FV
3. **2+个FV**：分配`Ref_struct`打包自由变量

### 5.3 互递归Well-Known闭包 (`well_known_closure_of_mut_rec_fn`)

使用 `Ref_late_init_struct` 类型，通过 `fix_single_var` 将递归自引用替换为环境变量。

### 5.4 闭包包装器 (`make_top_closure_item`)

为导出函数的闭包形式生成包装函数：

```
顶层函数 fn(env, params...) { body }
闭包包装器 closure_wrapper(env, params...) { Lapply(StaticFn fn, env, params...) }
```

---

## 6. 对象方法翻译

Trait对象方法（OOP风格）翻译为对象分配：

```ocaml
let make_object_wrapper ... abstract_obj_tid ... method_item =
  (* 为每个方法生成包装函数 *)
  let fn(self, params...) = 
    Lget_field(  // 提取self字段
      Lcast(obj, concrete_obj_tid),  // 转换到具体类型
      index=0, kind=Object
    )
    // 调用原始方法
    Lapply(StaticFn method_addr, self, params...)
  in ...
```

最终生成：
```
object_instance = Lallocate {
  kind = Object { methods = [method_wrapper_addrs...] };
  fields = [self_value]
}
```

---

## 7. 顶层项翻译

### 7.1 `Ctop_expr`

顶层表达式直接翻译为Clam表达式，追加到`init`序列中。

### 7.2 `Ctop_let`

- 如果是`Cexpr_function`：作为顶层函数处理
- 如果是简单常量（`Lconst`原始类型）：放入`globals`表
- 否则：放入`globals(name, None)`并生成初始化代码 `Llet { name; e; body = init }`

### 7.3 `Ctop_fn` / `Ctop_stub`

- `Ctop_fn`：创建顶层函数项，body来自Mcore表达式
- `Ctop_stub`：FFI桩，生成`Lstub_call`包装，函数类型参数转为`Ref_extern`

---

## 8. 类型翻译：`Transl_mtype_gc`

Mtype（单态化类型）→ Ltype（Wasm GC 线性类型）的映射：

| Mtype | Ltype | 说明 |
|---|---|---|
| `T_int` | `I32` | 整数 |
| `T_bool` | `I32` (i32_bool) | 布尔→i32 |
| `T_char` | `I32` | 字符→i32 |
| `T_int64` | `I64` | 64位整数 |
| `T_double` | `F64` | 浮点数 |
| `T_float` | `F32` | 单精度浮点 |
| `T_bytes` | `Ref_bytes` | 字节数组 |
| `T_string` | `Ref_string` / `Ref_nullable(tid_string)` | 字符串 |
| `T_unit` | `I32` (i32_unit) | unit→0 |
| `T_constr { type_constructor = tid }` | `Ref { tid }` | 构造器类型 |
| `T_tuple _` | `Ref { tid }` | 元组 |
| `T_fixedarray _` | `Ref { tid }` | 定长数组 |
| `T_func _` | `Ref_closure_abstract / Ref_extern` | 函数类型 |
| `T_trait _` | `Ref_any` | trait对象 |
| `T_raw_func _` | `Ref_func` | 原始函数指针 |

---

## 9. 与 Core IR 的关键差异

| 方面 | Core IR (Mcore) | Clam IR |
|---|---|---|
| 标识符 | `Core_ident.t` | `Clam_ident.t`（携带Ltype） |
| 类型 | `Mtype.t`（高层类型） | `Ltype.t`（Wasm GC线性类型） |
| 函数调用 | `Cexpr_apply { kind = Normal\|Join }` | `Lapply { fn = StaticFn\|Dynamic\|Object }` 或 `Ljoinapply` |
| 闭包 | 隐式的letfn | 显式的`Lclosure` + `Lclosure_field` |
| 内存分配 | 无显式分配 | `Lallocate` 显式分配 |
| join点 | `letfn kind=Tail_join\|Nontail_join` | `Ljoinlet` |
| 模式匹配 | `Cexpr_switch_constr` | `Lswitch`（直接对应Wasm br_on_cast） |
| 可变变量 | `Cexpr_assign` | `Lassign` |
| 异常 | `Cexpr_handle_error` | `Lcatch` |
| 返回 | `Cexpr_return { return_kind }` | `Lreturn expr`（Result分配已显式化） |

---

## 10. 初始化与全局变量

`transl_prog` 最后阶段处理全局变量初始化：

```ocaml
type translate_result = {
  globals : (Ident.t * Constant.t option) list;  (* None=需运行时初始化 *)
  init : Clam.lambda;     (* 初始化代码 *)
  test : Clam.lambda;     (* 测试代码 *)
}
```

- 简单常量全局变量：直接放入`globals(name, Some const)`，在Wasm数据段初始化
- 需计算的全局变量：`globals(name, None)` + `Llet { name; e = init_expr; body = init }`

---

## 总结

`clam_of_core.ml` 完成了 Core IR → Clam IR 的关键转型：

1. **类型降级**：Mtype高维类型 → Ltype线性类型（Wasm GC兼容）
2. **闭包显式化**：隐式函数闭包 → 显式的 `Lclosure`/`Lclosure_field` 操作
3. **分配显式化**：构造器/元组/记录 → `Lallocate` 内存分配指令
4. **控制流简化**：join点 → `Ljoinlet`，模式匹配 → `Lswitch`（br_on_cast）
5. **调试信息**：无条件添加 `Levent` 包装（debug模式）或省略
