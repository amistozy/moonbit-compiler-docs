# Trait 系统详解

MoonBit 的 Trait 系统类似 Rust，支持 trait 声明、实现、默认方法、父 trait、以及运行时多态（通过 trait object）。相关模块：`trait_decl.ml`、`trait_impl.ml`、`trait_closure.ml`、`method_env.ml`、`ext_method_env.ml`。

---

## 核心概念

```mbt
// 声明 trait
pub trait Show {
  to_string(Self) -> String
}

// 实现 trait
pub impl Show for Point with to_string(self) -> String {
  "(\{self.x}, \{self.y})"
}

// 使用 trait bound
pub fn print[T : Show](x : T) {
  println(@Show.to_string(x))
}
```

---

## 1. Trait 声明 (`trait_decl.ml`)

### 语法 AST (parsing_syntax.ml)

```ocaml
type trait_decl = {
  trait_name : binder;                        (* trait 名 *)
  trait_supers : tvar_constraint list;        (* 父 trait *)
  trait_methods : trait_method_decl list;     (* 方法签名 *)
  trait_vis : visibility;
  ...
}

type trait_method_decl =
  | Trait_method of {
      name : binder;
      has_error : bool;         (* 方法是否可抛出错误 *)
      quantifiers : tvar_binder list;  (* 方法的泛型参数 *)
      params : trait_method_param list;
      return_type : type_expr option;
    }
```

### 类型检查

`trait_decl.ml` 处理 trait 声明到 `Global_env` 的注册：
- 将方法签名存入 `Type_path → method_info` 映射
- 验证父 trait 的存在性
- 检查方法签名中的类型约束

---

## 2. Trait 实现 (`trait_impl.ml`)

### Impl 语法

```ocaml
type impl_decl = {
  self_ty : type_expr option;   (* impl Trait for Type *)
  trait : type_name;             (* 实现的 trait *)
  method_name : binder;          (* 实现的方法名 *)
  params : param list;
  ret_ty : type_expr option;
  body : decl_body;
  ...
}
```

### 实现处理

1. **类型检查**：验证 impl 的方法签名与 trait 声明一致
2. **注册**：将实现存入 `Method_env`（常规方法）或 `Ext_method_env`（扩展方法）
3. **一致性检查**：孤儿规则（至少 trait 或 type 在本地定义）

---

## 3. 方法环境 (`method_env.ml`, `ext_method_env.ml`)

### Method_env

存储 trait 方法的实现映射 `(Type_path, method_name, trait) → method_impl`：

```ocaml
(* 查找方法实现 *)
val find_method_opt :
  monofy_env ->
  type_name:Type_path.t ->
  method_name:string ->
  trait:Type_path.t ->
  method_info option
```

### Ext_method_env

存储扩展方法（`impl Trait with method()` 不带 self type 的形式）。

---

## 4. Trait 闭包 (`trait_closure.ml`)

### 父 Trait 的传递闭包

当 trait 有父 trait 时，需要计算其传递闭包（所有祖先 trait）：

```ocaml
(* 计算 trait 约束的传递闭包 *)
let compute_closure ~types constraints =
  (* BFS 遍历父 trait 关系 *)
  let rec add_super_traits ~loc_ ~supers trait =
    match find_trait_by_path types trait with
    | None -> ()
    | Some trait_decl ->
        List.iter trait_decl.supers ~f:(fun new_trait ->
          if not (visited.mem new_trait) then
            add_to_closure new_trait;
            add_super_traits ~supers:(trait::supers) new_trait)
```

例如，`trait Eq : Show` 意味着要求 `Eq` 的 impl 者也必须实现 `Show`。

---

## 5. 编译流程

### 静态分发（单态化时）

`monofy.ml` 中的 `generate_trait_method` 处理泛型函数中的 trait 方法调用：

```ocaml
let generate_trait_method type_name method_type method_name env ... =
  match find_method_opt ~type_name ~method_name ~trait with
  | Some { prim = Some p } when not (Primitive.is_intrinsic p) ->
      `Prim p                          (* 有prim实现，直接调用 *)
  | Some meth ->
      (* 实例化泛型参数 *)
      let method_type', kind, ty_args = instantiate_method meth in
      `Method (new_binder, meth.prim)  (* 生成单态化调用 *)
  | None -> assert false
```

编译结果：
- 静态调用 → `Mcore.apply func args ~kind:(Normal ...)`
- prim 方法 → `Mcore.prim prim args`

### 动态分发（Trait Object）

`Cexpr_as { expr; trait; obj_type }` → `Cexpr_object { methods_key; self }`：

```ocaml
(* x as Show → trait object *)
method visit_Cexpr_as ctx obj trait obj_ty loc_ =
  let type_ = Type_args.mangle_ty monofied_obj_ty in
  Mcore.make_object ~loc:loc_
    ~methods_key:{ trait; type_ }
    ~ty:trait_mty self
```

Trait object 在 Clam 层成为包含方法表的结构体：
```
Clam: Lallocate(Object { methods: [method1_addr, method2_addr, ...] })
Wasm: (struct (field (ref func)) (field (ref func)) ...)
```

调用时通过索引访问方法表：
```
Clam: Lapply { fn = Object { obj; method_index; method_ty }; args }
Wasm: Struct_get(method_index) → Call_ref
```

---

## 6. 方法解析流程

```
用户代码: @Show.to_string(x)
    │
    ▼
类型检查: Stype.t 推断x的类型 → 查找 Show trait → 解析 to_string
    │
    ▼
Core IR: Cexpr_apply { func = Plocal_method { index; trait; method_name } }
    │
    ▼
单态化: 找到具体类型 → 查找对应 impl → 生成具体调用
    │
    ├── 静态: Cexpr_apply { func = concrete_fn }
    │
    └── 动态: Cexpr_object { methods_key } → 运行时方法表查找
```

---

## 7. 孤儿规则与一致性

MoonBit 的 trait 实现遵循孤儿规则：
- 对于 `impl Trait for Type`，至少 `Trait` 或 `Type` 必须在当前包中定义
- 防止不同包的冲突实现

在 `trait_impl.ml` 中通过检查 `Type_path` 的包前缀来实现。

---

## 8. 与 Rust trait 的对比

| 特性 | MoonBit | Rust |
|------|---------|------|
| 声明语法 | `pub trait T { fn m(Self) }` | `pub trait T { fn m(&self); }` |
| 实现语法 | `impl T for X with method(...) {...}` | `impl T for X { fn method(&self) {...} }` |
| Self 类型 | 显式参数 | 隐式 `&self` |
| 父 trait | `trait Eq : Show` | `trait Eq: PartialEq` |
| 动态分发 | `x as Show` → trait object | `&dyn Show` |
| 泛型约束 | `fn f[T : Show](x: T)` | `fn f<T: Show>(x: T)` |
| 孤儿规则 | ✓ | ✓ |
| 关联类型 | ✗ | ✓ |
