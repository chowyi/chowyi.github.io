title: Hello Rust
date: 2025-07-12 18:27:35
tags:
- Rust
categories:
- Program
---

近来网上冲浪时，在 Github trending 上越来越多的看到一些 Rust 项目，也常看到一些项目的简介写着 “这是一款使用 Rust 开发的 xxx”。这令我不禁好奇，使用某种编程语言开发也能成为一个软件值得一提的特点吗？不过这确实引起了我的兴趣，简单的搜索一下，Rust 以其高性能和内存安全著称，软件作者通过强调使用 Rust，传达其软件在性能和安全性上的优势。

这两年的工作陷入了理不清的需求和讨论不完的产品方案之中，很久没有那种“长脑子”的感觉了。现在有点时间不如来学学 Rust，就算是赶下时髦（也许没赶上）？

## 边学边记

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

### Option

`Option` 是标准库中定义的枚举，可以用来表示空值。

```rust
enum Option<T> {
    None,
    Some(T),
}
```

(心声：暂时对 Rust 中的枚举还不是很理解，之后再来补充吧！ [:thinking_face:])

## match

分支的返回值即整个 `match` 的返回值。

在 Rust 中 match 必须显式的匹配所有的可能分支。`other` 类似于 Golang 中的 `default`，`_` 表示忽略绑定的值。

`match` 本身比较简单，但要注意理解**绑定值的模式**和匹配 `Option<T>` 的用法。  

## if let 和 let else

可以认为是 `match` 的语法糖，用来写出更简洁的代码。（心声：但我认为并不好理解。也许熟悉后能体会到好处吧。）

## 包和Crate

> 包（package）是提供一系列功能的一个或者多个 crate 的捆绑。一个包会包含一个 Cargo.toml 文件。

> crate 是 Rust 在编译时最小的代码单位。crate 有两种形式：二进制 crate 和库 crate。

模块用来给代码分组，在 crate 中可以层层划分模块和子模块。

书中这部分篇幅较大，举了很多例子，要耐心学习，要注意理解：
 1. 引用模块的路径规则（绝对路径、相对路径、super）
 2. 模块、函数、结构体字段、枚举的私有性
 3. use 的用法和惯用规则  
    1. 使用 use 将函数引入作用域，而不是函数本身。
    2. 使用 use 引入结构体、枚举和其他项时，习惯是指定它们的完整路径。
    3. pub use 重导出
    4. 使用嵌套路径合并相同路径前缀的 use
    5. glob 运算符，即 `use std:collections::*` 这样的用法（小心使用！）。


> 注意你只需在模块树中的某处使用一次 mod 声明就可以加载这个文件。一旦编译器知道了这个文件是项目的一部分（并且通过 mod 语句的位置知道了代码在模块树中的位置），项目中的其他文件应该使用其所声明的位置的路径来引用那个文件的代码，这在“引用模块项目的路径”部分有讲到。换句话说，mod 不是你可能会在其他编程语言中看到的 “include” 操作。

(心声：怎么理解 mod 不同于其他语言中的“include”呢？ [:thinking_face:])

## 集合 collections

> 集合指向的数据是储存在堆上的，这意味着数据的数量不必在编译时就已知，并且还可以随着程序的运行增长或缩小。

### 向量 vector

> 它在内存中彼此相邻地排列所有的值。vector 只能储存相同类型的值。

使用索引或`get`方法获取 vector 中的值，超出索引范围时，前者会 panic，后者会返回 None 枚举。  
```rust
    let v = vec![1, 2, 3, 4, 5];

    let third: &i32 = &v[2];
    println!("The third element is {third}");

    let third: Option<&i32> = v.get(2);
    match third {
        Some(third) => println!("The third element is {third}"),
        None => println!("There is no third element."),
    }
```

当 vector 中的某一个元素的引用被借出后，整个 vector 都将不能被改变。（底层原因是因为： vector 扩容时，可能会重新分配内存。）
```rust
    let mut v = vec![1, 2, 3, 4, 5];

    let first = &v[0];

    v.push(6);

    println!("The first element is: {first}");
```

使用 for in 可以遍历或修改 vector 中的元素，但不能在循环中插入或删除元素，因为 for 本身获取了 vector 的引用。
```rust
    let mut v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;
    }
```

vector 中的元素必须是同一类型的，但使用枚举可以绕过这一限制。  
(心声：那怎么分配内存啊？ [:thinking_face:])
```rust
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }

    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];
```

当 vector 被丢弃时，所有其内容也会被丢弃。

(心声：vector 和 slice 有什么区别？ [:thinking_face:])

### 字符串 string

slice `str` 是 Rust 核心语言中的类型，`String`是 Rust 标准库中定义的类型。  
`String`是对字节 vector 的特殊封装。

以下两种方式都可以创建`String`，效果完全一样：
```rust
let s1 = String::from("hello");
let s2 = "hello".to_string();
```

拼接字符串的几种方式：
```rust
// push_str
let mut s = String::from("foo");
s.push_str("bar");

// push 单个字符
let mut s = String::from("lo");
s.push('l');

// 加号运算符
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; // 注意 s1 被移动了，不能继续使用

// 加号运算符的函数签名
fn add(self, s: &str) -> String {
    // ...
}

// format! 宏
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");
let s = format!("{s1}-{s2}-{s3}");
```

`String` 不能使用索引访问其中的字符。  
因为 `String` 是 `Vec<u8>` 的封装，按字节存储 UTF-8 编码的数据。UTF-8 是一种可变长编码。使用索引不确保能访问到有效的字符，所以 Rust 不允许这样做。并且`String`也不能保证索引操作预期的 O(1) 时间。

`str` slice 允许使用索引访问其中的字节，但如果访问的索引截断了字符的字节，则会 panic。
```rust
let hello = "Здравствуйте"; // 这些字母都是两个字节长度

let s1 = &hello[0..4]; // 前两个字符
let s2 = &hello[0..1]; // panic
```

遍历字符串
```rust
fn main() {
    // 按字符遍历
    for c in "Зд".chars() {
        println!("{c}");
    }

    // 按字节遍历
    for b in "Зд".bytes() {
        println!("{b}");
    }

    // String 支持同样的操作
    let s = String::from("Зд");
    for b in s.chars() {
        println!("{b}");
    }

    for b in s.bytes() {
        println!("{b}");
    }
}

```

### hash map

> 所有的键必须是相同类型，值也必须都是相同类型。

用法：
```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

for (key, value) in &scores {
    println!("{key}: {value}");
}

// insert if NOT exists
scores.entry(String::from("Blue")).or_insert(50);
```

默认使用一种叫做 SipHash 的哈希函数，这是一种安全但不是最快的算法，可以自己指定哈希函数。

## 错误处理

类似于 Golang 中的函数的返回值经常会有一个 error，Rust 使用 `Result` 枚举来返回成功的值或 error。

可以使用 match 来匹配不同的错误。

使用 expect 来快捷处理 panic
```rust
use std::fs::File;

fn main() {
    let greeting_file = File::open("hello.txt")
        .expect("hello.txt should be included in this project");
}
```

使用 ? 运算符来快捷的传播错误：
```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username_file = File::open("hello.txt")?;
    let mut username = String::new();
    username_file.read_to_string(&mut username)?;
    Ok(username)
}
```

还可以链式调用：
```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();

    File::open("hello.txt")?.read_to_string(&mut username)?;

    Ok(username)
}
```
有点像 JavaScript。


> `?` 运算符只能被用于返回值与 `?` 作用的值相兼容的函数。  
>  `?` 也可用于 `Option<T>` 值。



> 注意你可以在返回 Result 的函数中对 Result 使用 ? 运算符，可以在返回 Option 的函数中对 Option 使用 ? 运算符，但是不可以混合搭配。? 运算符不会自动将 Result 转化为 Option，反之亦然；在这些情况下，可以使用类似 Result 的 ok 方法或者 Option 的 ok_or 方法来显式转换。

## 泛型

函数中的泛型
```rust
// 注意这里必须限制 T 实现了 std::cmp::PartialOrd 函数体中才能比大小
fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

结构体中的泛型
```rust
struct Point<T> {
    x: T,
    y: T,
}

// 结构体方法的泛型
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

枚举中的泛型：
```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

使用泛型不会带来运行时的消耗。在编译时编译器会生成针对具体类型的代码，这一过程叫做泛型代码的**单态化（monomorphization）**。

## trait

trait 翻译过来时“特征”的意思，类似于其他语言中的 interface。

trait 中可以定义默认方法，默认方法还能调用暂未实现的其他方法（这点想在 Golang 实现就比较复杂了）。
```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

### trait 作为参数

两种写法效果一样，下面一种被称为  trait bound 语法。
```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// trait bound 语法
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

可以指定多个 trait bound
```rust
pub fn notify(item: &(impl Summary + Display)) {
}

pub fn notify<T: Summary + Display>(item: &T) {    
}
```

为了避免函数签名太长难以阅读，可以使用 `where` 从句：
```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
}
```

在返回值中使用 trait。返回单一的类型是没问题，但因为编译器的限制，不能返回多个类型。
```rust
fn returns_summarizable() -> impl Summary {
    SocialPost {
        // ...
    }
}
```

使用 trait bound 可以有条件地只为那些实现了特定 trait 的类型实现方法。

## 生命周期
编译器无法判断出返回的引用的生命周期，就需要要显示的添加注解。

```rust
// 编译报错
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}

// 添加了生命周期注解
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

生命周期省略的三条规则：
> 1. 编译器为每一个引用参数都分配一个生命周期参数。
> 2. 如果只有一个输入生命周期参数，那么将它赋予给所有输出生命周期参数
> 3. 如果方法有多个输入生命周期参数并且其中一个参数是 &self 或 &mut self，说明这是个方法，那么所有输出生命周期参数被赋予 self 的生命周期。

## 自动化测试

(先跳过这一章，优先学习语言特性)

## 构建命令行程序

跟着书中的内容动手敲一下代码，有很多暂时不理解为什么这样写的地方。主要是对函数的定义，参数和返回值什么时候用引用，什么时候不用一时反应不过来。

## 闭包

闭包就是可以捕获环境中变量的“匿名函数”。例如，这个例子中，变量`x`被捕获在了闭包`add_x`中。
```rust
let x = 5;
let add_x = |y| y + x;  // 闭包捕获了外部变量 x
println!("{}", add_x(3)); // 输出 8
```

闭包的语法：
```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }  // 函数
let add_one_v2 = |x: u32| -> u32 { x + 1 }; // 带有类型注解的完整定义的闭包
let add_one_v3 = |x|             { x + 1 }; // 省略了类型注解的闭包
let add_one_v4 = |x|               x + 1  ; // 省略了花括号
```

闭包中的类型注解通常是可以省略的，编译器可以自行推断。

使用`move`来强制闭包获取它使用值的所有权。

闭包有3种`Fn`trait，编译器会根据闭包的具体逻辑在自动的实现一个、两个或全部 trait。

> 1. `FnOnce` 适用于只能被调用一次的闭包。所有闭包至少都实现了这个 trait，因为所有闭包都能被调用。一个会将捕获的值从闭包体中移出的闭包只会实现 FnOnce trait，而不会实现其他 Fn 相关的 trait，因为它只能被调用一次。
> 2. `FnMut` 适用于不会将捕获的值移出闭包体，但可能会修改捕获值的闭包。这类闭包可以被调用多次。
> 3. `Fn` 适用于既不将捕获的值移出闭包体，也不修改捕获值的闭包，同时也包括不从环境中捕获任何值的闭包。这类闭包可以被多次调用而不会改变其环境，这在会多次并发调用闭包的场景中十分重要。

## 迭代器

实现了 `Iterator` trait 的对象就是使用 `.iter()` 来获取它的迭代器。

`iter()`方法生成的是不可变引用的迭代器。  
`iter_mut()`方法生成的是可变引用的迭代器。  
`into_iter()`方法生成的是可以获取所有权的迭代器。  

迭代器可以链式调用，且是惰性的，直到调用消费适配器方法。
```rust
fn main() {
    let v1: Vec<i32> = vec![1, 2, 3];
    let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
    assert_eq!(v2, vec![2, 3, 4]);
}
```

`map()`接收实现了`FnMut`trait的函数或闭包。上面的map中的闭包也可以改成函数。（试着写一个 addOne 函数，作为初学者我没写出来）。  
想想参数为什么是这样的？
```rust
fn add_one(x: &i32) -> i32 {
    x + 1
}
```
