title: Hello Rust
date: 2025-07-12 18:27:35
tags:
- Rust
categories:
- Program
---
# Hello Rust

近来网上冲浪时，在 Github trending 上越来越多的看到一些 Rust 项目，也常看到一些项目的简介写着 “这是一款使用 Rust 开发的 xxx”。这令我不禁好奇，使用某种编程语言开发也能成为一个软件值得一提的特点吗？不过这确实引起了我的兴趣，简单的搜索一下，Rust 以其高性能和内存安全著称，软件作者通过强调使用 Rust，传达其软件在性能和安全性上的优势。

这两年的工作陷入了理不清的需求和讨论不完的产品方案之中，很久没有那种“长脑子”的感觉了。现在有点时间不如来学学 Rust，就算是赶下时髦（也许没赶上）？

# 边学边记

我找到了 [Rust 程序设计语言 简体中文版](https://kaisery.github.io/trpl-zh-cn/title-page.html)，现在快速的学习一下 Rust 吧。

*Tips: 这不是教程。我已经非常熟悉 Golang 和 Python，这里仅记录我学习过程中认为比较独特的地方。如果你也熟悉 Golang，可以参考我的笔记来快速了解 Rust。*

## 安装

我使用 VsCode on MacOS，在 Terminal 使用命令安装：
```shell
$ curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh

# 回车确认使用默认配置即可

$ source ~/.cargo/env 

$ rustc --version
rustc 1.88.0 (6b00bc388 2025-06-23)
```

## 基本概念

### 变量

使用`let`定义变量，变量默认是不可变的，使用`mut`关键字设置变量的可变性。
```rust
let x = 5;
let mut x = 5;

```

### 常量

使用`const`来定义常量。
```rust
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
```

### 遮蔽（shadowing）

能看懂这段代码就理解遮蔽了。
```rust
fn main() {
    let x = 5;

    let x = x + 1;

    {
        let x = x * 2;
        println!("The value of x in the inner scope is: {x}"); // output 12
    }

    println!("The value of x is: {x}"); // output 6
}
```

### 数据类型

- 标量类型
        - 整型
        - 浮点型
        - 布尔型
        - 字符
- 复合类型
        - 元组 tuple
        - 数组 array

数组的长度是固定的，更灵活的类型是 vector（什么是 vector，后面学习）。

### 函数

在 Rust 中，函数的返回值等同于函数体最后一个表达式的值。使用 return 关键字和指定值，可从函数中提前返回；但大部分函数隐式的返回最后的表达式。

要注意**表达式**和**语句**的概念。

两个有代表性的例子：
```rust
fn plus_one(x: i32) -> i32 {
    x + 1
    // x + 1; 注意：这里如果有分号会编译错误
}

fn five() -> i32 {
    5
}
```


### 控制流

#### if 表达式

```rust
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

注意的是代码中的条件必须是 bool 值，下面的写法会编译失败。
```rust
fn main() {
    let number = 3;

    if number { // 编译失败，必须显示的转为 bool 类型
        println!("number was three");
    }
}
```

if 是表达式，不是语句，所以可以把 if 表达式的结果复制给变量。
```rust
    let condition = true;
    let number = if condition { 5 } else { 6 };
```

#### loop

loop 可以有返回值。  
break 后可以添加返回值。

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {result}");
}
```

**循环标签**：如果存在嵌套的循环，可以用通过标签来指定break或continue哪一层循环。（我不喜欢这个特性，有点像其他语言中的 goto）

#### while

没什么特别的。
```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{number}!");

        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```

#### for 

for 可以用来遍历集合中的元素。  

还可以遍历范围表达式（range expression）。

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {element}");
    }

    for (index, element) in a.iter().enumerate() {
        println!("index: {index} value : {element}");
    }

    for number in (1..4).rev() {
        println!("{number}!");
    }
    println!("LIFTOFF!!!");
}
```

(心声： for in 和 enumerate 有点像 Python [:thinking_face:])

## 所有权

> 
    一些语言中具有垃圾回收机制，在程序运行时有规律地寻找不再使用的内存；   
    在另一些语言中，程序员必须亲自分配和释放内存。  

    Rust 则选择了第三种方式：通过所有权系统管理内存，编译器在编译时会根据一系列的规则进行检查。如果违反了任何这些规则，程序都不能编译。在运行时，所有权系统的任何功能都不会减慢程序的运行。

原来 Rust 没有垃圾回收啊。  
所有权的主要目的就是管理堆数据。

### 所有权规则

>
    1. Rust 中的每一个值都有一个 所有者（owner）。
    2. 值在任一时刻有且只有一个所有者。
    3. 当所有者离开作用域，这个值将被丢弃。

#### String 类型
就字符串字面值来说，我们在编译时就知道其内容，所以文本被直接硬编码进最终的可执行文件中。这使得字符串字面值快速且高效。不过这些特性都只得益于字符串字面值的不可变性。

String 类型存储在堆上，可以存储编译时未知大小的文本。
```rust
{
    let mut s = String::from("hello");

    s.push_str(", world!"); // push_str() 在字符串后追加字面值

    println!("{s}"); // 将打印 `hello, world!`
}
```

> 当变量离开作用域，Rust 为我们调用一个特殊的函数。这个函数叫做 drop，在这里 String 的作者可以放置释放内存的代码。Rust 在结尾的`}`处自动调用 drop。


这段代码不能运行：
```rust
    let s1 = String::from("hello");
    let s2 = s1;

    println!("{s1}, world!");
```
`s1` 和 `s2` 都是栈上的指针，指向了堆上实际的数据。当 `s2` 和 `s1` 离开作用域，它们都会尝试释放相同的内存。这是一个叫做 二次释放（double free）的错误。

为了确保内存安全，在 `let s2 = s1`; 之后，Rust 认为 `s1` 不再有效，因此 Rust 不需要在 `s1` 离开作用域后清理任何东西。只有 `s2` 是有效的，当其离开作用域，它就释放自己的内存。

这里的赋值类似浅拷贝，但不是！这里叫做**移动(move)**。  

当你给一个已有的变量赋一个全新的值时，Rust 将会立即调用 drop 并释放原始值的内存。
```rust
    let mut s = String::from("hello");
    s = String::from("ahoy");

    println!("{s}, world!");
```

**只在栈上的数据赋值：拷贝**。

如果一个类型实现了 `Copy` trait，那么一个旧的变量在将其赋值给其他变量后仍然有效。  
Rust 不允许自身或其任何部分实现了 `Drop` trait 的类型使用 `Copy` trait。

### 所有权与函数

在调用函数传参时，变量的所有权进入了函数内部，函数外将不能再使用。  
可以通过返回值再带回所有权，但太麻烦了。可以使用**引用（references）**来解决这个问题。

```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，
    println!("{}", x);              // 所以在后面可继续使用 x

} // 这里，x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 没有特殊之处

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{some_string}");
} // 这里，some_string 移出作用域并调用 `drop` 方法。
  // 占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{some_integer}");
} // 这里，some_integer 移出作用域。没有特殊之处
```

## 引用与借用

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{s1}' is {len}.");
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```
`&`是引用，`*`是解引用。  
我们将创建一个引用的行为称为**借用（borrowing）**。

### 可变引用

借用来的引用变量是（默认）不允许修改的。  
除非显示的声明：
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

可变引用职能创建一个。创建了一个可变引用后，就不能再创建这个对象的引用，无论是可变的还是不可变的。（心声：类似加了写锁[:thinking_face:]）

> 这个限制的好处是 Rust 可以在编译时就避免数据竞争。数据竞争（data race）类似于竞态条件，它可由这三个行为造成：  
    - 两个或更多指针同时访问同一数据。  
    - 至少有一个指针被用来写入数据。  
    - 没有同步数据访问的机制。  

可以使用大括号来创建一个新的作用域，以允许拥有多个可变引用，只是不能**同时**拥有：
```rust
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    } // r1 在这里离开了作用域，所以我们完全可以创建一个新的引用

    let r2 = &mut s;
```

注意一个引用的作用域从声明的地方开始一直持续到最后一次使用为止。
```rust
    let mut s = String::from("hello");

    let r1 = &s; // 没问题
    let r2 = &s; // 没问题
    println!("{r1} and {r2}");
    // 此位置之后 r1 和 r2 不再使用

    let r3 = &mut s; // 没问题
    println!("{r3}");
```

## Slice 类型

### String Slice

```rust
    let s = String::from("hello world");

    let hello = &s[0..5];
    let hello = &s[..5];
    let world = &s[6..11];
    let world = &s[6..];
```
(心声： 看起来和 Golang 中的 Slice 类似 [:thinking_face:])  

“字符串 slice” 的类型声明写作 `&str`

```rust
// 返回空格分隔的字符串中的第一个单词
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear(); // 错误！

    println!("the first word is: {word}");
}
```

[:thinking_face:] Golang 中类似的问题是什么情况？


```go
func main() {
        s := "hello world"
        hello := s[:5]
        s = "" // 指向了新的字符串，不会修改原有字符串
        fmt.Println(fmt.Sprintf("fisrt word is %s", hello)) // 仍可正确输出
}

// 原始字符串通过引用计数和可达性来进行内存回收
```


**字符串字面值就是 Slice**

对于上面的示例，函数签名  
`fn first_word(s: &str) -> &str {`   
比  
`fn first_word(s: &String) -> &str {`  
更加通用。

### 其他类型的 slice
```rust
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];
assert_eq!(slice, &[2, 3]);
```
这个 slice 的类型是 `&[i32]`。

## 结构体

定义结构体
```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```

创建结构体
```rust
fn build_user(email: String, username: String) -> User {
    User {
        active: true,
        username, // 简化写法，像 JavaScript
        email,
        sign_in_count: 1,
    }
}
```

结构体的更新语法
```rust
fn main() {
    // --snip--

    let user2 = User {
        email: String::from("another@example.com"),
        ..user1
    };
}
```

(心声：更像 JavaScript 了， 注意这里是两个点号`..`)

使用结构体更新语法和`=`赋值一样，也发生了数据的移动。**总体上**说我们在创建 user2 后就不能再使用 user1 了，因为 user1 的 username 字段中的 String 被移到 user2 中。具体到字段来看，`user1.email`可以继续使用，因为未被移动。`active`和`sign_in_count`也未被移动，因为它们是实现`Copy` trait 的类型。

### 元组结构体

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
}
```

### 类单元结构体（unit-like structs）

类单元结构体常常在你想要在某个类型上实现 trait 但不需要在类型中存储数据的时候发挥作用。

(心声：类似 Golang 的空结构体？[:thinking_face:])

```rust
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}
```

### 结构体方法

在`impl`块中定义方法，方法的第一个参数是`self`。  
(心声： 又和 Python 有点像了。)

```rust
struct Rectangle {
    width: u32,
    height: u32,
}


impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle{
        width: 30,
        height: 50,
    };

    println!("The area of the rectangle is {}", rect1.area());
}
```

方法的参数`&self`实际是`self: &Self`的缩写。在一个 `impl` 块中，`Self` 类型是 `impl` 块的类型的别名。

方法名可以和字段名相同，比如可以为`Rectangle`添加一个`width()`方法。使用时，带括号时调用的方法，不带括号时则是值字段。

### 关联函数

在 `impl` 块中定义，但不以 `self` 为第一个参数的函数被称为**关联函数**，常被用作构造函数。我们已经使用了一个这样的函数：在 `String` 类型上定义的 `String::from` 函数。

### 多个 impl 块

每个结构体都允许拥有多个 `impl` 块。可以把方法分散在多个块中，但没必要。

## 枚举

定义枚举
```rust
enum IpAddrKind {
    V4,
    V6,
}
```

枚举类型可以嵌入值
```rust
    enum IpAddr {
        V4(u8, u8, u8, u8),
        V6(String),
    }

    let home = IpAddr::V4(127, 0, 0, 1);

    let loopback = IpAddr::V6(String::from("::1"));
```

可以内嵌多种类型:
```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

枚举上还可以定义方法：
```rust
    impl Message {
        fn call(&self) {
            // 在这里定义方法体
        }
    }

    let m = Message::Write(String::from("hello"));
    m.call();
```

(心声：和我熟悉的任何一门编程语言中的枚举都不一样，竟然可以嵌入值，还能定义方法？！[:thinking_face:])

## 未完待续

未完待续...

