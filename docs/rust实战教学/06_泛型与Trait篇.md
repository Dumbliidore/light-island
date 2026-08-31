---
title: 6. 泛型与Trait篇
icon: simple/rust
comments: true
tags:
  - Rust
---
# 6. 泛型与Trait篇

> 对应章节：第10章（泛型、trait、生命周期）。  
> 本篇掌握行为抽象（trait）、代码复用（泛型）与借用安全（生命周期）。

---

## 项目 13：形状库

**目标**：用 trait 抽象"能算面积的形状"，泛型复用代码，生命周期处理引用。

### 创建项目

```bash
cargo new shapes --lib
cd shapes
```

> 说明：这是**库项目**（--lib），生成 `src/lib.rs`。如需命令行演示，可自行添加 `src/main.rs`（bin 与 lib 同名会自动互认）。

### 功能要求

1. 定义 `Shape` trait（area + name）
2. 实现 `Circle`、`Rectangle`、`Triangle`
3. 泛型函数接收任意形状
4. 演示生命周期 `longest`

### 知识点

| 知识点 | 说明 |
|--------|------|
| `trait` 定义 | 行为抽象（接口） |
| `impl Trait for Type` | 实现 trait |
| 泛型 `<T: Shape>` | 类型参数 + trait 约束 |
| trait 对象 `&dyn Shape` | 动态分发 |
| 生命周期 `'a` | 引用有效范围 |
| 静态 vs 动态分发 | 编译期 vs 运行期 |

### 代码

`src/lib.rs`：

```rust
// trait 定义"行为"：任何实现它的类型都能计算面积、给出名字
pub trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

// 三个形状结构体（字段设为 pub，方便在 main 中直接构造）
pub struct Circle { pub radius: f64 }
pub struct Rectangle { pub width: f64, pub height: f64 }
pub struct Triangle { pub base: f64, pub height: f64 }

// 为 Circle 实现 Shape
impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str { "圆形" }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str { "矩形" }
}

impl Shape for Triangle {
    fn area(&self) -> f64 {
        self.base * self.height / 2.0
    }
    fn name(&self) -> &str { "三角形" }
}

// 泛型函数：T 必须实现 Shape
// 编译期把 T 替换为具体类型 → 静态分发，性能好
pub fn describe<T: Shape>(shape: &T) -> String {
    format!("{} 面积是 {:.2}", shape.name(), shape.area())
}

// trait 对象：&dyn Shape 表示"任意实现 Shape 的类型"
// 运行时查虚表（动态分发），可以装不同类型的形状混合使用
pub fn describe_all(shapes: &[&dyn Shape]) -> Vec<String> {
    shapes.iter().map(|s| describe(*s)).collect()
}

// 生命周期：返回的引用与较短的输入引用同寿命
// 'a 是编译器用来检查"返回的引用不会悬空"的标注
pub fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

`src/main.rs`：

```rust
use geometry::{Circle, Rectangle, Shape, Triangle, describe, describe_all, longest};

fn main() {
    let c = Circle { radius: 3.0 };
    let r = Rectangle { width: 4.0, height: 5.0 };
    let t = Triangle { base: 6.0, height: 2.0 };

    // 泛型：编译期确定具体类型
    println!("{}", describe(&c));
    println!("{}", describe(&r));
    println!("{}", describe(&t));

    // trait 对象（动态分发）：三种不同类型放进同一个数组
    let shapes: [&dyn Shape; 3] = [&c, &r, &t];
    for s in describe_all(&shapes) {
        println!("{}", s);
    }

    // 生命周期：longest 返回的引用与传入参数绑定
    let s1 = String::from("hello");
    let s2 = String::from("world");
    let result = longest(&s1, &s2);   // result 借用 s1/s2，它们不能提前释放
    println!("最长的是：{}", result);
}
```

### 运行

```bash
cargo run
```

### 注意事项

1. **泛型 vs trait 对象**：`describe<T: Shape>` 编译期展开（静态分发，性能好）；`&dyn Shape` 运行时查虚表（动态分发，灵活，可混合类型）。按需选择。
2. **生命周期 `'a` 是约束**：`longest<'a>(a: &'a str, b: &'a str) -> &'a str` 表示"返回值和输入引用寿命相同"——编译器用这个约束检查借用不悬空。
3. **`&[&dyn Shape]`**：数组元素是引用，引用指向的是 trait 对象。
4. **trait 需要 `pub` 才能跨 crate 使用**：`describe`/`describe_all`/`longest` 在 main 中调用，必须 `pub`。

### 进阶：纯库 + 单元测试

- **trait 更小**：`Area` 只有 `area` 方法——接口越小，实现越省
- **Triangle 用三边**：字段 `a/b/c` 配海伦公式，而不是底×高÷2
- **固有 `impl` 构造器**：每个形状配 `new` 关联函数（注意没加 `pub`，见要点3）
- **`total_area` 动态分发求和**：`&[&dyn Area]` 混装三种形状
- **没有 main.rs**：纯库项目，正确性全靠单元测试验证


`src/lib.rs`：

```rust
pub trait Area {
    fn area(&self) -> f64;
}

pub struct Circle {
    pub radius: f64,
}

pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

pub struct Triangle {
    pub a: f64,
    pub b: f64,
    pub c: f64,
}

impl Area for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

// 固有 impl：类型自己的方法，不属于任何 trait
impl Circle {
    fn new(radius: f64) -> Self {
        Circle { radius }
    }
}

impl Area for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

impl Rectangle {
    fn new(width: f64, height: f64) -> Self {
        Rectangle { width, height }
    }
}

impl Triangle {
    fn new(a: f64, b: f64, c: f64) -> Self {
        Triangle { a, b, c }
    }
}

impl Area for Triangle {
    fn area(&self) -> f64 {
        // 海伦公式：s = 半周长，面积 = √(s(s-a)(s-b)(s-c))
        let s = (self.a + self.b + self.c) / 2.0;
        (s * (s - self.a) * (s - self.b) * (s - self.c)).sqrt()
    }
}

// trait 对象切片：不同形状混在一起求总面积（动态分发）
pub fn total_area(shapes: &[&dyn Area]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

pub fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn circle_area() {
        let circle = Circle { radius: 5.0 };
        assert_eq!(circle.area(), std::f64::consts::PI * 25.0);
    }

    #[test]
    fn rectangle_area() {
        let rectangle = Rectangle {
            width: 5.0,
            height: 10.0,
        };
        assert_eq!(rectangle.area(), 50.0);
    }

    #[test]
    fn triangle_area() {
        let triangle = Triangle {
            a: 3.0,
            b: 4.0,
            c: 5.0,
        };
        assert_eq!(triangle.area(), 6.0);   // 3-4-5 直角三角形
    }

    #[test]
    fn total_area_of_shapes() {
        let c1 = Circle::new(1.0);
        let c2 = Circle::new(2.0);
        let shapes: Vec<&dyn Area> = vec![&c1, &c2];

        let total = total_area(&shapes);
        // 浮点数不能直接比相等，用误差范围判断
        assert!((total - 15.7079).abs() < 0.0001);
    }

    #[test]
    fn total_area_mixed() {
        let c = Circle::new(1.0);
        let r = Rectangle::new(3.0, 4.0);
        let t = Triangle::new(3.0, 4.0, 5.0);
        let shapes: Vec<&dyn Area> = vec![&c, &r, &t];

        let total = total_area(&shapes);
        assert!((total - 21.1416).abs() < 0.001);
    }

    #[test]
    fn longest_test() {
        let s1 = String::from("short");
        let s2 = String::from("longer");
        assert_eq!(longest(&s1, &s2), &s2[..]);
    }
}
```

### 运行

```bash
cargo test
# running 6 tests
# test tests::circle_area ... ok
# test tests::rectangle_area ... ok
# ...
# test result: ok. 6 passed
```

### 要点

1. **接口设计取舍**：`Area`（单方法）vs `Shape`（area+name）——不需要名字时，小 trait 让实现者少写代码；需要时再加。trait 可以之后扩展，但删除方法算破坏性变更。
2. **固有 impl vs trait impl**：`Circle::new` 写在普通 `impl Circle` 里（类型自带能力）；`fn area` 写在 `impl Area for Circle` 里（满足契约）。调用方式也不同：`Circle::new(1.0)` vs `c.area()`。
3. **可见性实战**：`new` 没加 `pub`，库外部调不到它；单元测试在同一个 crate 内，所以能用——这就是 `pub` 边界的实际含义。想让外部用构造器就加 `pub`。
4. **浮点断言**：`(total - expected).abs() < 0.0001` 而不是 `assert_eq!`——二进制浮点有舍入误差，精确比较会翻车。
