+++
date = "2016-04-20T09:46:25+08:00"
draft = false
title = "TDD in Rust"
image = "views.jpg"
tags = ["rust", "programming"]
topics = ["Development"]
+++

## Intro

Rust 是一门系统编程语言,目标是帮助程序员写出安全,快速的代码.本文将以TDD的方式使用Rust中创建一个计算器.

<!--more-->

## Cargo

Cargo是Rust中的包管理器,默认集成在安装包里边.通过使用Cargo可以方便管理Rust中的依赖,进行单元测试以及构建版本.

### 创建项目

首先通过Cargo创建一个项目calc:

```sh
Cargo create calc --bin
```

这会在当前目录下生成一个calc的目录,在该目录下的src文件夹下有一个main.rs,就是我们的主程序代码了.这个其中包括了一个简单的hello world程序,我们运行一下看看:

```sh
cd calc
Cargo run
```

输出:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running `target\debug\calc.exe`
Hello, world!
```

Cargo会自动编译当前项目的代码并生成exe程序执行.

## 计算器

下面以计算器为示例,介绍Rust的一些基本特性和用法.通常我学习一门新语言都会使用这么语言编写一些代码作为练习,现在我比较喜欢编写四则运算器来作为练习.

我希望达到的效果是实现四则运算,支持括号,加减乘除以及正负号等操作:

```
> 1 + 2 * (3-1)
= 5

> 9 +-+---+++---8
= 1
```

我以TDD的方式来实现这样一个程序.

### 测试用例

Rust一个很好的特性就是集成了单元测试工具,可以很方便我们做TDD编程,在我们的main.rs最后增加如下代码:

```cpp
#[cfg(test)]
mod tests {
    #[test]
    fn test_plus() {
        assert_eq!(1, 0);
    }
}
```

通过Cargo进行测试:

```
Cargo test
```

输出:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running D:\home\Projects\tmp\calc\target\debug\calc-a5ab18be1f87e4e3.exe

running 1 test
test tests::test_plus ... FAILED

failures:

---- tests::test_plus stdout ----
    thread 'tests::test_plus' panicked at 'assertion failed: `(left == right)` (left: `1`, right: `0`)', main.rs:13


failures:
    tests::test_plus

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured
```

可以看到,Cargo会自动编译代码并运行测试用例输出结果.

### 简单加法

下一步,我们开始实现计算器,首先最简单的测试用例:1 + 1 = 2.

```cpp
#[cfg(test)]
mod tests {
    fn calc(expr: &str) -> i32 {
        let mut interpreter = Interpreter::new(String::from(expr));
        interpreter.interpret()
    }

    #[test]
    fn test_plus() {
        assert_eq!(calc("1+1"), 2);
    }
}
```

首先,我们设计解析器入口,输入为一个计算表达式字符串,解释器包含一个方法interpret用于计算输入的表达式.

测试用例中,我们测试简单的 1+1=2.

下面可以开始设计代码了.为了方便解析,我们首先定义一系列Token表示解析表达式的类型信息:

```cpp
#[derive(Clone, Debug, Eq, PartialEq)]
enum Token {
    INTEGER(i32),
    PLUS,
    EOF,
}
```

在Rust中,enum和C/C++中的枚举类似,但是功能更加强大,可以为枚举添加更多信息,在匹配时可支持模式匹配,功能非常强大.

#[derive(Clone, Debug, Eq, PartialEq)] 这一段信息用于描述当前enum的一些附加属性,我们添加了Clone,Debug等几个属性,提示编译器为当前enum添加了复制,打印调试信息等功能.

第一个属性INTEGER(i32),表示数字,数字有一个属性为32位int,这里就是我们的解析出来的数字了.

下一步,定义我们的解析器:

```cpp
pub struct Interpreter {
    text: String,
    pos: usize,
    current_char: Option<char>,
}
```

Rust中使用struct来定义一个类型,当前的Interpreter包含三个属性,分别是输入字符,当前解析位置,和当前字符,前面两个属性类型都很好理解.最后一个类型为Option,这个Option在Rust中也是一个enum类型:

```cpp
pub enum Option<T> {
    None,
    Some(T)
}
```

源码中这样定义Option,可以看到他要么是一个None,要么是一个类型为T的值,在Rust中可以经常看到这样的表示方式,后续我们在看如何使用.

```cpp
impl Interpreter {
    // 这个方法就是实例化一个Interpreter类型的方法new,Rust中默认使用new
    // 方法来初始化一个类型.在这段的末尾,单独一个interpreter表示返回该值.这也
    // 是Rust的一个特性,最后一条语句不带分号结束为返回值.
    fn new(text: String) -> Interpreter {
        let mut interpreter = Interpreter {
            text: text,
            pos: 0,
            current_char: None,
        };
        if interpreter.text.len() > 0 {
            interpreter.current_char = Some(interpreter.text.as_bytes()[0] as char);
        }
        interpreter
    }

    fn advance(&mut self) {
        self.pos += 1;
        if self.pos > self.text.len()-1 {
            self.current_char = None;
        } else {
            // 这里使用Option中的Some()来表示
            self.current_char = Some(self.text.as_bytes()[self.pos] as char);
        }
    }

    fn next_token(&mut self) -> Token {
        while let Some(c) = self.current_char {
            if c.is_digit(10) {
                self.advance();
                // 使用枚举中的INTEGER(i32)来表示解析出来的值,后续通过匹配就能够直接获取这个值
                return Token::INTEGER(c.to_digit(10).unwrap() as i32);
            }

            match c {
                '+' => {
                    self.advance();
                    return Token::PLUS;
                },
                // Rust使用match来进行匹配,有点类似于C/C++中的switch,只是功能更强大.match默认
                // 情况下不会像C/C++一样匹配后不break则会继续运行下一个分支,默认只会匹配一个分
                // 支然后自动break,default分支使用 _ => 来表示.
                _ => panic!("Invalid char"),
            }
        }
        Token::EOF
    }

    fn expr(&mut self) -> i32 {
        let lhs = match self.next_token() {
            Token::INTEGER(i) => i,
            _ => panic!("Invalid num"),

        };

        let _ = self.next_token();

        let rhs = match self.next_token() {
            Token::INTEGER(i) => i,
            _ => panic!("Invalid num"),
        };

        lhs + rhs
    }

    fn interpret(&mut self) -> i32 {
        self.expr()
    }
}
```

实现以上代码后,再次在命令行中执行:

```sh
cargo test
```

输出为:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-cf9c4ad57c782c3c.exe

running 1 test
test tests::test_plus ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

可以看到测试通过,我们第一个测试用例1+1=2也就测试成功了.

### 简单减法

下一步,我们实现简单减法,也就是一位数的减法,首先更新测试用例:

```cpp
#[test]
fn test_minus() {
    assert_eq!(calc("1-1"), 0);
    assert_eq!(calc("1-2"), -1);
    assert_eq!(calc("2-1"), 1);
}
```

为了实现减法,需要增加一个Token类型:

```cpp
enum Token {
    INTEGER(i32),
    PLUS,
    MINUS,
    EOF,
}
```

重构一下Interpreter代码:

```cpp
fn next_token(&mut self) -> Token {
    // 这里使用while let语句来解析current_char中的Option属性,Rust中语法非常灵活,有点脚本语言的意思
    while let Some(c) = self.current_char {
        if c.is_digit(10) {
            self.advance();
            return Token::INTEGER(c.to_digit(10).unwrap() as i32);
        }
        match c {
            '+' => {
                self.advance();
                return Token::PLUS;
            },
            // 新增解析减号的方法
            '-' => {
                self.advance();
                return Token::MINUS;
            }
            _ => panic!("Invalid char"),
        }
    }
    Token::EOF
}

// 新增一个factor方法专门用于解析数字类型
fn factor(&mut self) -> i32 {
    match self.next_token() {
        Token::INTEGER(i) => i,
        _ => panic!("Invalid factor"),
    }
}

fn expr(&mut self) -> i32 {
    let lhs = self.factor();
    let op = self.next_token();
    match op {
        // 新增减号操作
        Token::PLUS => lhs + self.factor(),
        Token::MINUS => lhs - self.factor(),
        _ => panic!("Invalid op"),
    }
}
```

再次执行cargo test:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 2 tests
test tests::test_minus ... ok
test tests::test_plus ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

测试通过!


### 忽略空格

现在我们的计算器代码是不支持解析空格的,当遇到空格时,解析器需要自动略过,首先更新测试用例:

```cpp
    #[test]
    fn test_plus() {
        check("1+1", 2);
        check("1 + 1", 2);
    }

    #[test]
    fn test_minus() {
        assert_eq!(calc("1 - 1"), 0);
        assert_eq!(calc("1- 2"), -1);
        assert_eq!(calc("2  - 1"), 1);
    }
```

在Interpreter中增加一个方法skip_whitespace:

```cpp
fn skip_whitespace(&mut self) {
    while let Some(c) = self.current_char {
        if c.is_whitespace() {
            self.advance();
        } else {
            break;
        }
    }
}
```

修改next_token()函数:

```cpp
fn next_token(&mut self) -> Token {
    while let Some(c) = self.current_char {
        // 忽略空格
        if c.is_whitespace() {
            self.skip_whitespace();
            continue;
        }

        if c.is_digit(10) {
            self.advance();
            return Token::INTEGER(c.to_digit(10).unwrap() as i32);
        }
        match c {
            '+' => {
                self.advance();
                return Token::PLUS;
            },
            '-' => {
                self.advance();
                return Token::MINUS;
            }
            _ => panic!("Invalid char"),
        }
    }
    Token::EOF
}
```

再次执行cargo test:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 2 tests
test tests::test_minus ... ok
test tests::test_plus ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

测试通过!

### 简单乘除法

有了上面的框架,其实实现简单的乘除法已经非常方便了,首先还是增加测试用例:

```cpp
#[test]
fn test_mul() {
    assert_eq!(calc("1 * 1"), 1);
    assert_eq!(calc("9 * 8 "), 72);
    // assert_eq!(calc("13 * 11"), 143);
}
#[test]
fn test_div() {
    assert_eq!(calc("9/3"), 3);
    // assert_eq!(calc("144 / 12 "), 12);
    // assert_eq!(calc("90 / 10"), 9);
}
```

新增两个Token类型:

```cpp
enum Token {
    INTEGER(i32),
    PLUS,
    MINUS,
    MUL,
    DIV,
    EOF,
}
```

在next_token方法中新增解析*,/符号的操作:

```cpp
fn next_token(&mut self) -> Token {
    while let Some(c) = self.current_char {
        if c.is_whitespace() {
            self.skip_whitespace();
            continue;
        }
        if c.is_digit(10) {
            self.advance();
            return Token::INTEGER(c.to_digit(10).unwrap() as i32);
        }
        match c {
            '+' => {
                self.advance();
                return Token::PLUS;
            },
            '-' => {
                self.advance();
                return Token::MINUS;
            }
            '*' => {
                self.advance();
                return Token::MUL;
            }
            '/' => {
                self.advance();
                return Token::DIV;
            }
            _ => panic!("Invalid char"),
        }
    }
    Token::EOF
}
```

修改计算乘除操作:

```cpp
fn expr(&mut self) -> i32 {
    let lhs = self.factor();
    let op = self.next_token();
    match op {
        Token::PLUS => lhs + self.factor(),
        Token::MINUS => lhs - self.factor(),
        Token::MUL => lhs * self.factor(),
        Token::DIV => lhs / self.factor(),
        _ => panic!("Invalid op"),
    }
}
```

运行测试用例cargo test:

```cpp
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 4 tests
test tests::test_div ... ok
test tests::test_mul ... ok
test tests::test_plus ... ok
test tests::test_minus ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured
```

测试通过!

### 多位数加减乘除

目前的代码只能够支持一位数的加减乘除操作,如何实现多位数操作呢?首先我们还是先更新测试用例:

```cpp
fn check(expr: &str, expected_result: i32) {
    assert_eq!(calc(expr), expected_result);
}

#[test]
fn test_plus() {
    check("19  + 23", 42);
}

#[test]
fn test_minus() {
    check("23 - 9", 14);
}

#[test]
fn test_mul() {
    check("13 * 11", 143);
}

#[test]
fn test_div() {
    check("144 / 12 ", 12);
    check("90 / 10", 9);
}
```

一些重复的测试这里就不再这里显示了.


可以想到,需要修改解析函数来识别多位数,新增一个方法专门用于解析数字:

```cpp
fn integer(&mut self) -> i32 {
    let mut result = String::new();
    while let Some(c) = self.current_char {
        if c.is_digit(10) {
            result.push(c);
            self.advance();
        } else {
            break;
        }
    }
    result.parse::<i32>().unwrap()
}
```

修改解析函数:

```cpp
fn next_token(&mut self) -> Token {
    while let Some(c) = self.current_char {
        if c.is_whitespace() {
            self.skip_whitespace();
            continue;
        }

        // 通过调用integer函数来解析数字类型.
        if c.is_digit(10) {
            return Token::INTEGER(self.integer());
        }
        match c {
            '+' => {
                self.advance();
                return Token::PLUS;
            },
            '-' => {
                self.advance();
                return Token::MINUS;
            }
            '*' => {
                self.advance();
                return Token::MUL;
            }
            '/' => {
                self.advance();
                return Token::DIV;
            }
            _ => panic!("Invalid char"),
        }
    }
    Token::EOF
}
```

再次运行测试用例cargo test:

```cpp
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 4 tests
test tests::test_minus ... ok
test tests::test_div ... ok
test tests::test_mul ... ok
test tests::test_plus ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured
```

### 连续加减乘除运算

目前代码只支持多位数加减乘除多位数,下一步我们实现多位数的连续加减乘除操作.按照惯例,更新测试用例:

```cpp
#[test]
fn test_mul_plus() {
    check("1+1+1", 3);
    check("1+2+2", 5);
    check("19  + 23+1", 43);
}

#[test]
fn test_mul_minus() {
    check("11-9-9+ 2", -5);
}

#[test]
fn test_mul_mul() {
    check("2 * 3 *4", 24);
}

#[test]
fn test_mul_div() {
    check("90 / 10 * 2", 18);
    check("9 / 3/ 3", 1);
}
```

从测试用例可以看到,我们只要遇到了运算符就继续进行运算:将当前结果继续应用到下一位数字中,其实一个循环操作就能够解决了,因此只需要修改expr方法:

```cpp
fn expr(&mut self) -> i32 {
    let mut result = self.factor();
    loop {
        match self.next_token() {
            Token::PLUS => {
                result += self.factor();
            },
            Token::MINUS => {
                result -= self.factor();
            },
            Token::MUL => {
                result *= self.factor();
            },
            Token::DIV => {
                result /= self.factor();
            },
            _ => break,
        }
    }
    result
}
```

很简单,运行一下cargo test:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 8 tests
test tests::test_div ... ok
test tests::test_mul_div ... ok
test tests::test_mul ... ok
test tests::test_minus ... ok
test tests::test_mul_minus ... ok
test tests::test_mul_mul ... ok
test tests::test_mul_plus ... ok
test tests::test_plus ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured
```

### 重构代码

目前代码有一定规模了,我们所有代码都集中在解释器Interpreter代码中,为了更好的维护代码,限制我们队代码进行重构,将解析和解释进行分离:将解析抽象为Lexer对象,将计算放在Interpreter对象中:

```cpp
#[derive(Clone, Debug, Eq, PartialEq)]
enum Token {
    INTEGER(i32),
    PLUS,
    MINUS,
    MUL,
    DIV,
    EOF,
}

pub struct Lexer {
    text: String,
    pos: usize,
    current_char: Option<char>,
}

impl Lexer {
    fn new(text: String) -> Lexer {
        let mut lexer = Lexer {
            text: text,
            pos: 0,
            current_char: None,
        };
        if lexer.text.len() > 0 {
            lexer.current_char = Some(lexer.text.as_bytes()[0] as char);
        }
        lexer
    }

    fn skip_whitespace(&mut self) {
        while let Some(c) = self.current_char {
            if c.is_whitespace() {
                self.advance();
            } else {
                break;
            }
        }
    }

    fn advance(&mut self) {
        self.pos += 1;
        if self.pos > self.text.len()-1 {
            self.current_char = None;
        } else {
            self.current_char = Some(self.text.as_bytes()[self.pos] as char);
        }
    }

    fn integer(&mut self) -> i32 {
        let mut result = String::new();
        while let Some(c) = self.current_char {
            if c.is_digit(10) {
                result.push(c);
                self.advance();
            } else {
                break;
            }
        }
        result.parse::<i32>().unwrap()
    }

    fn next_token(&mut self) -> Token {
        while let Some(c) = self.current_char {
            if c.is_whitespace() {
                self.skip_whitespace();
                continue;
            }
            if c.is_digit(10) {
                return Token::INTEGER(self.integer());
            }
            match c {
                '+' => {
                    self.advance();
                    return Token::PLUS;
                },
                '-' => {
                    self.advance();
                    return Token::MINUS;
                }
                '*' => {
                    self.advance();
                    return Token::MUL;
                }
                '/' => {
                    self.advance();
                    return Token::DIV;
                }
                _ => panic!("Invalid char"),
            }
        }
        Token::EOF
    }
}

pub struct Interpreter {
    lexer: Lexer,
}

impl Interpreter {
    fn new(lexer: Lexer) -> Interpreter {
        Interpreter {
            lexer: lexer,
        }
    }

    fn factor(&mut self) -> i32 {
        match self.lexer.next_token() {
            Token::INTEGER(i) => i,
            _ => panic!("Invalid factor"),
        }
    }

    fn expr(&mut self) -> i32 {
        let mut result = self.factor();
        loop {
            match self.lexer.next_token() {
                Token::PLUS => {
                    result += self.factor();
                },
                Token::MINUS => {
                    result -= self.factor();
                },
                Token::MUL => {
                    result *= self.factor();
                },
                Token::DIV => {
                    result /= self.factor();
                },
                _ => break,
            }
        }
        result
    }
}
```

由于有测试用例做支撑,重构起来其实非常有信心,运行一下cargo test:

```cpp
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 8 tests
test tests::test_mul ... ok
test tests::test_div ... ok
test tests::test_mul_div ... ok
test tests::test_minus ... ok
test tests::test_mul_minus ... ok
test tests::test_mul_mul ... ok
test tests::test_mul_plus ... ok
test tests::test_plus ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured
```

测试通过!


### 支持运算符优先级

按照惯例,添加测试用例:

```cpp
#[test]
fn test_multi_ops() {
    check("1 + 2*3", 7);
    check("8 * 3 + 1", 25);
    check("8 * 3 + 2*5", 34);
}
```

这一步是计算器的一个难点,优先级一共有两个级别:

    1. * /
    2. + -

乘除的优先级是高于加减的,因此我们在解析时,需要优先解析乘除,再解析加减,修改代码如下:

```cpp
fn consume(&mut self, token: Token) {
    if Some(token) == self.current_token {
        self.current_token = Some(self.lexer.next_token());
    } else {
        panic!("Invalid consume");
    }
}

fn factor(&mut self) -> i32 {
    if let Some(token) = self.current_token.clone() {
        match token {
            Token::INTEGER(i) => {
                self.consume(Token::INTEGER(i));
                return i;
            },
            _ => panic!("Invalid token"),
        }
    }
    panic!("Invalid factor");
}

fn term(&mut self) -> i32 {
    let mut result = self.factor();
    while let Some(token) = self.current_token.clone() {
        match token {
            Token::MUL => {
                self.consume(Token::MUL);
                result *= self.factor();
            },
            Token::DIV => {
                self.consume(Token::DIV);
                result /= self.factor();
            },
            _ => break,
        }
    }
    result
}

fn expr(&mut self) -> i32 {
    let mut result = self.term();
    while let Some(token) = self.current_token.clone() {
        match token {
            Token::PLUS => {
                self.consume(Token::PLUS);
                result += self.term();
            },
            Token::MINUS => {
                self.consume(Token::MINUS);
                result -= self.term();
            },
            _ => break,
        }
    }
    result
}
```

    0. factor => int
    1. term => * /
    2. expr => + -

代码中我们通过这三级解析方式来处理优先级的问题,这样就能够正确计算四则运算了.

运行测试用例:

```
   Compiling calc v0.1.0 (file:///D:/Home/Projects/tmp/calc)
     Running target\debug\calc-d41ffd04c8d0c698.exe

running 9 tests
test tests::test_div ... ok
test tests::test_mul ... ok
test tests::test_minus ... ok
test tests::test_mul_div ... ok
test tests::test_mul_minus ... ok
test tests::test_mul_mul ... ok
test tests::test_mul_plus ... ok
test tests::test_multi_ops ... ok
test tests::test_plus ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured
```

测试通过!

通过以上讲解,我们就把基本的四则运算在Rust中实现了,Rust自带的单元测试模块能够很好的保证代码的正确性,我们也可以使用这些测试模块来进行自测.

本项目中还对代码进行了一定的优化,使用AST(抽象语法树)来描述运算,增加括号,一元运算符等,这里就不再详细描述了,可以查看本项目代码进行深入了解,谢谢!

