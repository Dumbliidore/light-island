---
title: 4. 第三方crate篇
icon: simple/rust
comments: true
tags:
  - Rust
---
# 4. 第三方crate篇

> 对应章节：第8章。  
> 本篇掌握引入第三方 crate 与文件持久化。

---

## 项目 8：简单 HTTP 客户端

**目标**：引入第三方 crate（reqwest），发送 HTTP 请求获取网页内容。

### 创建项目

```bash
cargo new http_client
cd http_client
cargo add reqwest --features blocking   # 添加最新版 reqwest，启用 blocking 特性
```

### 功能要求

1. 输入一个 URL
2. 发送 GET 请求
3. 打印状态码（数字 + 原因短语）和响应头数量
4. 打印正文字节数与前 200 字符预览

### 知识点

| 知识点 | 说明 |
|--------|------|
| 第三方 crate | Cargo.toml 引入 |
| `reqwest::blocking` | 阻塞式 HTTP |
| `Result` 错误处理 | 网络错误处理 |
| `?` 运算符 | 错误传播 |
| `Box<dyn Error>` | 任意错误 trait 对象 |
| `status().canonical_reason()` | 状态码原因短语，可能为 None 用 `unwrap_or` 兜底 |
| `headers().len()` | 响应头数量 |
| `body.len()` | 字节数（非字符数，中文占多字节） |

### 代码

依赖已由 `cargo add reqwest --features blocking` 添加（`cargo search reqwest` 可查最新版本），无需手动编辑 Cargo.toml。

`src/main.rs`：

```rust
use std::env;

// main 返回 Result：允许用 ? 传播错误，出错时进程以非零码退出
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args: Vec<String> = env::args().collect();
    let url = &args[1];

    // blocking：阻塞直到收到响应；? 出错自动返回
    let response = reqwest::blocking::get(url)?;

    let status = response.status();
    // as_u16 拿数字，canonical_reason 拿原因短语（"OK"/"Not Found"...）
    // 非标准状态码的 reason 可能是 None，用 unwrap_or 兜底
    println!(
        "Status: {} {}",
        status.as_u16(),
        status.canonical_reason().unwrap_or("")
    );

    let header_count = response.headers().len();
    println!("Header count: {}", header_count);

    let body = response.text()?;   // 读取正文
    println!("正文长度: {} 字节", body.len());   // len() 是字节数

    // chars().take(200)：只取前 200 个字符（避免中文按字节截断）
    let preview: String = body.chars().take(200).collect();
    println!("正文开头：{}", preview);

    Ok(())
}
```

### 运行

```bash
cargo run -- https://example.com
# Status: 200 OK
# Header count: 6
# 正文长度: 1256 字节
# 正文开头：<!doctype html>...
```

### 注意事项

1. **`Result<(), Box<dyn std::error::Error>>`**：`Box<dyn Error>` 是"任意错误"的 trait 对象，允许 `?` 混用不同类型错误——这是 main 函数传播错误的常用写法。
2. **`?` 运算符**：`get(...)?` 出错时自动返回错误，代码不用层层 match——第9章详细讲。
3. **`features = ["blocking"]`**：reqwest 默认是异步的，要加 blocking 特性才有同步 API。
4. **首次 `cargo build` 较慢**：要下载编译依赖，属正常。
5. **`unwrap_or("")` 兜底 Option**：`canonical_reason()` 对非标准状态码返回 `None`，用空字符串兜底而不是 unwrap 崩溃——比项目1的 expect 又进了一步。
6. **改进练习**：不带参数直接运行会因 `args[1]` 越界 panic，参考项目2加上用法提示。

---

## 项目 9：命令行笔记管理器

**目标**：基于文件的增删查，掌握文件持久化。

### 创建项目

```bash
cargo new notes
cd notes
```

### 功能要求

```
cargo run -- list              # 查看所有笔记
cargo run -- add "买牛奶"      # 添加
cargo run -- remove 1          # 删除编号
```

笔记保存到 `notes.txt`。

### 知识点

| 知识点 | 说明 |
|--------|------|
| `fs::read_to_string` | 读文件 |
| `fs::write` | 写文件 |
| `args` 解析 | 子命令模式 |
| 文本行持久化 | `join("\n")` |
| `map(String::from)` | `&str` 转 `String` |

### 代码

`src/main.rs`：

```rust
use std::env;
use std::fs;

const FILE: &str = "notes.txt";   // 常量：文件路径

// 读取笔记到 Vec<String>（每行一条）
fn load_notes() -> Vec<String> {
    // 文件不存在时返回空 Vec（不 panic，允许首次运行）
    match fs::read_to_string(FILE) {
        Ok(content) => content.lines().map(String::from).collect(),
        Err(_) => Vec::new(),
    }
}

// 保存：把 Vec 用换行连接后一次性写入
fn save_notes(notes: &[String]) {
    let content = notes.join("\n");
    fs::write(FILE, content).expect("保存失败");
}

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() < 2 {
        println!("用法：cargo run -- list | add <内容> | remove <编号>");
        return;
    }

    let mut notes = load_notes();   // 每次启动从文件读入内存

    // 子命令模式：args[1] 是命令名
    match args[1].as_str() {
        "list" => {
            for (i, n) in notes.iter().enumerate() {
                println!("{}. {}", i + 1, n);
            }
        }
        "add" => {
            if args.len() >= 3 {
                // args[2..].join(" ")：多个单词拼成一条笔记（含空格）
                notes.push(args[2..].join(" "));
                save_notes(&notes);   // 改完立刻写回
                println!("已添加");
            }
        }
        "remove" => {
            if let Ok(idx) = args[2].parse::<usize>() {
                // 1-based 转 0-based 需减 1，先做边界检查
                if idx >= 1 && idx <= notes.len() {
                    notes.remove(idx - 1);
                    save_notes(&notes);
                    println!("已删除");
                }
            }
        }
        _ => println!("未知命令"),
    }
}
```

### 运行

```bash
cargo run -- add "学习 Rust"
cargo run -- add "写笔记管理器"
cargo run -- list       # 1. 学习 Rust  2. 写笔记管理器
cargo run -- remove 1
cat notes.txt
```

### 注意事项

1. **文件持久化**：`load_notes` 每次启动读取，`save_notes` 每次修改后写回——数据跨进程存活。
2. **`args[2..].join(" ")`**：`add` 时把多个单词拼接成一条笔记（"买牛奶"中间有空格也完整保存）。
3. **`map(String::from)`**：`content.lines()` 产生 `&str`，需要 `String::from` 转成拥有所有权的 `String` 存入 Vec。
4. **子命令模式**：`args[1]` 作为命令名，是命令行工具最常见的结构。

### 进阶：JSON + 标签系统

重写存储层，升级点：

- **JSON 存储**：serde 序列化整个笔记库（对比原版的纯文本行）
- **HashMap 索引**：`HashMap<String, Note>` 以标题为键，O(1) 查找
- **标签系统**：每条笔记可打多个标签，支持按标签搜索
- **模块拆分**：`Note` 结构体独立到 `src/note.rs`

```bash
cargo add serde --features derive
cargo add serde_json
```

`src/main.rs`：

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::env;
mod note;

use note::Note;

const FILE: &str = "notes.json";

// 整个笔记库的容器：标题 → 笔记内容
#[derive(Deserialize, Serialize)]
struct Store {
    notes: HashMap<String, Note>,
}

fn load_store() -> Store {
    match std::fs::read_to_string(FILE) {
        Ok(content) => serde_json::from_str(&content).expect("JSON 解析失败"),
        Err(_) => Store {
            notes: HashMap::new(),   // 首次运行：文件不存在
        },
    }
}

fn save_store(store: &Store) {
    let serialized = serde_json::to_string(store).expect("JSON 序列化失败");
    std::fs::write(FILE, serialized).expect("保存失败");
}

fn main() {
    let args: Vec<String> = env::args().collect();

    if args.len() < 2 {
        println!("Usage: cargo run -- <command> [args...]");
        return;
    }

    let mut store = load_store();

    let command = args[1].as_str();

    match command {
        // 用法：cargo run -- add <标题> <内容>
        "add" => {
            let n = Note {
                tags: Vec::new(),
                content: args[3].clone(),
            };
            store.notes.insert(args[2].clone(), n);
            println!("Note added: {}", args[2]);
        }
        "list" => {
            for (i, (title, _)) in store.notes.iter().enumerate() {
                println!("{}: {}", i + 1, title);
            }
        }
        "show" => {
            if let Some(n) = store.notes.get(&args[2]) {
                println!(
                    "标题：{}\n标签：{}\n内容：{}",
                    args[2],
                    n.tags.join(" "),
                    n.content
                );
            } else {
                println!("Note not found: {}", args[2]);
            }
        }
        "remove" => {
            // remove 返回 Option：Some(被删值) 说明确实存在
            if store.notes.remove(&args[2]).is_some() {
                println!("Note removed: {}", args[2]);
            } else {
                println!("Note not found: {}", args[2]);
            }
        }
        "tag" => {
            // get_mut 拿可变借用才能 push 标签
            match store.notes.get_mut(&args[2]) {
                Some(n) => {
                    if !n.tags.contains(&args[3]) {
                        n.tags.push(args[3].clone());
                        println!("Tag added: {}", args[3]);
                    } else {
                        println!("{} 已有标签 {}", args[2], args[3]);
                    }
                }
                None => {
                    println!("Note not found: {}", args[2]);
                }
            }
        }
        "search" => {
            let keyword = &args[2];
            // any()：只要有一个标签匹配就算命中
            let found: Vec<_> = store
                .notes
                .iter()
                .filter(|(_, n)| n.tags.iter().any(|t| t == keyword))
                .map(|(title, _)| title)
                .collect();

            if found.is_empty() {
                println!("No notes found with keyword: {}", keyword);
            } else {
                for title in found {
                    println!("- {}", title);
                }
            }
        }
        _ => {
            println!("Unknown command: {}", command);
        }
    }
    save_store(&store);   // 无论什么命令，最后统一写回
}
```

`src/note.rs`：

```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize)]
pub struct Note {
    pub tags: Vec<String>,
    pub content: String,
}
```

### 运行

```bash
cargo run -- add rust笔记 "今天学了所有权"
cargo run -- tag rust笔记 学习
cargo run -- add todo "明天买牛奶"
cargo run -- list
cargo run -- show rust笔记
cargo run -- search 学习        # 按标签搜出 rust笔记
cat notes.json                  # 查看 JSON 结构
```

### 要点

1. **JSON vs 文本行**：原版一行一条笔记，结构固定；JSON 版可以给每条笔记挂任意字段（标签列表），扩展时不用改存储格式。
2. **`HashMap` 的键即 ID**：以标题为键天然去重——但同名 add 会静默覆盖旧笔记，想想怎么改成先检查再插入。
3. **`get_mut` 才能改数据**：`tag` 命令要 push 标签，必须拿 `&mut Note`。
4. **迭代器链搜索**：`iter().filter(...).map(...).collect()` 三段式是集合查询的标准套路，`any()` 则是"存在性判断"。

---

## 项目 10：待办事项管理器（文件持久化版）

> 本项目是[项目4](02_结构体与集合篇.md)的进阶版：解决数据存内存、退出即失忆的问题，升级为子命令式 CLI，数据持久化到 `todo.txt`。

### 功能要求

```
cargo run -- add <内容>       # 添加
cargo run -- list             # 列出（[x]/[ ] 标记完成态）
cargo run -- done <编号>      # 标记完成
cargo run -- remove <编号>    # 删除
```

### 知识点

| 知识点 | 说明 |
|--------|------|
| 结构体携带状态 | `Todo { text, done }` |
| `fs::read_to_string` / `fs::write` | 文件读写 |
| `.lines().filter().map()` | 迭代器链解析每一行 |
| `get_mut` | 安全获取可变借用 |
| 子命令分发 | `match args[1]` |

### 创建项目

```bash
cargo new todo
cd todo
```

`src/main.rs`：

```rust
use std::{env, fs};

struct Todo {
    text: String,
    done: bool,
}

const FILE: &str = "todo.txt";

fn load_todos() -> Vec<Todo> {
    match fs::read_to_string(FILE) {
        Ok(content) => content
            .lines()
            .filter(|line| !line.is_empty())   // 跳过空行
            .map(parse_todo)
            .collect(),
        Err(_) => Vec::new(),   // 文件不存在（第一次运行）→ 空列表
    }
}

// 把一行 "[x] 买牛奶" 解析回 Todo 结构体
fn parse_todo(line: &str) -> Todo {
    let done = line.starts_with("[x]");
    let text = line[4..].to_string();   // 切掉前缀 "[x] " 或 "[ ] "
    Todo { text, done }
}

fn save_todos(todos: &[Todo]) {
    let mut content = String::new();
    for todo in todos {
        let mark = if todo.done { "[x]" } else { "[ ]" };
        content.push_str(&format!("{} {}\n", mark, todo.text));
    }
    fs::write(FILE, content).expect("写入文件失败");
}

fn main() {
    let args: Vec<String> = env::args().collect();

    let command = args[1].as_str();
    match command {
        "add" => {
            let mut todos = load_todos();
            todos.push(Todo {
                text: args[2].clone(),
                done: false,
            });
            save_todos(&todos);
            println!("Added: {}", args[2]);
        }
        "list" => {
            let todos = load_todos();
            for (i, todo) in todos.iter().enumerate() {
                let mark = if todo.done { "[x]" } else { "[ ]" };
                println!("{:>2} {} {}", i + 1, mark, todo.text);
            }
        }
        "done" => {
            let mut todos = load_todos();
            let index: usize = match args[2].parse() {
                Ok(n) => n,
                Err(_) => {
                    println!("Invalid index: {}", args[2]);
                    return;
                }
            };

            // get_mut 返回 Option<&mut Todo>：可变借用才能改 done 字段
            match todos.get_mut(index - 1) {
                Some(todo) => {
                    todo.done = true;
                    println!("Marked as done: {}", todo.text);
                    save_todos(&todos);
                }
                None => {
                    println!("Invalid index: {}", index);
                }
            }
        }
        "remove" => {
            let mut todos = load_todos();
            let index: usize = match args[2].parse() {
                Ok(n) => n,
                Err(_) => {
                    println!("Invalid index: {}", args[2]);
                    return;
                }
            };

            if index >= 1 && index <= todos.len() {
                let removed = todos.remove(index - 1);
                save_todos(&todos);
                println!("Removed: {}", removed.text);
            } else {
                println!("Invalid index: {}", index);
            }
        }
        _ => {
            println!("Unknown command: {}", command);
        }
    }
}
```

### 运行

```bash
cargo run -- add 买牛奶
cargo run -- add 写作业
cargo run -- list          #  1 [ ] 买牛奶 ...
cargo run -- done 1        # Marked as done: 买牛奶
cargo run -- list          #  1 [x] 买牛奶（重启后依然是完成态）
cargo run -- remove 2      # Removed: 写作业
```

### 要点

1. **读写循环**：每次命令都是 `load → 修改 → save` 全量读写小文件，简单可靠；数据量大时才需要真正的数据库。
2. **文本即协议**：`todo.txt` 每行一条记录，前缀 `[x]`/`[ ]` 就是存储格式——序列化最朴素的形态。JSON 方案见项目11。
3. **`get_mut` vs 索引赋值**：`todos.get_mut(i)` 返回 `Option<&mut Todo>`，越界安全；比直接 `todos[i].done = true` 多一层保护。
4. **改进练习**：不带参数运行会因 `args[1]` 越界 panic，参考项目4加上用法提示。

---

## 项目 11：联系人通讯录（JSON 序列化版）

> 本项目是[项目5](02_结构体与集合篇.md)的进阶版，引入 serde 把整个 HashMap 序列化为 `contacts.json`，并把数据层拆分到独立模块。

### 功能要求

```
cargo run -- add <姓名> <电话>    # 添加
cargo run -- list                 # 列出全部
cargo run -- remove <姓名>        # 删除
cargo run -- search <关键字>      # 按姓名模糊搜索
```

### 知识点

| 知识点 | 说明 |
|--------|------|
| `#[derive(Serialize, Deserialize)]` | 一行获得 JSON 双向转换 |
| `serde_json::from_str` / `to_string_pretty` | 反序列化 / 格式化输出 |
| `Err(_) => 空 map` | 首次运行零配置 |
| `mod contacts;` | 数据层拆分独立文件 |
| `filter + contains` | 模糊搜索 |

### 创建项目

```bash
cargo new contacts
cd contacts
cargo add serde --features derive   # Serialize/Deserialize 派生宏
cargo add serde_json                # JSON 解析与生成
```

`src/main.rs`：

```rust
use std::collections::HashMap;
use std::env;

mod contacts;

fn main() {
    let mut contacts = contacts::Contacts::load();
    let args: Vec<String> = env::args().collect();

    let command = args[1].as_str();

    match command {
        "add" => {
            contacts.map.insert(args[2].clone(), args[3].clone());
            println!("已保存： {} {}", args[2], args[3]);
        }

        "list" => {
            for (key, phone) in &contacts.map {
                println!("{}: {}", key, phone);
            }
        }
        "remove" => {
            let key = &args[2];
            if let Some(phone) = contacts.map.remove(key) {
                println!("已删除： {} {}", key, phone);
            } else {
                println!("没有联系人： {}", key);
            }
        }
        "search" => {
            let key = &args[2];
            // filter：键里包含关键字的全部留下（contains 是模糊匹配）
            let matches: Vec<_> = contacts
                .map
                .iter()
                .filter(|(k, _)| k.contains(key))
                .collect();

            if matches.is_empty() {
                println!("没有匹配的联系人： {}", key);
            } else {
                for (k, v) in &matches {
                    println!("{}: {}", k, v);
                }
            }
        }
        _ => {
            println!("Invalid command");
        }
    }
    contacts.save();
}
```

`src/contacts.rs`：

```rust
use std::collections::HashMap;
use std::fs;

// derive 让 Contacts 自动获得 JSON 序列化/反序列化能力
#[derive(serde::Serialize, serde::Deserialize)]
pub struct Contacts {
    pub map: HashMap<String, String>,
}

const FILE: &str = "contacts.json";

impl Contacts {
    pub fn load() -> Contacts {
        match fs::read_to_string(FILE) {
            Ok(content) => serde_json::from_str(&content).expect("JSON 解析错误"),
            Err(_) => Contacts {
                map: HashMap::new(),   // 第一次运行：文件还不存在
            },
        }
    }

    pub fn save(&self) {
        let json = serde_json::to_string_pretty(self).expect("JSON 序列化错误");
        fs::write(FILE, json).expect("写入文件失败");
    }
}
```

### 运行

```bash
cargo run -- add 张三 13800000000
cargo run -- add 张伟 13900000000
cargo run -- list
cargo run -- search 张       # 模糊匹配出两条
cargo run -- remove 张三
cat contacts.json            # 查看 JSON 长什么样
```

### 要点

1. **值简化为 `String`**：只需要电话号码时，`HashMap<String, String>` 比 `HashMap<String, Contact>` 更直接——不必为了结构体而结构体。
2. **`#[derive(Serialize, Deserialize)]`**：一行代码让结构体获得 JSON 双向转换能力，这是 Rust 生态最常用的持久化套路。
3. **`Err(_) => 空 map`**：load 时把"文件不存在"当作空数据处理，程序首次运行零配置可用。
4. **`mod contacts;` 分文件**：数据层（load/save）与命令逻辑分离，模块组织见 [03_模块系统篇](03_模块系统篇.md)。
