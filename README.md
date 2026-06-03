# MoonBit 编译器源码阅读笔记

本目录是个人对 [MoonBit 编译器](https://github.com/moonbitlang/moonbit-compiler) 源码的阅读分析与学习笔记，由 DeepSeek 辅助分析生成。

## 文档索引

| 文档 | 说明 |
|------|------|
| [architecture.md](./architecture.md) | 编译器整体架构分析：流水线阶段、IR 设计、优化Pass、后端、包系统等 |
| [parsing.md](./parsing.md) | 语法分析与 AST 详解：词法分析、AST定义、递归下降解析器、运算符处理 |
| [type-system.md](./type-system.md) | 类型系统详解：Stype/Mtype/Ltype_gc 四层类型、unification、Wasm GC映射 |
| [ir-overview.md](./ir-overview.md) | 中间表示详解：Core/Mcore/Clam/Dwarfsm 四层IR的AST结构与转换关系 |
| [codegen.md](./codegen.md) | Wasm GC 代码生成详解：Clam→Dwarfsm转换、二进制编码、优化Pass |
| [optimization.md](./optimization.md) | 优化Pass详解：10个Pass的算法、目的和执行顺序 |
| [trait-system.md](./trait-system.md) | Trait系统详解：声明、实现、静态/动态分发、方法解析 |
| [pattern-match.md](./pattern-match.md) | 模式匹配编译详解：从语法模式到决策树的全流程 |
| [monomorphization.md](./monomorphization.md) | 单态化详解：泛型展开的工作列表算法、类型替换 |
| [error-handling.md](./error-handling.md) | 错误处理与Async变换：checked errors、CPS状态机 |

## 项目概览

MoonBit 编译器（`moonc`）使用 **OCaml** 编写，基于 **dune** 构建。所有源码集中在 `src/` 目录，约 250+ 个 `.ml` 文件，通过文件名前缀进行逻辑分组：

```
src/
├── basic_*.ml        基础数据结构与工具（Hash表、Map、Set、Vec、UTF8等）
├── lex_*.ml          词法分析（Token定义、字面量、注释、Unicode、分号插入）
├── parsing_*.ml      语法分析（递归下降解析器、AST定义、运算符处理）
├── typing_*.ml       类型信息存储
├── type*.ml          类型系统与类型检查（mtype、ctype、ltype、stype等）
├── typedtree*.ml     类型化AST
├── typer.ml          主类型检查器（双向检查、unification、推断）
├── pattern_*.ml      模式匹配（类型检查、静态信息）
├── core*.ml          Core IR（高级中间表示，含优化Pass）
├── mcore*.ml         Mcore IR（单态化后中间表示）
├── clam*.ml          Clam IR（低级中间表示，接近Wasm GC）
├── monofy*.ml        单态化（泛型展开）
├── pass_*.ml         优化Pass（内联、栈分配、Lambda提升等）
├── dwarfsm_*.ml      Dwarfsm（Wasm IR 的 OCaml 表示/编解码）
├── wasm*.ml          Wasm GC 代码生成
├── pkg*.ml           包系统（加载、路径、配置文件）
├── trait*.ml         Trait 系统（声明、实现、闭包）
├── driver*.ml        编译驱动（入口、流水线编排）
├── moon0_main.ml     主入口（moonc 二进制）
├── global_env.ml     全局编译环境
├── diagnostics.ml    诊断系统
├── errors.ml         错误定义
├── attribute*.ml     属性/注解系统
├── json*.ml          JSON 解析（moon.pkg.json）
└── ...
```

## 编译流水线

```
源文件 (.mbt)
  → 词法分析 (Lexing)
  → 语法分析 (Parsing) → Syntax AST
  → 类型检查 (Type Checking) → Typedtree
  → Core IR 转换 → 优化Pass → Core.program
  → 链接 → 单态化 → Mcore.t
  → Clam IR 转换 → 优化 → Clam.prog
  → Wasm GC 代码生成 → .wasm
```

## 子命令速查

| 子命令 | 功能 |
|--------|------|
| `check` | 类型检查并产出 `.mi` 接口文件 |
| `build-package` | 构建单包，产出 `.core` + `.mi` |
| `compile` | 完整编译流水线，产出 `.wasm` |
| `link-core` | 链接 Core IR 并生成 Wasm |
| `bundle-core` | 合并多个 `.core` 文件 |
| `gen-test-info` | 提取测试信息 |

## 外部资源

- [MoonBit 官网](https://www.moonbitlang.com)
- [MoonBit 文档](https://docs.moonbitlang.com)
- [MoonBit 语言之旅](https://tour.moonbitlang.com)
- [MoonBit 标准库](https://github.com/moonbitlang/core)
- [MoonBit 构建工具](https://github.com/moonbitlang/moon)
- [社区论坛](https://discuss.moonbitlang.com)
