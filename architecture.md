# MoonBit 编译器架构分析

## 概览

MoonBit 编译器（`moonc`）是一个用 **OCaml** 编写、基于 **dune** 构建系统的多遍（multi-pass）编译器。源代码约 250+ 个 `.ml` 文件集中在单一 `src/` 目录下，采用经典的编译流水线架构：**词法分析 → 语法分析 → 类型检查 → Core IR → 单态化 → Clam IR → Wasm GC 代码生成**。

---

## 整体流水线

```
源文件 (.mbt)
    │
    ▼
┌──────────────────────────────────────────────────┐
│ 1. 词法分析 (Lexing)                              │
│    lex_menhir_token, lex_literal, lex_comment ... │
└──────────────────────────────────────────────────┘
    │ Token Stream
    ▼
┌──────────────────────────────────────────────────┐
│ 2. 语法分析 (Parsing)                             │
│    parsing_parse → parsing_main → parsing_syntax  │
│    产出: Syntax AST (未类型化的语法树)              │
└──────────────────────────────────────────────────┘
    │ Parse.output (Syntax AST)
    ▼
┌──────────────────────────────────────────────────┐
│ 3. 类型检查 (Type Checking)                       │
│    typer.ml, pattern_typer.ml, toplevel_typer.ml │
│    产出: Typedtree.output (类型化AST)              │
└──────────────────────────────────────────────────┘
    │ Typedtree.output + Global_env.t
    ▼
┌──────────────────────────────────────────────────┐
│ 4. Core IR 转换 (core_of_tast.ml)                 │
│    将类型化AST翻译为Core中间表示                    │
└──────────────────────────────────────────────────┘
    │ Core.program
    ▼
┌──────────────────────────────────────────────────┐
│ 5. Core IR 优化流水线 (多遍Pass)                   │
│    InlineSingleUseJoin → EliminateAsync →         │
│    Contification → RemoveLetAlias → Stackalloc →  │
│    UnboxLoopParams → PropagateConstr →            │
│    LambdaLift → DCE                               │
└──────────────────────────────────────────────────┘
    │ Core.program (优化后)
    ▼
┌──────────────────────────────────────────────────┐
│ 6. 链接 (core_link.ml)                            │
│    链接多个包/模块的Core IR                       │
└──────────────────────────────────────────────────┘
    │ Core_link.output
    ▼
┌──────────────────────────────────────────────────┐
│ 7. 单态化 (Monofy.ml)                             │
│    泛型/多态代码 → 单态代码                        │
│    产出: Mcore.t                                  │
└──────────────────────────────────────────────────┘
    │ Mcore.t
    ▼
┌──────────────────────────────────────────────────┐
│ 8. Clam IR (clam_of_core.ml)                      │
│    转换为更接近Wasm GC的低级中间表示               │
│    优化: UnusedLet                                 │
│    产出: Clam.prog                                │
└──────────────────────────────────────────────────┘
    │ Clam.prog
    ▼
┌──────────────────────────────────────────────────┐
│ 9. Wasm GC 代码生成 (wasm_of_clam_gc.ml)          │
│    Clam → Dwarfsm AST → 二进制Wasm                │
│    优化: ShrinkWasmir, ElimEquivDefn              │
└──────────────────────────────────────────────────┘
    │ .wasm (Wasm GC二进制)
```

---

## 各层详细分析

### 1. 词法分析 (`lex_*.ml`)

| 文件 | 职责 |
|------|------|
| `lex_menhir_token.ml` | 定义 Token 类型（为 Menhir 解析器生成器设计） |
| `lex_literal.ml` | 数字、字符串、字符等字面量词法 |
| `lex_comment.ml` | 注释处理（含文档注释） |
| `lex_unicode.ml` / `lex_unicode_lex.ml` | Unicode 支持 |
| `lex_semi_insertion.ml` | 自动分号插入（类似 Go 的规则） |
| `lex_moon_rt.ml` | MoonBit 运行时特定词法 |
| `lex_keyword_tbl.ml` | 关键字表 |
| `lex_vec_comment.ml` / `lex_vec_token.ml` | Token/注释的Vec容器 |
| `lex_token_triple.ml` | Token三元组（用于分号插入） |
| `lex_menhir_token_util.ml` | Token工具函数 |
| `lexing.ml` | 词法分析驱动 |

### 2. 语法分析 (`parsing_*.ml`)

这是代码量最大的模块。核心文件：

| 文件 | 行数 | 职责 |
|------|------|------|
| `parsing_syntax.ml` | 12,240 | **语法AST定义**：表达式、模式、类型、顶层声明等所有语法结构 |
| `parsing_main.ml` | 8,000+ | **递归下降解析器核心**：`parse_top`, `parse_expr`, `parse_pattern` 等 |
| `parsing_parse.ml` | - | 解析入口 |
| `parsing_operators.ml` | - | 运算符优先级/结合性处理 |
| `parsing_compact.ml` | - | AST压缩/简化 |
| `parsing_segment.ml` | - | 输入分段 |
| `parsing_ast_lint.ml` | - | AST后处理/lint |
| `parsing_header_parser.ml` | - | 文件头解析 |
| `parsing_import_path.ml` | - | 导入路径解析 |
| `parsing_core.ml` | - | 解析器核心状态机 |
| `parsing_util.ml` | - | 解析器工具函数 |
| `parsing_menhir_state.ml` | - | Menhir 解析器状态 |
| `parsing_partial_info.ml` | - | 部分解析信息（IDE支持） |
| `parsing_syntax_util.ml` | - | 语法结构工具函数 |
| `parsing_interp.ml` | - | 字符串插值解析 |

MoonBit 语法支持的特性包括：
- **类型定义**：`type`、`struct`、`enum`、`typealias`、`traitalias`、错误类型 (`type!`)
- **函数**：`fn`、`async fn`、外部函数 (`extern`)、嵌入式代码
- **Trait/Impl**：`trait` 声明、`impl` 实现
- **测试**：`test` 块
- **常量/变量**：`let`、`const`
- **模式匹配**：构造器模式、记录模式、Map模式、数组模式（含 `..` spread）、`or` 模式、范围模式、别名模式

### 3. 类型系统 (`typer.ml`, `typedtree.ml`, `type*.ml`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `typer.ml` | 5,391 | 主类型检查器：包含双向类型检查、unification、类型推断 |
| `typedtree.ml` | 10,866 | 类型化AST定义（带类型的语法树） |
| `typedtree_util.ml` | - | 类型化AST工具函数 |
| `typedecl_info.ml` | - | 类型声明信息 |
| `mtype.ml` | - | MoonBit 类型表示（高级类型） |
| `ctype.ml` | - | 约束类型系统 |
| `ltype.ml` / `ltype_gc.ml` / `ltype_gc_util.ml` | - | 低级类型（Wasm GC 对应），用于后端 |
| `stype.ml` | - | 结构类型表示 |
| `stub_type.ml` | - | FFI/外部桩类型 |
| `pattern_typer.ml` | - | 模式类型检查 |
| `toplevel_typer.ml` | - | 顶层声明类型检查 |
| `type_constraint.ml` | - | 类型约束处理 |
| `constraint_cache.ml` | - | 约束缓存 |
| `type_subst.ml` | - | 类型替换 |
| `poly_type.ml` | - | 多态类型 |
| `type_alias.ml` | - | 类型别名 |
| `type_args.ml` | - | 类型参数 |
| `type_lint.ml` | - | 类型相关lint |
| `type_path_util.ml` | - | 类型路径工具 |
| `transl_type.ml` | - | 类型翻译（mtype→ltype） |
| `transl_mtype_gc.ml` | - | mtype到Wasm GC类型的翻译 |
| `tvar_env.ml` | - | 类型变量环境 |
| `typeutil.ml` | - | 类型工具函数 |
| `typing_info.ml` | - | 类型信息存储 |
| `typecheck_driver_util.ml` | - | 类型检查驱动 |

类型系统核心机制：
- **双向类型检查**（Bidirectional type checking）
- **Unification-based 类型推断**
- **Trait 约束**（类似 Rust trait，但使用 `:+` 语法表示约束）
- **结构化类型**（struct）与 **代数数据类型**（enum）

### 4. 中间表示 (IR)

MoonBit 编译器使用 **三层中间表示**：

#### Core IR (`core.ml` — 4,854行)

- 类型化后的**高级中间表示**
- 保留函数名、类型信息、构造器等
- 支持闭包、async、foreach 等高层结构
- 标识符系统：`Basic_core_ident`

相关文件：
| 文件 | 职责 |
|------|------|
| `core.ml` | Core IR 定义 |
| `core_of_tast.ml` | Typedtree → Core 转换 |
| `core_util.ml` | Core 工具函数 |
| `core_dce.ml` | Core 死代码消除 |
| `core_format.ml` | Core 序列化/反序列化 |
| `core_link.ml` | Core 多模块链接 |

#### Mcore IR (`mcore.ml` — 4,893行)

- **单态化后的 Core IR**
- 泛型代码已展开为具体类型的版本
- 标识符系统：`Basic_core_ident`（同 Core）
- 是 Core → Clam 之间的桥梁

相关文件：
| 文件 | 职责 |
|------|------|
| `mcore.ml` | Mcore IR 定义 |
| `mcore_util.ml` | Mcore 工具函数 |
| `pass_layout.ml` | 数据布局优化（在 Mcore 上运行） |

#### Clam IR (`clam.ml` — 3,332行)

- **低级中间表示**，紧密对应 Wasm GC 指令集
- 显式的控制流（block、loop、br、if）
- GC 引用类型（ref、null、array）
- 标识符系统：`Clam_ident`
- 类型系统：`Ltype_gc`（直接映射 Wasm GC 类型）

相关文件：
| 文件 | 职责 |
|------|------|
| `clam.ml` | Clam IR 定义 |
| `clam_of_core.ml` | Mcore → Clam 转换 |
| `clam_util.ml` | Clam 工具函数 |
| `clam_ident.ml` / `clam1_ident.ml` | Clam 标识符系统 |

```
MoonBit源 ─→ Typedtree ─→ Core ─→ Mcore ─→ Clam ─→ Dwarfsm(wasm) ─→ .wasm
                         泛型     单态化    低级IR    Wasm IR     二进制
```

### 5. 优化 Pass (`pass_*.ml`)

Core IR 上的优化流水线（按执行顺序）：

| 顺序 | Pass | 文件 | 功能 |
|------|------|------|------|
| 1 | InlineSingleUseJoin | `pass_inline_single_use_join.ml` | 内联单次使用的 join 点 |
| 2 | EliminateAsync | `eliminate_async.ml` | 将 async 转换为状态机 |
| 3 | Contification | `pass_contification.ml` | 将函数调用转换为 continuation |
| 4 | RemoveLetAlias | `pass_let_alias.ml` | 消除 let 别名 |
| 5 | Stackalloc | `pass_stackalloc.ml` | 栈分配优化（unbox可变记录） |
| 6 | UnboxLoopParams | `pass_unbox_loop_params.ml` | 循环参数 unboxing |
| 7 | PropagateConstr | `pass_propagate_constr.ml` | 构造器传播 |
| 8 | LambdaLift | `lambda_lift.ml` | Lambda 提升（闭包转换） |
| 9 | DCE | `core_dce.ml` | 死代码消除 |

Clam IR 上的优化：

| Pass | 文件 | 功能 |
|------|------|------|
| UnusedLet | `pass_unused_let.ml` | 消除未使用的绑定 |

Mcore IR 上的优化：

| Pass | 文件 | 功能 |
|------|------|------|
| Layout | `pass_layout.ml` | 数据布局优化 |

### 6. 单态化 (`monofy*.ml`)

| 文件 | 职责 |
|------|------|
| `monofy.ml` | 单态化主入口 |
| `monofy_env.ml` | 单态化环境 |
| `monofy_analyze.ml` | 分析哪些泛型实例需要生成 |
| `monofy_instances.ml` | 实例生成 |
| `monofy_worklist.ml` | 工作列表算法 |

单态化将多态函数按具体调用类型展开，生成特化版本。MoonBit 中没有运行时泛型/动态分发（除 trait 方法外）。

### 7. Wasm GC 后端

这是目前此仓库**唯一公开的后端**（LLVM 后端为 nightly 专用）：

```
Clam.prog
    │
    ▼
wasm_of_clam_gc.ml        ← Clam → Dwarfsm 转换
    │
    ▼
dwarfsm_ast.ml            ← Dwarfsm AST（Wasm的OCaml表示）
    │
    ├── dwarfsm_encode.ml          ← 二进制 Wasm 编码
    ├── dwarfsm_encode_wasm.ml     ← 文本格式 Wasm 编码
    ├── dwarfsm_elim_equivdefn.ml  ← 等价定义去重
    ├── dwarfsm_local_resolve.ml   ← 局部变量解析
    └── dwarfsm_encode_resolve.ml  ← 编码时符号解析
    │
    ▼
shrink_wasmir.ml           ← Wasm IR 瘦身优化
    │
    ▼
dwarfsm_encode.ml           ← 最终二进制输出
    │
    ▼
.wasm 文件
```

相关文件：
| 文件 | 职责 |
|------|------|
| `wasm_of_clam_gc.ml` | Clam → Dwarfsm 转换 |
| `dwarfsm_ast.ml` | Dwarfsm AST 定义 |
| `dwarfsm_basic.ml` | Dwarfsm 基础类型 |
| `dwarfsm_itype.ml` | Dwarfsm 指令类型 |
| `dwarfsm_instr_utils.ml` | 指令工具函数 |
| `dwarfsm_encode.ml` | 二进制 Wasm 编码 |
| `dwarfsm_encode_context.ml` | 编码上下文 |
| `dwarfsm_encode_resolve.ml` | 编码时符号解析 |
| `dwarfsm_encode_wasm.ml` | 文本格式 Wasm 编码 |
| `dwarfsm_elim_equivdefn.ml` | 等价定义去重 |
| `dwarfsm_local_resolve.ml` | 局部变量解析 |
| `dwarfsm_parse.ml` | Dwarfsm 解析 |
| `wasmlinear_constr.ml` | Wasm 线性指令构造器 |
| `wasmgc_constr.ml` | Wasm GC 特定指令构造器 |
| `wasmir_util.ml` | Wasm IR 工具函数 |
| `wasm_lex.ml` | Wasm 文本格式词法 |
| `shrink_wasmir.ml` | Wasm IR 瘦身优化 |

### 8. 包系统与依赖管理

| 文件 | 职责 |
|------|------|
| `pkg.ml` | 包加载、查找 |
| `pkg_info.ml` | 包信息 |
| `pkg_path_tbl.ml` | 包路径表 |
| `pkg_config_util.ml` | 解析 `moon.pkg.json`（导入配置、main检测、warn/alert） |
| `parsing_import_path.ml` | 解析导入路径 |
| `mi_format.ml` | MoonBit 接口文件 (`.mi`) 格式 |

相关文件：
| 文件 | 职责 |
|------|------|
| `qual_ident_tbl.ml` | 限定标识符表 |
| `placeholder_env.ml` | 占位符环境 |
| `exported_functions.ml` | 导出函数管理 |
| `grouped_typedefs.ml` | 分组类型定义 |

### 9. 入口与驱动

`moonc` 二进制支持以下子命令，入口文件为 `moon0_main.ml`：

| 子命令 | 流水线覆盖范围 |
|--------|----------------|
| `check` | 解析 → 类型检查 → 生成 `.mi` |
| `build-package` | 解析 → 类型检查 → Core IR → 优化 → 导出 `.core` + `.mi` |
| `compile` | 完整流水线 → `.wasm` |
| `link-core` | Core 链接 → 单态化 → Wasm 生成 |
| `bundle-core` | 多个 `.core` 文件合并 |
| `gen-test-info` | 提取测试信息 |

关键驱动文件：
| 文件 | 职责 |
|------|------|
| `moon0_main.ml` | 主入口点，子命令分发 |
| `driver_util.ml` | 核心驱动逻辑，定义流水线的精确执行顺序 |
| `driver_config.ml` | 命令行参数配置 |
| `driver_compenv.ml` | 编译环境参数解析 |
| `compile_env.ml` | 编译环境重置管理 |

**`driver_util.ml`** 是核心驱动逻辑，定义了流水线的精确执行顺序和各阶段的回调接口。核心调用链：

```
parse → postprocess_ast → tast_of_ast → core_of_tast → monofy_core_link
→ clam_of_mcore → wasm_gen → (dwarfsm_encode)
```

### 10. 基础设施层

#### 基础数据结构 (`basic_*.ml`)

| 文件 | 说明 |
|------|------|
| `basic_prelude.ml` | 基础预定义 |
| `basic_alist.ml` | 关联列表 |
| `basic_arr.ml` | 数组工具 |
| `basic_base16.ml` / `basic_base64.ml` | Base16/Base64 编解码 |
| `basic_bigint.ml` | 大整数 |
| `basic_binary_search.ml` | 二分搜索 |
| `basic_byteseq.ml` | 字节序列 |
| `basic_compress_stamp.ml` | 压缩标签 |
| `basic_config.ml` | 全局配置 |
| `basic_core_ident.ml` | Core IR 标识符 |
| `basic_diet.ml` / `basic_diet_gen.ml` / `basic_diet_intf.ml` | 离散区间树 |
| `basic_duplicate_check.ml` | 重复检查 |
| `basic_encoders.ml` | 编码器 |
| `basic_fn_address.ml` | 函数地址 |
| `basic_hash*.ml` | Hash 表系列（通用、Int、String、Set） |
| `basic_ident.ml` | 通用标识符 |
| `basic_int.ml` | 整数工具 |
| `basic_io.ml` | I/O 操作 |
| `basic_iter.ml` / `basic_iter_utils.ml` | 迭代器 |
| `basic_json_utils.ml` | JSON 工具 |
| `basic_longident.ml` | 长标识符（含包路径） |
| `basic_lst.ml` | 列表工具 |
| `basic_map*.ml` | Map 系列 |
| `basic_qual_ident.ml` | 限定标识符 |
| `basic_ref.ml` | 引用类型 |
| `basic_scc.ml` | 强连通分量 |
| `basic_set*.ml` | Set 系列 |
| `basic_strutil.ml` | 字符串工具 |
| `basic_type_path.ml` | 类型路径 |
| `basic_ty_ident.ml` | 类型标识符 |
| `basic_uchar_utils.ml` | Unicode 字符工具 |
| `basic_uint32.ml` / `basic_uint64.ml` | 无符号整数 |
| `basic_unsafe_external.ml` | 不安全外部调用 |
| `basic_utf8.ml` | UTF-8 支持 |
| `basic_uuid.ml` | UUID 生成 |
| `basic_vec*.ml` | Vec（动态数组）系列 |
| `basic_vlq64.ml` | 可变长度量编码 |

#### 诊断系统

| 文件 | 职责 |
|------|------|
| `diagnostics.ml` | 错误报告系统核心（收集、过滤、检查） |
| `errors.ml` | 各类错误消息定义 |
| `error_code.ml` / `error_code_utils.ml` | 错误码管理（`E####` 格式） |
| `local_diagnostics.ml` | 局部诊断 |
| `warnings.ml` | 警告管理 |

#### 全局编译环境

| 文件 | 职责 |
|------|------|
| `global_env.ml` | 全局编译环境（类型表、函数表、方法表） |
| `global_ctx2.ml` | 全局上下文 |
| `local_env.ml` | 局部环境 |
| `local_type.ml` | 局部类型 |
| `local_typing_worklist.ml` | 局部类型检查工作列表 |
| `compile_env.ml` | 编译环境重置管理 |

#### S表达式（调试输出）

| 文件 | 职责 |
|------|------|
| `w.ml`（S表达式类型） | S表达式核心类型定义和格式化 |
| `moon_sexp_conv.ml` | S表达式转换（类似 ppx_sexp_conv） |
| `sexp_token.ml` | S表达式 Token |
| `sexp_visitors.ml` | S表达式遍历器 |

#### JSON 解析（用于 moon.pkg.json）

| 文件 | 职责 |
|------|------|
| `json.ml` | JSON 数据模型 |
| `json_types.ml` | JSON 类型定义 |
| `json_lexer.ml` | JSON 词法分析 |
| `json_parse.ml` | JSON 解析 |
| `json_literal.ml` | JSON 字面量 |

#### 内建与基元

| 文件 | 职责 |
|------|------|
| `builtin.ml` | MoonBit 语言内建类型和函数 |
| `primitive.ml` | 基元操作（算术、比较、位运算等） |
| `moon_intrinsic.ml` | MoonBit 内建函数 |
| `moon_sexp_conv.ml` | MoonBit S表达式转换 |

#### 属性系统

| 文件 | 职责 |
|------|------|
| `attribute.ml` | 属性/注解数据模型 |
| `attribute_parser.ml` | 属性解析 |
| `checked_attributes.ml` | 已验证属性 |

#### Trait 系统

| 文件 | 职责 |
|------|------|
| `trait_decl.ml` | Trait 声明处理 |
| `trait_impl.ml` | Trait 实现处理 |
| `trait_closure.ml` | Trait 闭包转换 |

#### 代码生成辅助

| 文件 | 职责 |
|------|------|
| `name_mangle.ml` | 符号名称修饰 |
| `const_table.ml` | 常量表 |
| `const_util.ml` | 常量工具 |
| `fn_arity.ml` | 函数元数信息 |
| `derive.ml` / `derive_args.ml` | derive 宏扩展 |
| `constr_info.ml` | 构造器信息 |

#### 其他

| 文件 | 职责 |
|------|------|
| `tag.ml` | 构造器标签 |
| `label.ml` | 记录标签 |
| `loc.ml` / `rloc.ml` | 位置信息（源文件位置和相对位置） |
| `info.ml` | 编译信息 |
| `join.ml` | join 点（控制流合并） |
| `control_ctx.ml` | 控制上下文 |
| `dce_context.ml` | 死代码消除上下文 |
| `dead_code.ml` | 死代码检测 |
| `check_match.ml` | match 完整性检查 |
| `check_purity.ml` | 纯度检查 |
| `transl_match.ml` | match 翻译 |
| `or_pat.ml` | or 模式处理 |
| `pat_path.ml` | 模式路径 |
| `patmatch_db.ml` / `patmatch_static_info.ml` | 模式匹配数据库/静态信息 |
| `pattern_id.ml` | 模式ID |
| `specialize_operator.ml` | 运算符特化 |
| `object_util.ml` | 对象工具 |
| `runtime_gc.ml` / `runtime_gc_js_string_api.ml` | 运行时 GC 接口 |
| `addr_table_gc.ml` | 地址表（GC版本） |
| `printer.ml` | 代码打印 |
| `docstring.ml` | 文档注释 |
| `alerts.ml` | 告警管理 |
| `value_info.ml` / `value_tracing.ml` | 值信息/追踪 |
| `action.ml` | 动作/副作用追踪 |
| `version.ml` | 版本信息 |
| `git_commit.ml` | Git commit 信息 |
| `ice_catcher.ml` | 内部编译器错误（ICE）捕获 |

---

## 架构特点总结

1. **经典的多遍编译架构**：每层 IR 都有独立的 AST 定义，通过转换函数逐层降低抽象级别。

2. **三层 IR 设计**：`Core`（高级，带泛型）→ `Mcore`（中高级，单态化）→ `Clam`（低级，接近 Wasm GC），逐层降低抽象。

3. **OCaml 全栈**：编译器完全用 OCaml 编写，利用其强大的代数数据类型和模式匹配能力。

4. **Wasm GC 优先**：当前公开后端仅支持 Wasm GC 目标（`Config.target = Wasm_gc`），利用 GC 引用类型实现 MoonBit 的内存管理。

5. **单态化而非运行时泛型**：通过 `monofy` 模块在编译期展开泛型，避免运行时性能开销。

6. **模块化清晰但扁平组织**：所有源文件在单一 `src/` 目录下，通过文件名前缀实现逻辑分组（`parsing_*`, `lex_*`, `core_*`, `basic_*` 等）。

7. **无独立 LLVM 后端**：LLVM 后端标记为 nightly-only，不在本仓库中公开。

8. **完整的 FFI 支持**：通过 `extern` 关键字支持嵌入式和导入式外部函数调用。

9. **丰富的模式匹配**：支持构造器模式、记录模式、Map模式、数组模式、范围模式、or模式和别名模式。

10. **Trait系统**：类似 Rust 的 trait，支持与 impl 结合使用，通过 trait closure 实现运行时多态。
