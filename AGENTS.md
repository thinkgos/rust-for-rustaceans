# AGENTS.md — 《Rust for Rustaceans》中文翻译规范

本仓库是《Rust for Rustaceans》的中文翻译, 使用 mdBook 组织与构建.
本文档是翻译与写作规范, 所有新增和修改的译文必须遵守.

## 1. 项目结构

- `src/chapter-NN.md` — 各章译文, NN 为两位数字 (`chapter-00.md` ~ `chapter-13.md`)
- `src/SUMMARY.md` — mdBook 目录, 新增章节必须同步更新
- `book.toml` — mdBook 配置
- `book/` — 构建产物, 已被 gitignore, 不要提交
- CI: push 到 `main` 后自动 `mdbook build` 并部署到 GitHub Pages

## 2. 标点符号 (强制)

全文使用**英文(半角)标点**, 禁止使用全角中文标点:

| 使用       | 禁止                      |
| ---------- | ------------------------- |
| `,`        | `，` `、`                 |
| `.`        | `。`                      |
| `;`        | `；`                      |
| `:`        | `：`                      |
| `?` `!`    | `？` `！`                 |
| `(` `)`    | `（` `）`                 |
| `"` `"`    | `“` `”` `「` `」`         |
| `--`       | `——`                      |
| `...`      | `……`                      |

- 标点前不加空格, 标点后加一个空格 (句末标点后同样加空格再接下一句).
- 括号紧贴其内容, 括号外侧遵循上一条规则: 如 `栈(`stack`), 堆(`heap`).`
- 列举分隔使用 `,`, 不使用顿号 `、`: 如 `值, 变量和指针`.

## 3. 专业术语 (强制)

**保留原文, 不翻译**:

1. 代码元素: 关键字, 类型名, 函数/方法名, trait 名, 宏名, 属性, 模块/路径, crate 名, 命令行, 编译错误信息, URL, 文件名.
   一律用反引号包裹: `Box::new`, `macro_rules!`, `#[repr(C)]`, `std::thread::spawn`.
2. 工具与生态名称: Rust, Cargo, rustc, clippy, rustup, mdBook, Rustacean 等.
3. 惯用英文术语 (不译, 不加引号直写): trait, crate, token, panic, unwrap, lint, monomorphization, vtable, DST 等.
   注意: trait 不译作"特征", crate 不译作"库", panic 不译作"恐慌".

**概念性术语**: 译为中文, 但**首次出现时**附英文原文, 格式为 `中文(`english`)`, 后文直接用中文.
统一译法 (全书保持一致, 未列出的术语以先译章节为准):

| 英文                    | 译文     | 英文                    | 译文     |
| ----------------------- | -------- | ----------------------- | -------- |
| ownership               | 所有权   | borrow                  | 借用     |
| lifetime                | 生命周期 | scope                   | 作用域   |
| move                    | 移动     | copy / clone            | 复制/克隆 |
| drop                    | 释放     | leak                    | 泄漏     |
| variable / value        | 变量/值  | pointer / reference     | 指针/引用 |
| stack / heap            | 栈/堆    | place                   | 位置     |
| closure                 | 闭包     | iterator                | 迭代器   |
| generic / trait object  | 泛型/trait 对象 | smart pointer    | 智能指针 |
| fat pointer             | 胖指针   | shadowing               | 变量遮蔽 |
| thread                  | 线程     | concurrency / parallelism | 并发/并行 |
| exclusive / shared      | 独占/共享 | mutable / immutable    | 可变/不可变 |

## 4. 代码与清单

- 代码块一律用 ` ```rust ` 包裹. **代码本身不改动** (标识符, 结构, 数值保持原样), 代码内注释翻译为中文, 注释仍用英文标点.
- 代码清单编号: `清单 N-M`, N 为章号, M 为章内序号 (从 1 递增, 与原书对应).
- 清单标题写在代码块末尾的注释行: `// 清单 N-M: 标题`.
- 正文引用写法: `请看清单 N-M 中的代码`, `清单 N-M 显示了...`.
- 原书代码行尾的序号标注 `(1)` `(2)` 保留原样, 正文用 `(1)` `(2)` 引用.

## 5. 标题与结构

- 章文件: `chapter-NN.md` (NN 两位数字); H1 用中文数字: `# 第一章 基础`, `# 第十章 并发性`.
- 节/小节用 `##` `###`; 术语型标题可附英文: `## 不意外的(Unsurprising)`, `### 声明宏(Declarative Macros)`.
- `SUMMARY.md` 条目用阿拉伯数字, 与文件名一致: `- [第N章-标题](chapter-NN.md)`.
- 正文交叉引用用阿拉伯数字: `第3章`, `见第10章`.

## 6. 正文体例

- 第二人称统一用"你", 不用"您".
- 行内代码/标识符用反引号; 强调用 `**粗体**`; 原书斜体用 `*斜体*`.
- 中文与英文/行内代码之间不加空格: 如 `在`Rust`中`.
- 原书 NOTE 旁注统一写作: `> 注意: ...`; 带标题的旁注, 标题单独一行 `> 旁注标题`, 正文续接其后.
- 翻译要求: 忠实原文, 不增删内容. 语序可为通顺而调整, 但不得改变技术含义.
- 原书的交叉引用 (如 "in Chapter 10") 需核对并改写为译本实际章节号.
- 引用原书章节名/术语时, 以本仓库已译名称为准, 不要另造译名.

## 7. 工作流

- 每章一个分支 `feat-chapterNN`, 完成后合入 `main`.
- 提交信息: `feat: add chapterNN` / `fix: ...` / `chore: ...`.
- 本地预览: `mdbook serve`; 构建: `mdbook build`.
- 提交前自检:
  1. `SUMMARY.md` 链接与实际文件一致;
  2. 清单编号 `N-M` 连续且引用一致;
  3. 无全角标点;
  4. 术语与既有章节一致 (对照第 3 节术语表).
