# 属性/注解系统

MoonBit 的属性系统（Attribute System）提供一种在声明上附加元数据的机制，类似 Rust 的 `#[attr]` 或 Java 的 `@Annotation`。属性在词法分析阶段被识别为 `ATTRIBUTE` Token，由 Menhir 生成的微型 LALR 解析器解析，在类型检查阶段进行语义验证。

---

## 1. 模块架构

```
lex_menhir_token.ml       → ATTRIBUTE token生成
        │
        ▼
attribute.ml              → 属性AST定义（attr_expr, attr_prop）
attribute_parser.ml       → Menhir LALR解析器（Attribute::attr_expr）
        │
        ▼
checked_attributes.ml     → 语义验证：Tattr_alert, Tattr_intrinsic
```

---

## 2. 属性AST定义：`attribute.ml`

### 2.1 属性标识符

```ocaml
type attr_id = { qual : string option; name : string }
```

支持两种形式：
- 简单名：`@alert` → `{ qual = None; name = "alert" }`
- 限定名：`@pkg.alert` → `{ qual = Some "pkg"; name = "alert" }`

### 2.2 属性表达式

```ocaml
type attr_expr =
  | Ident of attr_id                        (* @attr *)
  | String of Lex_literal.string_literal    (* "string" *)
  | Apply of attr_id * attr_prop list       (* @attr(props...) *)

and attr_prop =
  | Labeled of string * attr_expr           (* label = value *)
  | Expr of attr_expr                       (* positional argument *)
```

### 2.3 语法示例

```
@intrinsic("arithmetic_shift_right")     → Apply({name="intrinsic"}, [Expr(String ...)])
@alert(deprecated, "use newFn instead")  → Apply({name="alert"}, [Expr(Ident{deprecated}), Expr(String ...)])
@deprecated("use newFn instead")         → Apply({name="deprecated"}, [Expr(String ...)])
@pkg.custom_attr(flag=true, name="x")    → Apply({qual=Some"pkg";name="custom_attr"}, [Labeled("flag", Ident{true}), Labeled("name", String...)])
```

### 2.4 原始属性类型

词法分析器产出的原始属性：

```ocaml
type t = {
  loc_ : Rloc.t;           (* 源码位置 *)
  raw : string;            (* 原始文本 *)
  parsed : attr_expr option;  (* 解析后的AST（None=解析失败） *)
}
```

`parsed`为`None`表示属性文本无法解析为`attr_expr`，这种情况下属性被静默忽略。

---

## 3. 属性解析器：`attribute_parser.ml`

`attribute_parser.ml` 是一个由 **Menhir** 生成的 LALR(1) 解析器，600+行的自动生成代码。它实现了如下语法：

```
attr_expr ::= LIDENT                          → Ident(qual=None, name)
            | LIDENT DOT_LIDENT               → Ident(qual=Some LIDENT, name=DOT_LIDENT)
            | STRING                           → String(string_literal)
            | LIDENT LPAREN properties RPAREN → Apply(name, props)
            | LIDENT DOT_LIDENT LPAREN properties RPAREN → Apply(qualified_name, props)

properties ::= ε                      → []
             | non_empty_properties   → ...

non_empty_properties ::= property                      → [prop]
                       | property COMMA non_empty_properties → prop::props

property ::= expr                                    → Expr(expr)
           | LIDENT EQUAL expr                       → Labeled(name, expr)
```

**关键设计**：这是一个专用于属性的微型语法，与MoonBit主语法完全独立。属性解析在token级别完成，不依赖主解析器。

```ocaml
let attribute _menhir_lexer _menhir_lexbuf =
  (* 入口函数，调用Menhir生成的LALR状态机 *)
  let (MenhirBox_attribute v) = _menhir_run_00 ... in
  v
```

---

## 4. 语义验证：`checked_attributes.ml`

类型检查阶段调用 `Checked_attributes.check` 对属性进行语义验证：

### 4.1 支持的内建属性

```ocaml
type attribute =
  | Tattr_alert of { loc_ : location; category : string; message : string }
  | Tattr_intrinsic of { loc_ : location; intrinsic : string }

type t = attribute list  (* 函数可携带多个属性 *)
```

### 4.2 验证逻辑

```ocaml
let check ~local_diagnostics ~context attrs =
  Lst.fold_left attrs [] (fun acc attr ->
    match attr with
    | { parsed = Some expr; loc_; _ } ->
      match expr with
      | Apply({name="intrinsic"}, Expr(String{string_val})::[]) ->
          (* 仅允许在顶层函数上 *)
          if context = `TopFun then Tattr_intrinsic :: acc
          else warn "unused attribute intrinsic"; acc

      | Apply({name="deprecated"}, Expr(String message)::[]) ->
          (* 仅允许在顶层函数上 *)
          if context = `TopFun then Tattr_alert { category="deprecated"; message } :: acc
          else warn "unused attribute deprecated"; acc

      | Apply({name="alert"}, [Expr(Ident{name=category}); Expr(String message)]) ->
          (* 仅允许在顶层函数上 *)
          if context = `TopFun then Tattr_alert { category; message } :: acc
          else warn "unused attribute alert"; acc

      | _ -> acc  (* 未识别的属性→忽略 *)
    | _ -> acc)
```

### 4.3 上下文限制

| 上下文 | 允许的属性 |
|---|---|
| `TopLet` | 无（警告unused） |
| `TopFun` | `@intrinsic`、`@deprecated`、`@alert` |
| `TopTypeDecl` | 无 |
| `Impl` | 无 |
| `Trait` | 无 |

当前设计下，**属性仅对顶层函数有意义**。

---

## 5. 属性类型详解

### 5.1 `@intrinsic`—内建函数标记

```mbt
///|
/// 标记函数为编译器内建实现
/// @intrinsic("arithmetic_shift_right")
pub fn Int::op_shr(self : Int, n : Int) -> Int = "%int.shr"
```

- 告诉编译器该函数有专门的内部实现
- 在`make_apply`中，内建函数会尝试`Core_util.try_apply_intrinsic`进行编译期优化
- `intrinsic`字符串对应`Primitive.prim`中的变体

### 5.2 `@deprecated`—废弃标记

```mbt
///|
/// @deprecated("Use newFunction instead")
pub fn oldFunction() -> Unit { ... }
```

- 编译为`Tattr_alert { category = "deprecated"; message = "..." }`
- 在调用点，通过`check_alerts`生成诊断告警

### 5.3 `@alert`—自定义告警

```mbt
///|
/// @alert(unsafe, "This function performs unchecked array access")
pub fn unsafe_get(arr : Array[Int], i : Int) -> Int { ... }
```

- `category`：告警类别名
- `message`：告警消息
- 通用告警框架：`@alert(category, message)`在调用点生成自定义告警

---

## 6. 告警触发流程

```
源码中的函数调用
    │
    ▼
类型检查器 → 查找被调用函数的checked_attributes
    │
    ▼
Checked_attributes.check_alerts
    │
    ├── Tattr_alert { category = "deprecated" } → Local_diagnostics.add_alert
    │   生成诊断："Use of deprecated function: <message>"
    │
    └── Tattr_alert { category = custom } → Local_diagnostics.add_alert
        生成诊断："<category>: <message>"
```

---

## 7. 属性在编译器流水线中的流转

```
源文件 (@attr)
    │
    ▼
词法分析 → ATTRIBUTE token
    │
    ▼ (原始文本保存为Attribute.t)
语法分析 → 附加到声明节点（Syntax AST）
    │
    ▼ (调用attribute_parser.ml解析)
类型检查 → Checked_attributes.check
    │         ↓
    │    Checked_attributes.t = [Tattr_alert | Tattr_intrinsic]
    │
    ▼
Typedtree → 附加到Typedtree.fun_decl.attrs
    │
    ▼
Core IR → attrs信息包含在Ctop_fn中
    │
    ▼
链接 → 告警信息传递给调用点
```

---

## 8. 扩展性

属性系统的设计预留了扩展空间：

1. **自定义属性**：`attr_id.qual`支持`@pkg.custom_attr`形式的限定名
2. **未识别属性静默忽略**：`check`中对未匹配的属性返回`acc`（跳过），不报错
3. **Menhir语法可扩展**：添加新的属性语法只需修改Menhir语法文件并重新生成`attribute_parser.ml`
4. **上下文扩展**：`check`函数的`context`参数可扩展到`TopLet`、`TopTypeDecl`等更多场景

---

## 与 `diagnostics.md` 的关系

`diagnostics.md` 涵盖警告码（1-45）和错误报告系统。本文档聚焦于属性的**解析、验证和应用**，特别是 `@deprecated`/`@alert` 如何触发告警诊断。两者在告警触发点交叉。
