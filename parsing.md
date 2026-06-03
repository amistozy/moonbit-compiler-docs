# 语法分析与 AST 详解

MoonBit 的语法分析采用**手写递归下降**解析器（非解析器生成器），分布在 `parsing_*.ml` 模块中。

---

## 解析流水线

```
源文件内容
    │
    ▼
parsing_segment.ml        ← 将输入分段（控制分隔符）
    │
    ▼
parsing_parse.ml          ← 解析入口，协调tokenizer与parser
    │
    ├── lex_*.ml          ← 词法分析（tokenize）
    │
    ▼
parsing_main.ml           ← 递归下降解析器核心
    │
    ▼
parsing_syntax.ml         ← AST 定义
    │
    ▼
parsing_ast_lint.ml       ← AST后处理（lint/规范化）
    │
    ▼
parsing_compact.ml        ← AST压缩/simplify
    │
    ▼
Parse.output              ← 最终语法分析产物
```

---

## 1. 词法分析 (`lex_*.ml`)

### Token 定义 (`lex_menhir_token.ml`)

Token 类型面向 Menhir 解析器生成器风格，虽然解析器本身是手写的：

```ocaml
(* Token 种类（部分） *)
type terminal =
  | T_LIDENT of string        (* 小写标识符 *)
  | T_UIDENT of string        (* 大写标识符 *)
  | T_PACKAGE_NAME of string  (* 包名 *)
  | T_DOT_LIDENT of string    (* .小写标识符 *)
  | T_DOT_UIDENT of string    (* .大写标识符 *)
  | T_INT of string           (* 整数字面量 *)
  | T_FLOAT of string         (* 浮点字面量 *)
  | T_CHAR of Uchar.t         (* 字符 *)
  | T_STRING of string        (* 字符串 *)
  | T_MULTILINE_STRING of string  (* 多行字符串 *)
  | T_BYTE of string          (* 字节 *)
  | T_BYTES of string         (* 字节串 *)
  | T_INTERP of expr * segment (* 字符串插值 *)
  | T_POST_LABEL of string    (* 后缀标签 ~label *)
  | T_ATTRIBUTE of string     (* 属性 #[...] *)
  | T_TRUE | T_FALSE | T_UNDERSCORE
  (* 关键字 *)
  | T_FN | T_ASYNC | T_LET | T_CONST | T_TYPE | T_TYPEALIAS
  | T_STRUCT | T_ENUM | T_TRAIT | T_TRAITALIAS | T_IMPL
  | T_EXTERN | T_PUB | T_PRIV | T_READONLY | T_MUTABLE
  | T_IF | T_ELSE | T_MATCH | T_LOOP | T_WHILE | T_FOR
  | T_BREAK | T_CONTINUE | T_RETURN | T_GUARD | T_TRY
  | T_DERIVE | T_TEST
  | T_AS | T_IS | T_FOR | T_WITH
  (* 操作符 *)
  | T_PLUS | T_MINUS | T_STAR | T_SLASH | T_PERCENT
  | T_EQUAL | T_THIN_ARROW | T_FAT_ARROW
  | T_DOT | T_COMMA | T_COLON | T_COLONCOLON | T_SEMI
  | T_LPAREN | T_RPAREN | T_LBRACKET | T_RBRACKET
  | T_LBRACE | T_RBRACE | T_BAR | T_AMPER
  | T_DOTDOT | T_RANGE_INCLUSIVE | T_RANGE_EXCLUSIVE
  | T_QUESTION | T_EXCLAMATION
  | T_VBARVBAR | T_AMPAMP (* || && *)
  | ...
```

### 分号插入 (`lex_semi_insertion.ml`)

MoonBit 实现了类似 Go 的自动分号插入：在特定 Token 前自动插入分号。判断逻辑基于前一个 Token 的类型和位置（行号变化）。

### 词法分析器

- **`lex_literal.ml`** — 处理数值、字符串、字符字面量
- **`lex_comment.ml`** — 注释和文档注释（`///`）处理
- **`lex_unicode.ml`** / `lex_unicode_lex.ml` — Unicode 标识符支持
- **`lex_moon_rt.ml`** — MoonBit 运行时特定词法（内建函数名识别）
- **`lexing.ml`** — 词法分析驱动

---

## 2. AST 定义 (`parsing_syntax.ml`, 12,240行)

### 顶层结构

```ocaml
type impl =
  | Ptop_typedef of typedef        (* type/struct/enum/typealias *)
  | Ptop_funcdef of funcdef        (* fn/async fn *)
  | Ptop_letdef of letdef          (* let/const 绑定 *)
  | Ptop_trait of trait_decl       (* trait 声明 *)
  | Ptop_trait_alias of trait_alias_decl  (* traitalias *)
  | Ptop_impl of impl_decl         (* impl 块 *)
  | Ptop_test of test_block        (* test 块 *)
  | Ptop_expr of expr_block        (* 顶层表达式（含main） *)
```

### 表达式 AST（核心结构）

```ocaml
type expr =
  (* 字面量 *)
  | Pexpr_constant of constant
  | Pexpr_unit
  | Pexpr_interp of (string * expr) list * string

  (* 标识符 *)
  | Pexpr_ident of var

  (* 函数 *)
  | Pexpr_fn of { is_async; params; body; return_type }
  | Pexpr_apply of { func; args }             (* f(args) *)
  | Pexpr_method of { obj; method; args }     (* obj.method(args) *)

  (* 构造 *)
  | Pexpr_constr of constructor * expr option (* Constr(args) *)
  | Pexpr_tuple of expr list
  | Pexpr_record of field_def list            (* { a: 1, b: 2 } *)
  | Pexpr_record_update of { record; fields }
  | Pexpr_array of expr list                   (* [1, 2, 3] *)
  | Pexpr_map of (constant * expr) list        (* {1: "a", 2: "b"} *)

  (* 字段访问 *)
  | Pexpr_field of { record; accessor }      (* record.field *)
  | Pexpr_mutate of { record; label; field } (* record.field = val *)

  (* 控制流 *)
  | Pexpr_block of expr list
  | Pexpr_if of { cond; ifso; ifnot }
  | Pexpr_match of { obj; arms : match_arm list }
  | Pexpr_loop of { loop_body; loop_args }
  | Pexpr_for of { binder; expr; body }
  | Pexpr_while of { cond; body }
  | Pexpr_break of expr option
  | Pexpr_continue of expr option
  | Pexpr_return of expr option

  (* 操作符 *)
  | Pexpr_binary of { lhs; op; rhs }
  | Pexpr_unary of { op; expr }
  | Pexpr_pipe of { lhs; pipe; rhs }          (* lhs |> func *)

  (* 类型 *)
  | Pexpr_as of { expr; typ }                  (* expr as Type *)
  | Pexpr_annot of { expr; typ }               (* (expr : Type) *)
  | Pexpr_hole                                 (* _ 洞 *)

  (* let绑定 *)
  | Pexpr_let of { binder; ty; expr; body }
  | Pexpr_letmut of { binder; ty; expr; body }

  (* 错误处理 *)
  | Pexpr_try of { expr; catch }               (* try { ... } catch *)
  | Pexpr_guard of { expr; handler }           (* expr! 快捷错误处理 *)
  | Pexpr_label of { label; expr }             (* 带标签表达式 *)
```

### 模式

```ocaml
type pattern =
  | Ppat_constant of constant              (* 字面量模式 *)
  | Ppat_var of binder                     (* 变量绑定 *)
  | Ppat_any                               (* _ *)
  | Ppat_constr of { constr; args; is_open } (* 构造器模式 *)
  | Ppat_tuple of pattern list             (* (a, b, c) *)
  | Ppat_record of { fields; is_closed }   (* { x, y } *)
  | Ppat_map of { elems; is_closed }       (* {1: a, 2: b} *)
  | Ppat_array of array_patterns           (* [a, b, ...rest] *)
  | Ppat_or of { pat1; pat2 }              (* a | b *)
  | Ppat_range of { lhs; rhs; inclusive }  (* 1..10 / 1..=10 *)
  | Ppat_alias of { pat; alias }           (* pat as name *)
  | Ppat_constraint of { pat; ty }         (* (pat : Type) *)
```

### 模式匹配臂

```ocaml
type match_arm = {
  arm_pat : pattern;
  arm_guard : expr option;  (* if guard *)
  arm_body : expr;
}
```

### 类型表达式

```ocaml
type type_expr =
  | Ptype_arrow of { params : type_expr list; ret : type_expr; err : type_expr option }
  | Ptype_constr of type_name
  | Ptype_tuple of type_expr list
  | Ptype_fn of { params : type_expr list; ret : type_expr }
  | Ptype_any                               (* _ *)
  | Ptype_hole                              (* _ *)
```

### 顶层声明

**类型声明**：
```ocaml
type typedef = {
  tycon : string;            (* 类型名 *)
  params : type_decl_binder list;  (* 类型参数 *)
  components : type_desc;    (* 类型体 *)
  type_vis : visibility;
  deriving_ : deriving_directive list;
  ...
}

type type_desc =
  | Ptd_record of field_decl list      (* struct *)
  | Ptd_variant of constr_decl list     (* enum *)
  | Ptd_newtype of type_expr            (* type alias *)
  | Ptd_alias of type_expr              (* typealias *)
  | Ptd_abstract                        (* abstract type *)
  | Ptd_extern                          (* extern type *)
  | Ptd_error of exception_decl         (* type! 错误类型 *)
```

**函数声明**：
```ocaml
type funcdef = {
  fun_decl : fun_decl;
  decl_body : decl_body;
  ...
}

type decl_body =
  | Decl_body of { expr; local_types }
  | Decl_stubs of stub_kind   (* extern "c" fn ... = "name" *)

type stub_kind =
  | Embedded of { language; code }    (* 嵌入式代码 *)
  | Import of { module_name; func_name }  (* 导入 *)
```

**Trait声明**：
```ocaml
type trait_decl = {
  trait_name : binder;
  trait_supers : tvar_constraint list;    (* 父trait *)
  trait_methods : trait_method_decl list; (* 方法签名 *)
  ...
}
```

**Impl声明**：
```ocaml
type impl_decl = {
  self_ty : type_expr option;  (* impl Trait for Type *)
  trait : type_name;            (* trait名 *)
  method_name : binder;         (* 方法名 *)
  params : param list;          (* 方法参数 *)
  ret_ty : type_expr option;    (* 返回类型 *)
  body : decl_body;             (* 方法体 *)
  ...
}
```

---

## 3. 解析器架构 (`parsing_main.ml`)

### 核心设计

解析器是纯手写的递归下降解析器，使用 `parsing_core.ml` 提供的基础设施。

### 解析状态

```ocaml
type parse_state  (* parsing_core.ml *)
(* 内部包含：
   - Token流 + 位置信息
   - 错误收集器
   - 模式（Normal | Panic）- 错误恢复
   - 同步点栈（用于错误恢复）
*)
```

### 错误恢复

MoonBit 解析器实现了优雅的错误恢复：

```ocaml
type parse_mode = Normal | Panic of { loc : Rloc.t; ... }

(* push_sync/pop_sync：设置同步点，在panic时跳转到此 *)
(* with_sync：在同步点保护下解析 *)
(* add_error_skipped：记录被跳过的构造 *)
```

### 核心解析函数（部分）

```ocaml
val parse_toplevel : parse_state -> impl list
val parse_top : parse_state -> impl              (* 单个顶层声明 *)
val parse_expr : parse_state -> expr              (* 表达式 *)
val parse_pattern : parse_state -> pattern         (* 模式 *)
val parse_type : parse_state -> type_expr          (* 类型表达式 *)
val parse_fun_decl : attrs -> parse_state -> fun_decl
val parse_trait_decl : attrs -> vis -> parse_state -> trait_decl
val parse_field_decl : parse_state -> field_decl
val parse_constr_decl : parse_state -> constr_decl
val parse_block_expr_with_local_types : parse_state -> local_type_decl list * expr
```

### First集建模

解析器使用显式的 `first_*` 列表进行前瞻：

```ocaml
val first_impl : token_kind list
    (* 可以开始一个顶层声明的token集合 *)

val first_expr : token_kind list
val first_simple_expr : token_kind list
val first_type : token_kind list
val first_qual_lident : token_kind list
val first_constr : token_kind list
val first_map_pattern_key : token_kind list
```

这些用于错误恢复中的 `add_error_unexpected` 来生成有意义的错误信息。

---

## 4. 运算符处理 (`parsing_operators.ml`)

MoonBit 支持自定义运算符（以字母开头的）和内置运算符。运算符解析涉及：

- **优先级表**：定义了内置运算符的优先级和结合性
- **自定义运算符**：通过函数名模式匹配识别（`op_*` 命名约定）
- **管道运算符**：`|>` 链式调用

```ocaml
(* 运算符优先级组（从低到高） *)
(* PIPE   : |> *)
(* ASSIGN : = *)
(* OR     : || *)
(* AND    : && *)
(* COMPARE: == != < > <= >= *)
(* BITOR  : | *)
(* BITXOR : ^ *)
(* BITAND : & *)
(* SHIFT  : << >> *)
(* ADD    : + - *)
(* MUL    : * / % *)
(* UNARY  : - ! (前缀) *)
```

---

## 5. 其他解析模块

| 模块 | 功能 |
|------|------|
| `parsing_ast_lint.ml` | AST后处理：验证、规范化、lint检查 |
| `parsing_compact.ml` | AST简化：折叠冗余节点 |
| `parsing_import_path.ml` | 解析导入路径字符串 |
| `parsing_header_parser.ml` | 文件头解析 |
| `parsing_partial_info.ml` | 部分解析信息（为IDE/LSP提供） |
| `parsing_interp.ml` | 字符串插值 `\(expr)` 解析 |
| `parsing_syntax_util.ml` | AST工具函数 |
| `parsing_util.ml` | 解析器工具（`make_attribute`等） |

---

## 6. 源位置追踪

MoonBit 编译器有两层位置系统：

| 模块 | 类型 | 用途 |
|------|------|------|
| `rloc.ml` | `Rloc.t` | 相对位置（解析阶段，文件内部偏移） |
| `loc.ml` | `Loc.t` | 绝对位置（包含文件名，解析后期使用） |

`get_absolute_loc` 函数将 `Rloc.t` 转换为 `Loc.t`。

---

## 7. Parse.output（解析产物）

```ocaml
type output = {
  name : string;         (* 文件名 *)
  ast : Syntax.impl list; (* 顶层声明列表 *)
  comments : ...;        (* 注释 *)
  pkg : ...;             (* 包信息 *)
}
```

该产物直接传递给类型检查器 `typer.ml`。
