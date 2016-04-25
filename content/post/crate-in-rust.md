+++
date = "2016-04-25T11:14:02+08:00"
description = ""
draft = false
tags = ["rust"]
title = "Rust中的包管理"
topics = ["Development"]

+++

Rust中使用Cargo工具进行包管理工作,默认通过cargo new创建出来的都是一个crate,相当于一个包.

<!--more-->

以前面一个用Rust编写的[计算器](../tdd-in-rust-1)为例,这个项目中,所有代码都是集中在一个文件中,我们将其进行分解,将不同模块放置到不同的文件中去.

每一个crate都包含一个lib.rs的文件,用于描述这个包需要导出的模块,通常情况下,我们可以通过执行命令:

```sh
cargo new crate_lib
```

来创建一个名为crate_lib的文件夹,打开包描述文件Cargo.toml文件:

```toml
[package]
name = "crate_lib"
version = "0.1.0"
authors = ["chenbin <chen.bin@xxx.com>"]
```

其中的name字段就是当前crate包的名称了.

打开该文件夹下的src目录,只有一个lib.rs文件,文件里边只有一个测试用例.

如果cargo new命令带了一个参数--bin的话,则会生成一个main.rs的文件,作为应用程序的入口代码.

回到当前的例子,查看Cargo.toml文件,可以看到,目前包名称为calc.

目前只有一个main.rs文件,根据各个功能,将其分解为几个文件token.rs, lexer.rs, parser.rs, parser.rs, interpreter.rs以及main.rs.为了描述这个包,我们手动创建lib.rs文件,在这个文件中描述各个模块的导出情况:


```cpp
// lib.rs
mod lexer;
mod token;
mod parser;
pub mod interpreter;
```

Rust中默认的包都是private的,只有标注了pub的模块才是public的.这一点和Rust的设计还是很统一的,如变量声明默认是不可变的,只有明确声明了mut的变量才是可变的.

拆分后的lexer.rs文件如下:

```cpp
// lexer.rs

// Rust通过use关键字调用token中的Token模块,这个地方的token就是lib.rs中描述的mod
use token::Token;


pub struct Lexer {
    text: String,
    pos: usize,
    current_char: Option<char>,
}

impl Lexer {
    // 所有方法默认都是private的,由于该构造方法需要在外部模块中调用,需要声明为pub
    pub fn new(text: String) -> Lexer {
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

    pub fn next_token(&mut self) -> Token {
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
                },
                '*' => {
                    self.advance();
                    return Token::MUL;
                },
                '/' => {
                    self.advance();
                    return Token::DIV;
                },
                '(' => {
                    self.advance();
                    return Token::LPAREN;
                },
                ')' => {
                    self.advance();
                    return Token::RPAREN;
                },
                _ => {}
            }

            panic!("Invalid Character");
        }
        Token::EOF
    }
}
```

其他几个文件与此类似,通过use关键字来调用当前模块中的其他文件,下面来查看main.rs文件:

```cpp
// main.rs

// extern crate表示调用calc这个包
extern crate calc;

use std::io;
use std::io::Write;

// 通过calc调用lib.rs中描述的Interpreter模块
use calc::interpreter::Interpreter;

fn main() {
    loop {
        let mut input = String::new();
        let _ = io::stdout().write(b">");
        let _ = io::stdout().flush();

        io::stdin().read_line(&mut input).unwrap();
        let mut interpreter = Interpreter::new(input.trim());
        println!("= {}", interpreter.interpret());
    }
}
```

main.rs中通过extern来声明需要调用的crate包,通过use来调用指定的模块.

这样就完成了模块的拆分工作,在Rust中通过crate包的方式来管理模块,还是非常灵活的.更多信息可以查看Rust文档中Crates and Modules这一章节.

本文代码可查看我的[Github](https://github.com/heychenbin/interpreter_tutorial/tree/master/rust/src).
