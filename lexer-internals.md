# 词法分析器内部实现

MoonBit的词法分析器（Lexer）由多个模块协作完成，将源文件的Unicode码点流转换为Token流。与传统的lex/yacc方案不同，MoonBit采用**手写词法分析器**，通过专门的Unicode感知的lexbuf直接操作整数码点数组。

---

## 1. 模块架构

```
源文件 (UTF-8编码)
    │
    ▼
lex_moon_rt.ml          ← 底层词法运行时：快速整数码点缓冲区
    │
    ▼
lex_unicode.ml          ← Unicode码点合法性校验（32个区间二分查找）
lex_keyword_tbl.ml      ← 关键字/保留字查找表
    │
    ▼
lex_literal.ml          ← 字面量类型定义（char/string/byte/bytes/interp）
lex_comment.ml          ← 注释Token类型
    │
    ▼
lex_menhir_token.ml     ← Token类型定义（~100个变体）
    │
    ▼
lex_vec_token.ml        ← Token缓冲（Vec.t包装）
lex_vec_comment.ml      ← 注释Token缓冲
    │
    ▼
lex_semi_insertion.ml   ← 自动分号插入（ASI）
    │
    ▼
parsing_parse.ml        ← 解析器入口，调用lexer tokenize
```

---

## 2. 底层词法运行时：`lex_moon_rt.ml`

### 2.1 自定义Lexbuf

MoonBit不直接使用OCaml标准库的`Lexing.lexbuf`（字符串/字节流），而是定义了自己的整数数组lexbuf：

```ocaml
type lexbuf = {
  buf : int array;        (* Unicode码点数组 *)
  len : int;              (* 缓冲区长度 *)
  mutable pos : int;      (* 当前位置 *)
  mutable start_pos : int;  (* Token起始位置 *)
  mutable marked_pos : int; (* 回溯标记位置 *)
  mutable marked_val : int; (* 回溯标记值 *)
}
```

**设计理由**：MoonBit的源文件先被解码为Unicode码点数组，再进行词法分析。这避免了对UTF-8字节编码的复杂处理，使词法分析器直接操作32位Unicode码点。

### 2.2 基本操作

| 操作 | 说明 |
|---|---|
| `next_int` | 读取下一个码点，返回`-1`表示EOF |
| `peek_next_int` | 查看下一个码点（不前进） |
| `start` | 标记Token起始位置 |
| `mark lexbuf i` | 保存回溯位置和值 |
| `backtrack` | 回到标记位置，返回保存值 |
| `backoff lexbuf i` | 向后退i个位置 |
| `lexeme_start/lexeme_end` | 获取当前Token的起止位置 |
| `current_code_point` | 获取最后读取的码点 |

---

## 3. Unicode支持：`lex_unicode.ml`

### 3.1 合法标识符码点

MoonBit标识符支持广泛的Unicode字符，使用**区间二分查找**验证：

```ocaml
let blocks : int array = [|
  (* ASCII *)
  0x30; 0x39;   (* 0-9 *)
  0x41; 0x5a;   (* A-Z *)
  0x5f; 0x5f;   (* _ *)
  0x61; 0x7a;   (* a-z *)
  (* Latin Extended *)
  0xa1; 0xac;   (* ¡ - ¬ *)
  0xae; 0x2af;  (* ® - ʯ *)
  (* CJK *)
  0x1100; 0x11ff;   (* Hangul Jamo *)
  0x1e00; 0x1eff;   (* Latin Extended Additional *)
  0x2070; 0x209f;   (* Superscripts and Subscripts *)
  0x2150; 0x218f;   (* Number Forms *)
  0x2e80; 0x2eff;   (* CJK Radicals Supplement *)
  0x2ff0; 0x2fff;   (* Ideographic Description Characters *)
  0x3001; 0x30ff;   (* CJK Symbols and Punctuation, Hiragana, Katakana *)
  0x31c0; 0x9fff;   (* CJK Strokes ~ CJK Unified Ideographs *)
  0xac00; 0xd7ff;   (* Hangul Syllables *)
  0xf900; 0xfaff;   (* CJK Compatibility Ideographs *)
  0xfe00; 0xfe0f;   (* Variation Selectors *)
  0xfe30; 0xfe4f;   (* CJK Compatibility Forms *)
  (* SMP *)
  0x1f000; 0x1fbff;  (* Mahjong, Domino, Playing Cards, Emoticons *)
  0x20000; 0x2a6df;  (* CJK Extension B *)
  0x2a700; 0x2ebef;  (* CJK Extension C, D *)
  0x2f800; 0x2fa1f;  (* CJK Compatibility Ideographs Supplement *)
  0x30000; 0x323af;  (* CJK Extension E, F *)
  0xe0100; 0xe01ef;  (* Variation Selectors Supplement *)
|]

let is_valid_unicode_codepoint c =
  Basic_binary_search.search_range c blocks <> -1
```

共32个Unicode区间，覆盖：
- ASCII字母数字和下划线
- 拉丁扩展
- 中日韩统一表意文字（CJK）
- 谚文音节
- 表情符号与特殊符号

---

## 4. Token定义：`lex_menhir_token.ml`

MoonBit Token类型的命名风格受Menhir解析器生成器启发（尽管解析器本身是手写的）：

```ocaml
type token =
  (* 关键字 *)
  | AS | ELSE | EXTERN | FN | IF | LET | CONST | MATCH | MUTABLE
  | TYPE | TYPEALIAS | STRUCT | ENUM | TRAIT | TRAITALIAS | DERIVE
  | WHILE | BREAK | CONTINUE | IMPORT | RETURN | THROW | RAISE
  | TRY | CATCH | PUB | PRIV | READONLY | TRUE | FALSE
  | TEST | LOOP | FOR | IN | IMPL | WITH | GUARD | ASYNC | IS

  (* 字面量 *)
  | INT of string            (* 数字字面量（字符串，含进制信息） *)
  | FLOAT of string          (* 浮点字面量 *)
  | CHAR of char_literal     (* 字符字面量 *)
  | STRING of string_literal (* 字符串字面量 *)
  | MULTILINE_STRING of string     (* 多行字符串 #| ... |# *)
  | MULTILINE_INTERP of interp_literal (* 多行插值字符串 $| ... |$ *)
  | INTERP of interp_literal  (* 插值字符串 "..." *)
  | BYTE of byte_literal     (* 字节字面量 b'X' *)
  | BYTES of bytes_literal   (* 字节数组字面量 b"..." *)

  (* 标识符 *)
  | UIDENT of string         (* 大写标识符（类型/构造器名） *)
  | LIDENT of string         (* 小写标识符（变量/函数名） *)
  | PACKAGE_NAME of string   (* 包名 @package *)
  | DOT_UIDENT of string     (* .大写标识符 *)
  | DOT_LIDENT of string     (* .小写标识符 *)
  | DOT_INT of int           (* .整数 *)
  | POST_LABEL of string     (* ~标签（labeled argument） *)
  | UNDERSCORE               (* _ *)

  (* 符号 *)
  | LPAREN | RPAREN | LBRACKET | RBRACKET | LBRACE | RBRACE
  | COLON | COLONCOLON | COMMA | SEMI of bool (* true=显式分号 *)
  | FAT_ARROW | THIN_ARROW | EQUAL | PIPE | DOTDOT | ELLIPSIS
  | PLUS | MINUS | CARET | AMPER | AMPERAMPER | BAR | BARBAR
  | QUESTION | EXCLAMATION | RANGE_INCLUSIVE | RANGE_EXCLUSIVE
  | INFIX1 of string  (* 自定义中缀运算符优先级1 *)
  | INFIX2 of string  (* 优先级2 *)
  | INFIX3 of string  (* 优先级3 *)
  | INFIX4 of string  (* 优先级4 *)
  | AUGMENTED_ASSIGNMENT of string  (* +=, -=, *=等等 *)

  (* 其他 *)
  | NEWLINE | EOF
  | COMMENT of Comment.t
  | ATTRIBUTE of string     (* 属性/注解 *)
```

### 4.1 `SEMI` Token

`SEMI of bool` 携带一个布尔值标记：
- `true`：显式分号（源码中写了`;`）
- `false`：自动插入的分号（ASI）

---

## 5. 关键字与保留字：`lex_keyword_tbl.ml`

### 5.1 关键字表

39个关键字，使用Hash表快速查找：

| 关键字 | Token | 关键字 | Token |
|---|---|---|---|
| `as` | AS | `else` | ELSE |
| `extern` | EXTERN | `fn` | FN |
| `if` | IF | `let` | LET |
| `const` | CONST | `match` | MATCH |
| `mut` | MUTABLE | `type` | TYPE |
| `typealias` | TYPEALIAS | `struct` | STRUCT |
| `enum` | ENUM | `trait` | TRAIT |
| `traitalias` | TRAITALIAS | `derive` | DERIVE |
| `while` | WHILE | `break` | BREAK |
| `continue` | CONTINUE | `import` | IMPORT |
| `return` | RETURN | `throw` | THROW |
| `raise` | RAISE | `try` | TRY |
| `catch` | CATCH | `pub` | PUB |
| `priv` | PRIV | `readonly` | READONLY |
| `true` | TRUE | `false` | FALSE |
| `_` | UNDERSCORE | `test` | TEST |
| `loop` | LOOP | `for` | FOR |
| `in` | IN | `impl` | IMPL |
| `with` | WITH | `guard` | GUARD |
| `async` | ASYNC | `is` | IS |

### 5.2 保留字

保留字是当前未使用但**不能**作为标识符的词：

```ocaml
let reserved = [|
  "module"; "move"; "ref"; "static"; "super"; "unsafe"; "use";
  "where"; "await"; "dyn"; "abstract"; "do"; "final"; "macro";
  "override"; "typeof"; "virtual"; "yield"; "local"; "method";
  "alias"; "assert";
|]
```

`is_reserved`检查确保这些词不会作为标识符使用。

---

## 6. 字面量类型：`lex_literal.ml`

### 6.1 Literal类型

```ocaml
type char_literal = { char_val : uchar; char_repr : string }
  (* char_val: Unicode码点值 *)
  (* char_repr: 源码表示形式（如"'a'"） *)

type string_literal = { string_val : string; string_repr : string }
  (* string_val: 解码后的实际字符串 *)
  (* string_repr: 源码表示形式（保留原始转义） *)

type byte_literal = { byte_val : int; byte_repr : string }
  (* byte_val: 0~255的整数值 *)
  (* byte_repr: 源码表示形式（如"b'X'"） *)

type bytes_literal = { bytes_val : string; bytes_repr : string }
  (* bytes_val: 字节序列（作为string存储） *)
  (* bytes_repr: 源码表示形式 *)
```

**设计要点**：每种字面量类型都同时保存`val`（语义值）和`repr`（源码表示），后者用于格式化工具精确还原原始写法。

### 6.2 插值字符串

```ocaml
type interp_literal = {
  segments : string list;    (* 字符串片段 *)
  exprs : int list;          (* 插值表达式位置 *)
}
```

MoonBit的`"hello \{name}"`格式的插值字符串不是由词法分析器处理的——词法分析器将其识别为`INTERP` token，由解析器进一步处理插值表达式。

---

## 7. 自动分号插入（ASI）：`lex_semi_insertion.ml`

### 7.1 ASI规则

MoonBit借鉴了JavaScript/Go的分号插入思想，但在编译时而非运行时：

```ocaml
let can_occur_before_semi token =
  (* 当前Token可以出现在分号之前（即一行末尾） *)
  match token with
  | UIDENT _ | LIDENT _ | ... | RBRACE | RPAREN | RBRACKET
  | UNDERSCORE | BREAK | CONTINUE | RETURN | THROW
  | QUESTION | EXCLAMATION | RANGE_INCLUSIVE | RANGE_EXCLUSIVE
  | PIPE | ELLIPSIS | POST_LABEL _ -> true
  | _ -> false

let can_occur_after_semi token =
  (* 下一个Token可以出现在分号之后（即新一行开头） *)
  match token with
  | UIDENT _ | LIDENT _ | ... | LBRACE | LPAREN | LBRACKET
  | TYPE | STRUCT | FN | IF | WHILE | FOR | ... -> true
  | _ -> false
```

插入分号的条件：前一个非注释Token**可以在行末**（`can_occur_before_semi`），**且**后一个Token**可以在行首**（`can_occur_after_semi`）。

### 7.2 特殊情况

```ocaml
(* 连续多行字符串不插入分号 *)
| (MULTILINE_STRING _ | MULTILINE_INTERP _),
  (MULTILINE_STRING _ | MULTILINE_INTERP _) -> ()
```

### 7.3 ASI上下文

```ocaml
type asi_context = { mutable last_unhandled_newline : int }
```

- 遇到`NEWLINE`时，记录位置
- 遇到下一个非注释、非换行Token时，检查是否需要插入
- 插入的`SEMI(false)`标记为faked_semi

---

## 8. 词法分析器入口：`parsing_parse.ml`

虽然lexer的核心模块分布在`lex_*.ml`中，词法分析的实际调用由`parsing_parse.ml`协调：

```
parsing_parse.ml (tokenize函数)
    │
    ├── 从源文件读取 → 编码为Unicode码点数组
    ├── 调用 lex_moon_rt 中的自定义lexer
    ├── 逐字符分类：
    │   ├── Unicode字母 → 标识符 → 查keyword_tbl
    │   ├── 数字 → 数字/浮点字面量
    │   ├── 引号 → 字符串/字符字面量
    │   ├── 符号 → 运算符/分隔符
    │   └── 空白/换行 → NEWLINE / 跳过空格
    ├── 处理注释（// 和 /* */）
    ├── 处理多行字符串 #|...|# 和 $|...|$
    ├── 处理属性注解 @attr
    └── 处理Unicode转义序列
```

---

## 9. Token缓冲与注释处理

### 9.1 Token缓冲

`lex_vec_token.ml` 和 `lex_vec_comment.ml`提供基于`Vec.t`的Token缓冲，支持：
- 追加Token
- 插入Token（ASI需要向后插入分号）
- 随机访问（ASI需要向前查看）

### 9.2 注释处理

注释Token（`COMMENT of Comment.t`）被收集但不出现在标准Token流中（`lex_semi_insertion.ml`直接跳过）：

```ocaml
| COMMENT _ -> ()  (* 注释不触发ASI *)
```

注释内容用于文档生成和工具支持。

---

## 10. MoonBit词法特性总结

| 特性 | 实现细节 |
|---|---|
| Unicode标识符 | 32个Unicode区间二分查找，支持CJK/谚文/emoji |
| 自定义中缀运算符 | `INFIX1`~`INFIX4`四级优先级 |
| 数字字面量重载 | 所有数字以字符串保留（类型推断时确定具体类型） |
| 多行字符串 | `#\|...\|#`（无转义）、`$\|...\|$`（支持插值） |
| 自动分号插入（ASI） | 编译时基于Token类型规则 |
| 字节字面量 | `b'X'`（单字节）、`b"..."`（字节序列） |
| 属性注解 | `@attr` 被识别为独立Token `ATTRIBUTE` |
| 插值字符串 | `"hello \{name}"` 识别为`INTERP` Token |

---

## 与 `parsing.md` 的关系

`parsing.md` 侧重于语法分析（AST构建），本文档深入词法分析器的**内部实现细节**，包括：词法运行时、Unicode码点处理、Token定义、ASI算法和字面量类型。两者互为补充。
