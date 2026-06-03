# Lambda 提升与闭包转换详解

Lambda 提升是 Core IR 优化流水线中倒数第二个 Pass，负责将嵌套的匿名函数（闭包）转换为顶层函数 + 显式环境参数。相关模块：`lambda_lift.ml`。

---

## 什么是 Lambda 提升

```mbt
fn outer(x : Int) -> Int {
  let y = x + 1
  fn inner(z : Int) -> Int {
    x + y + z   // 捕获 x 和 y
  }
  inner(10)
}
```

提升后：
```mbt
fn outer(x : Int) -> Int {
  let y = x + 1
  let env = { x; y }         // 环境打包
  inner_lifted(env, 10)
}

fn inner_lifted(env : Env, z : Int) -> Int {
  env.x + env.y + z
}
```

---

## 1. LambdaLift 核心数据结构

```ocaml
type lift_to_top =
  | Subtop                          (* 提升为 subtop（内嵌于原函数） *)
  | Toplevel of { name_hint : string }  (* 提升为独立顶层函数 *)

type lifter_context = {
  lift_to_top : lift_to_top;
  subtops : Core.subtop_fun_decl Vec.t;       (* 累积的提升函数 *)
  rename_table : Ident.t Ident.Map.t;         (* 重命名表 *)
  exclude : Ident.Set.t;                       (* 排除的标识符 *)
  non_well_knowns : Ident.Hashset.t;
  convert_info : (Ident.t * Stype.t) Ident.Map.t;
}

type capture_info =
  | Single of Ident.t * Stype.t             (* 单个捕获 *)
  | Multiple of (Ident.t * Stype.t) list    (* 多个捕获 *)

type convert_context = {
  env_ty : Stype.t;          (* 环境结构体类型 *)
  captures : capture_info;   (* 捕获的变量 *)
  self_binders : Ident.t list;
}
```

---

## 2. 算法流程

### Step 1: 自由变量分析

```ocaml
let free_vars ctx ~exclude fn =
  let fvs = Core_util.free_vars ~exclude fn in
  let dedup_map = Ident.Hash.create 17 in
  List.iter fvs ~f:(fun (id, ty) ->
    let id, ty = Ident.Map.find_default ctx.convert_info id id, ty in
    if not (Ident.Hash.mem dedup_map id || Ident.Set.mem exclude id) then
      Ident.Hash.add dedup_map id ty);
  Ident.Hash.to_list dedup_map
```

`Core_util.free_vars` 计算一个表达式中引用的所有未被绑定的变量（自由变量）。

### Step 2: 环境打包

根据捕获变量数量选择打包策略：

- **单个捕获** → 直接传递该变量（无需额外打包）
- **多个捕获** → 打包为记录结构体

```ocaml
match captures with
| Single (cap_id, _) ->
    (* 单捕获：直接使用 cap_id *)
    let env_binder = Ident.rename cap_id in
    ...
| Multiple _ ->
    (* 多捕获：创建环境结构体 *)
    let env_binder = Ident.fresh "*env" in
    let env_ty = T_constr { type_constructor = ... } in
    ...
```

### Step 3: 闭包体转换 (`convert_fn`)

```ocaml
let convert_fn ctx fn convert_ctx ~visit_expr =
  let { env_ty; captures; self_binders } = convert_ctx in
  (*
    1. 添加 env 参数到函数参数列表
    2. 将闭包体内对捕获变量的引用替换为 env 的字段访问
    3. 生成新的顶层/subtop 函数
  *)
```

### Step 4: 闭包创建点替换

在原始闭包创建点：

```ocaml
(* 原始: Cexpr_function { func; ... } *)
(* 替换为: 创建环境 + 创建闭包记录 *)

(* 环境创建 *)
let env = Cexpr_record {
  fields = captures |> List.map (fun (id, _) -> { label; pos; expr = Cexpr_var id })
}

(* 闭包记录 = { env, func_ptr } *)
Cexpr_record {
  fields = [env_field; func_field]
}
```

---

## 3. Subtop vs Toplevel

| 提升级别 | 条件 | 位置 |
|---------|------|------|
| Subtop | 闭包定义在函数内 | 作为 `top_fun_decl.subtops` 一起编译 |
| Toplevel | 闭包定义在顶层 | 独立 `Ctop_fn` |

```ocaml
type lift_to_top = Subtop | Toplevel of { name_hint : string }
```

---

## 4. 递归闭包

```mbt
fn make_counter() -> (() -> Int) {
  let mut count = 0
  fn counter() -> Int {
    count = count + 1
    count
  }
  counter
}
```

对于递归闭包，需要特殊处理：
- `self_binders` 包含闭包自身的引用
- 环境打包时需要包括闭包自身的引用

---

## 5. 与后续 Pass 的关系

LambdaLift 必须在以下 Pass 之后运行：
- **Contification**（join 函数不需要提升）
- **PropagateConstr**（构造器可能影响闭包分析）
- **InlineSingleUseJoin**（简化闭包调用图）

LambdaLift 之后：
- **DCE**（消除提升后不再使用的中间变量）
- 最终 Core IR → 单态化 → Mcore → Clam

在 Clam 层，闭包已经完全是显式的：
```ocaml
Lclosure { captures; address; tid }
(* 闭包 = 结构体 { 捕获1; 捕获2; ...; func_ptr } *)
```

---

## 6. 性能考虑

### 单捕获优化

```ocaml
| Single (cap_id, _) ->
    (* 不需要额外分配环境结构体 *)
    let env_binder = Ident.rename cap_id in
    ...
```

当闭包只捕获一个变量时，直接传递该变量，避免额外的记录分配。

### 多捕获

```ocaml
| Multiple captures ->
    (* 需要分配环境结构体 *)
    let env_ty = T_constr { type_constructor = fresh_type_path } in
    ...
```

多个捕获变量被打包为一个堆分配的记录（在 Clam 层成为 Wasm GC 的 struct）。

---

## 7. 完整示例

```mbt
fn map_add(n : Int, arr : Array[Int]) -> Array[Int] {
  arr.map(fn(x) { x + n })
}
```

LambdaLift 后：

```ocaml
(* 主函数 *)
Ctop_fn map_add:
  let env = { n } in
  let closure = { env, &map_add_inner } in
  arr.map(closure)

(* 提升的函数 *)
subtop map_add_inner(env, x):
  env.n + x
```

Clam 层：

```ocaml
(* 闭包结构体 *)
Lallocate(Struct) = { env_ref, func_ref }

(* 调用 *)
Lapply {
  fn = Object { obj=closure; method_index=0 };
  args = [x]
}
→
Struct_get(closure, 1)  (* 取 func_ref *)
Call_ref(...)           (* 调用 *)
```
