# 错误处理与 Async 变换详解

MoonBit 使用**类型化错误（checked errors）**机制，类似于 Result 类型但在语言层面深度集成。Error 类型是 Error trait 的子类型。Async 函数通过 CPS 变换编译为状态机。相关模块：`eliminate_async.ml`、`core_of_tast.ml`（错误处理翻译）。

---

## 1. 类型化错误系统

### 错误声明

```mbt
// 声明错误类型（suberror of Error）
suberror ParseError {
  InvalidChar(Char)
  InvalidEof
}
derive(Show)  // 自动实现 Error trait

// 函数声明可抛出错误
pub fn parse(s : String) -> Int!ParseError {
  if s == "" { raise ParseError::InvalidEof }
  ...
}
```

`!ParseError` 语法糖展开为返回 `Result[Int, ParseError]`。

### 错误类型 AST

```ocaml
(* parsing_syntax.ml *)
type type_desc =
  | Ptd_error of exception_decl    (* type! 错误类型 *)

type exception_decl =
  | No_payload                     (* 无payload *)
  | Single_payload of type_expr    (* 单个payload *)
  | Enum_payload of constr_decl list  (* 枚举式payload *)
```

`type!` 声明的类型自动成为 `Error` 的子类型：

```ocaml
(* stype.ml *)
type t =
  | T_constr of { ...; is_suberror_ : bool; ... }
  (* is_suberror_ = true 表示这是 Error 的子类型 *)
```

---

## 2. 错误传播机制

### 三种错误处理方式

```mbt
// 1. 自动传播（无需标记）
fn f() -> Int!ParseError {
  let x = parse("123")  // 错误自动向上传播
  x + 1
}

// 2. try! 中断（不声明raise）
fn g() -> Int {
  let x = try! parse("123")  // 出错则panic
  x + 1
}

// 3. try-catch 显式处理
fn h() -> Int {
  parse("123") catch {
    ParseError::InvalidEof => -1
    _ => 0
  }
}

// 4. 转换为 Result[T, Error]（调试用）
let result : Result[Int, Error] = try? parse("123")
```

### Core IR 中的错误表示

```ocaml
type return_kind =
  | Error_result of { is_error : bool; return_ty : typ }
  | Single_value
  (* 函数返回 Error_result 表示可能返回错误 *)

type handle_kind =
  | To_result                          (* try? → Result *)
  | Joinapply of var                   (* 错误传播到 continuation *)
  | Return_err of { ok_ty : typ }      (* try! → 错误时panic *)
```

### 翻译示例

**`raise ParseError::InvalidEof`**：
```ocaml
Cexpr_return {
  expr = Cexpr_constr { tag = InvalidEof_tag; ... };
  return_kind = Error_result { is_error = true; return_ty };
}
```

**`try! expr`**：
```ocaml
Cexpr_handle_error {
  obj = expr;
  handle_kind = Return_err { ok_ty };
}
→ 如果是错误结果 → raise Failure("unwrap error")
```

**`try? expr`**：
```ocaml
Cexpr_handle_error {
  obj = expr;
  handle_kind = To_result;
}
→ 包装为 Result 类型
```

**`expr catch { pattern => handler }`**：
```ocaml
Cexpr_switch_constr {
  obj = expr;  (* 假设expr返回 Error_result *)
  cases = [
    (InvalidEof_tag, Some binder, handler),
    ...
  ];
  default = Some re_raise  (* 未匹配的错误重新raise *)
}
```

---

## 3. 自动错误传播

MoonBit 编译器**自动推断**哪些调用可能产生错误并向上传播：

```ocaml
(* 在 core_of_tast.ml 中 *)
(* 当调用一个 raise 函数且调用者不处理错误时 *)
(* 自动插入错误传播 join *)
Cexpr_handle_error {
  obj = call_expr;
  handle_kind = Joinapply error_join;
}
```

这意味：
- 调用者不需要显式标记 `try`（不同于 Swift）
- 编译器追踪整个调用链中的错误流
- 错误在函数边界自动传播

---

## 4. Async 变换 (`eliminate_async.ml`)

### CPS (Continuation-Passing Style) 变换

MoonBit 的 `async` 通过 CPS 变换编译为同步状态机：

```mbt
async fn fetch_data(url : String) -> String {
  let response = await http_get(url)
  let data = await parse_response(response)
  data
}
```

变换为状态机：
```
State 0: call http_get(url, continuation=State1)
State 1: receive response, call parse_response(response, continuation=State2)
State 2: receive data, return data
```

### 是否需要 CPS？

```ocaml
let need_cps expr =
  try
    need_cps_visitor#visit_expr () expr;
    false
  with Need_cps -> true

(* 检测：是否存在 async apply 或 get_current_continuation *)
```

### 状态机结构

```ocaml
type continuation =
  | Identity                          (* 无continuation *)
  | Return of { cont; cont_ty }       (* 返回 *)
  | Simple of { state_id }            (* 跳转到指定状态 *)
  | Complex of (Core.expr -> Core.expr)  (* 复杂包装 *)

type state = {
  state_id : int;
  state_name : string;
  params : Core.param list;
  captures : (Ident.t * Stype.t) list;
  body : Core.expr;
}
```

### 变换过程

1. **切片**：按 `await` 点将 async 函数体切分为多个代码块
2. **状态分配**：每个代码块成为一个状态
3. **Continuation 传递**：
   - 每个 async 调用被替换为带 continuation 参数的调用
   - continuation 参数 = 下一个状态的 join 函数
4. **驱动循环**：生成一个 driver 函数，使用 loop 在不同状态间跳转

```ocaml
type ctx = {
  join_to_state : int Ident.Hash.t;     (* continuation → 状态映射 *)
  loop_info : loop_info Label.Hash.t;   (* 循环信息 *)
  return_cont : continuation;            (* 返回 continuation *)
  return_err_cont : continuation;        (* 错误返回 continuation *)
  driver_id : Ident.t;                   (* driver 函数标识符 *)
  driver_ty : Stype.t;
  states : state Vec.t;                 (* 所有状态 *)
  ...
}
```

### 生成的状态机

```
async fn f(x):
  let a = await g(x)     ← state 0
  let b = await h(a)     ← state 1
  return b               ← state 2

→

fn f(x):   (* 现在是一个普通同步函数 *)
  let state = 0
  let captures = {x, g_result, h_result} in
  loop(state, captures):
    if state == 0:
      g(x, continuation=state1_join)   (* 异步调用 *)
      break state1_join(...)
    elif state == 1:
      let a = g_result
      h(a, continuation=state2_join)
      break state2_join(...)
    elif state == 2:
      return h_result
```

---

## 5. 运行时支持

Async 变换后生成的代码依赖运行时提供：

- **调度器**：管理异步任务的调度（由 `moonbitlang/async` 包提供）
- **Continuation 调用**：`get_current_continuation` 基元操作
- **堆栈管理**：Wasm GC 的栈管理

运行时 API（core库）：
```mbt
// moonbitlang/async 包
pub fn sleep(ms : Int) -> Unit        // 异步sleep
pub fn with_task_group(f) -> Unit     // 结构化并发
task.spawn(fn)                        // 创建任务
task.spawn_bg(fn)                     // 后台任务
```

---

## 6. 完整错误处理编译示例

```mbt
suberror DivError { DivByZero }

fn div(a : Int, b : Int) -> Int!DivError {
  if b == 0 { raise DivError::DivByZero }
  a / b
}

fn compute(x : Int) -> Int!DivError {
  let a = div(x, 2)      // 错误自动传播
  let b = div(a, 0)      // 这里会出错
  b
}
```

### Core IR（compute 函数）：

```
Ctop_fn compute(x):
  let a =
    Cexpr_handle_error {
      obj = div(x, 2);
      handle_kind = Joinapply error_join  (* 错误传播 *)
    }
  in
  let b =
    Cexpr_handle_error {
      obj = div(a, 0);
      handle_kind = Joinapply error_join  (* 错误传播 *)
    }
  in
  Cexpr_return {
    expr = b;
    return_kind = Error_result { is_error = false; return_ty = T_int }
  }
```

### 运行时表示

错误值在运行时表示为带 tag 的堆分配结构体：
```
DivError::DivByZero  →  Wasm: (struct (field i32 0))  (* tag=0 *)
ParseError::InvalidEof  →  Wasm: (struct (field i32 0))  (* tag=0 *)
ParseError::InvalidChar('x')  →  Wasm: (struct (field i32 1) (field i32 120))
```

多个错误类型可以通过 `anyref` 统一处理，因为所有 Error 子类型都是 `Error` 的子类型。
