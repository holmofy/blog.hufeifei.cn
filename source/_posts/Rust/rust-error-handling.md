---
title: Rust错误处理
date: 2024-07-02
categories: Rust
tags: 
- Rust
---

Rust的错误处理机制和其他语言有很大的不同。

在C++、C#、Java、Javascript、Python等语言中，通常使用`throw`抛出异常或者返回成功的值。外部调用的地方使用`try/catch`进行捕获，除了[C++没有`finally`](https://en.cppreference.com/w/cpp/language/try)关键字外，像C#、Python、Java、Javascript都有基本一致的异常处理逻辑。像Java有三类异常：不可恢复的Error(如OutOfMemoryError、StackOverflowError)、受检异常(如IOException)、运行时异常(如NullPointerException)。特别是运行时异常由于隐式传递，运行在线上服务器经常出现令人头疼的问题。

C语言没有直接提供错误处理机制，通常返回`-1`或`NULL`以及全局错误码[`errno`](https://en.cppreference.com/w/cpp/error/errno)来做错误处理，可读性非常差。

这些语言还支持返回`null`，javascript甚至还有一个`undefined`，这导致大量的空指针异常。编程习惯好的可能会用运行时断言，但是性能上会有所损耗。

golang虽然不使用`throw`异常机制，但是得函数得有[两个返回值](https://go.dev/blog/error-handling-and-go)，而且还得用`if err != nil`语句判断是否存在错误，使用体验非常不好。

Rust的异常处理独辟蹊径用[`Result`](https://doc.rust-lang.org/std/result/)和[`Option`](https://doc.rust-lang.org/std/option/)这两个枚举来解决这些问题。

## Result枚举

`Result`用于返回结果和传递错误，它是个枚举，包含两种状态：

```rust
enum Result<T, E> {
   Ok(T),   // 表示调用成功的返回值
   Err(E),  // 表示调用出错的错误值
}
```

Rust官方文档中就有一个用例：

```rust
#[derive(Debug)]
enum Version { Version1, Version2 }

fn parse_version(header: &[u8]) -> Result<Version, &'static str> {
    match header.get(0) {
        None => Err("invalid header length"),
        Some(&1) => Ok(Version::Version1),
        Some(&2) => Ok(Version::Version2),
        Some(_) => Err("invalid version"),
    }
}

fn main(){
    let version = parse_version(&[1, 2, 3, 4]);
    match version {
        Ok(v) => println!("working with version: {v:?}"),
        Err(e) => println!("error parsing header: {e:?}"),
    }
}
```

针对`Result`使用`match`语句进行条件匹配。

## 忽略错误

上面的例子对于初学者可能有点懵逼，先从最简单的场景开始，我们可以忽略错误。这听起来很不靠谱，但确实有几个合法的用例：

* 我们正在对代码进行原型设计，不想花时间在错误处理上。
* 我们确信不会发生错误。
* 假设我们正在读取一个我们非常确定会存在的文件。

假设我们正在读取一个我们非常确定会存在的文件：

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./Cargo.toml").unwrap();
  println!("{}", content)
}
```

尽管我们知道该文件一定会存在，但是编译器无法知道这一点。因此，我们可以用`unwrap`告诉编译器信任我们并返回其中的值。

如果`read_to_string`函数返回`Ok()`值，则将获取`Ok`变量的内容并将其分配给`content`变量。如果它返回错误，它将[“panic”](https://doc.rust-lang.org/std/macro.panic.html)。Panic是不可恢复的错误，要么终止程序，要么退出当前线程。

请注意，`unwrap`在相当多的 Rust 示例中使用它来跳过错误处理。这主要是为了方便起见，在实际代码中不应该使用。

## 终止程序

某些错误无法处理或恢复。在这些情况下，最好通过终止程序来快速失败。

在上面的例子中：我们正在读取一个我们肯定会存在的文件。如果对于这个程序来说，该文件绝对重要，没有它就无法正常工作。如果由于某种原因，此文件不存在，最好终止该程序。

可以像上面`unwrap`一样用`expect`: 这两个方法效果相同，区别在于`expect`可以添加额外的错误信息。

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./Cargo.toml").expect("Can't read Cargo.toml");
  println!("{}", content)
}
```

可以查看官方文档中[!panic](https://doc.rust-lang.org/std/macro.panic.html)的介绍。

## 使用缺省值

在某些情况下，您可以通过提供默认值来处理错误。

例如，假设我们正在编写一个服务器，它侦听的端口可以使用`PORT`环境变量进行配置。如果未设置`PORT`环境变量，则访问该值将导致错误。但是我们可以通过提供默认缺省值来轻松处理它。

```rust
use std::env;

fn main() {
  let port = env::var("PORT").unwrap_or("3000".to_string());
  println!("{}", port);
}
```

`unwrap_or`函数允许我们提供默认值。

还有[`unwrap_or_else`](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or_else), [`unwrap_or_default`](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or_default)用另外的方法也可以提供默认值。

## 向上传递错误

当没有足够的上下文来处理错误时，可以冒泡（传递）错误到调用方函数。

下面是一个使用 Web 服务获取当前年份的人为示例：

```rust
use std::collections::HashMap;

fn main() {
  match get_current_date() {
    Ok(date) => println!("We've time travelled to {}!!", date),
    Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
  }
}

fn get_current_date() -> Result<String, reqwest::Error> {
  let url = "https://postman-echo.com/time/object";
  let result = reqwest::blocking::get(url);

  let response = match result {
    Ok(res) => res,
    Err(err) => return Err(err),
  };

  let body = response.json::<HashMap<String, i32>>();

  let json = match body {
    Ok(json) => json,
    Err(err) => return Err(err),
  };

  let date = json["years"].to_string();

  Ok(date)
}
```

在`get_current_date`中调用了两个函数：`get`发送请求和`json`解析响应体。

由于没有足够的上下文去处理错误，`get_current_date`直接把传递给了`main`函数。

用`match`语句匹配内部错误，导致代码噪音太多，Rust提供了[`?`操作符](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html#a-shortcut-for-propagating-errors-the--operator)的语法糖来解决这个问题，重写的代码如下：

```rust
use std::collections::HashMap;

fn main() {
  match get_current_date() {
    Ok(date) => println!("We've time travelled to {}!!", date),
    Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
  }
}

fn get_current_date() -> Result<String, reqwest::Error> {
  let url = "https://postman-echo.com/time/object";
  let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;
  let date = res["years"].to_string();

  Ok(date)
}
```

可以看到代码干净了很多。

`?`这个操作符和`unwrap`很相似，但是它不会panic，而是把错误传递给上一级调用函数。

不过需要注意`?`只能用在返回值为`Result`或`Option`类型的函数上。

## 向上传递多种类型的错误

在上面的例子中，`get`和`json`返回的都是`reqwest::Error`类型的错误，我们可以用`?`操作符进行传递。但是如果我们调用了另外的函数返回不同类型的错误会怎么样呢？

```diff rust
+ use chrono::NaiveDate;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
      Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
    }
  }

  fn get_current_date() -> Result<String, reqwest::Error> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;
-   let date = res["years"].to_string();
+   let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
+   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
+   let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

上面经过修改后的代码将会编译失败，因为调用`parse_from_str`返回的是`chrono::format::ParseError`而不是`reqwest::Error`。

我们可以对这个错误进行包装：

```diff rust
  use chrono::NaiveDate;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
      Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
    }
  }

- fn get_current_date() -> Result<String, reqwest::Error> {
+ fn get_current_date() -> Result<String, Box<dyn std::error::Error>> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
    let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

当我们想处理多种错误时，使用`Box<dyn std::error::Error>`作为错误类型会非常方便。

* 这里的[`std::error::Error`](https://doc.rust-lang.org/stable/std/error/trait.Error.html)是一个trait；
* 这里的`dyn std::error::Error`是Rust中的多态，代表的是实现了`std::error::Error`的[Trait object](https://doc.rust-lang.org/reference/types/trait-object.html)
* [`Box`](https://doc.rust-lang.org/book/ch15-01-box.html)代表的是指向堆中的数据。

但是`Result<String, Box<dyn std::error::Error>>`这么长的返回值，代码非常不好看。

目前有[`anyhow`](https://github.com/dtolnay/anyhow)、[`eyre`](https://github.com/eyre-rs/eyre)、[~~`failure`~~](https://github.com/rust-lang-deprecated/failure)等库可以很好的解决这个问题。这个待会儿再讲。

## 捕获Boxed错误

到目前为止，在`main`函数中我们只打印了错误，但没有处理它们。如果我们想处理`Box<dyn std::error::Error>`错误并从中恢复，我们需要对它们进行“downcast”：

```diff rust
use chrono::NaiveDate;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
-     Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
+     Err(e) => {
+       eprintln!("Oh noes, we don't know which era we're in! :(");
+       if let Some(err) = e.downcast_ref::<reqwest::Error>() {
+         eprintln!("Request Error: {}", err)
+       } else if let Some(err) = e.downcast_ref::<chrono::format::ParseError>() {
+         eprintln!("Parse Error: {}", err)
+       }
+     }
    }
  }

  fn get_current_date() -> Result<String, Box<dyn std::error::Error>> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
    let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

请注意，我们需要了解`get_current_date`实现细节（内部的不同错误）才能在`main`中对它们进行“downcast”。

可以参考[`downcast`](https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast)、[`downcast_mut`](https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast_mut)

## 应用(Application) vs 库(Library)

如前所述，`Box<dyn std::error::Error>`的缺点是，如果我们想处理潜在的错误，我们需要了解实现细节。当我们返回`Box<dyn std::error::Error>`时，具体类型信息将被擦除了。

为了以不同的方式处理不同的错误，我们需要将它们转换为具体类型，并且这种转换可能会在运行时失败。

然而，在没有上下文的情况下，说某事是“缺点”并不是很有用。一个好的经验法则是考虑一下你正在编写的代码是“应用程序”还是“库”：

### 应用

* 你正在编写的代码将由终端用户使用
* 应用程序代码产生的大多数错误不会被处理，而是记录或报告给用户
* 使用`Box<dyn std::error::Error>`是可以的

### 库

* 您正在编写的代码将被其他代码使用。“库”可能是开源`crate`、内部库等
* 错误是库 API 的一部分，因此您的使用者要知道会出现哪些错误并从中恢复
* 库中的错误通常由使用者处理，因此需要对其进行结构化且易于[match错误](https://doc.rust-lang.org/1.30.0/book/2018-edition/ch06-02-match.html#matches-are-exhaustive)
* 如果`Box<dyn std::error::Error>`，则使用者需要注意代码、依赖项等产生的错误
* 我们可以返回自定义错误，而不是`Box<dyn std::error::Error>`

## 自定义错误

对于库代码，我们可以将所有错误转换为我们自定义的错误并传递它们而不是使用`Box<dyn std::error::Error>`。

在上面的例子中，有两个错误`reqwest::Error`和`chrono::format::ParseError`，我们可以把他们分别转换为`MyCustomError::HttpError`和`MyCustomError::ParseError`。

```rust
// error.rs

pub enum MyCustomError {
  HttpError,
  ParseError,
}
```

[`Error`](https://doc.rust-lang.org/std/error/trait.Error.html)trait继承自`Debug`和`Display`两个trait，因此我们还要实现[`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html)和[`Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html)这两个trait：

```rust
// error.rs

use std::fmt;

#[derive(Debug)]
pub enum MyCustomError {
  HttpError,
  ParseError,
}

impl std::error::Error for MyCustomError {}

impl fmt::Display for MyCustomError {
  fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
    match self {
      MyCustomError::HttpError => write!(f, "HTTP Error"),
      MyCustomError::ParseError => write!(f, "Parse Error"),
    }
  }
}
```

我们创建了自己的自定义错误！

这显然是一个简单的示例，因为错误变体没有包含有关错误的更多信息。但这应该足以作为创建更复杂和更现实的自定义错误的起点。

以下是流行的库中一些典型的例子：[ripgrep](https://github.com/BurntSushi/ripgrep/blob/12.1.1/crates/regex/src/error.rs)、[reqwest](https://github.com/seanmonstar/reqwest/blob/v0.10.7/src/error.rs)、[csv](https://github.com/BurntSushi/rust-csv/blob/master/src/error.rs) 和 [serde_json](https://github.com/serde-rs/json/blob/master/src/error.rs)。

我们也可以使用[`thiserror`](https://github.com/dtolnay/thiserror)、[`snafu`](https://github.com/shepmaster/snafu)、[`quick-error`](https://github.com/tailhook/quick-error)等库来帮助我们自定义错误。

## 向上传递自定义错误

让我们用自定义错误来修改我们的代码：

```diff rust
 // main.rs

+ mod error;

  use chrono::NaiveDate;
+ use error::MyCustomError;
  use std::collections::HashMap;

  fn main() {
    // skipped, will get back later
  }

- fn get_current_date() -> Result<String, Box<dyn std::error::Error>> {
+ fn get_current_date() -> Result<String, MyCustomError> {
    let url = "https://postman-echo.com/time/object";
-   let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;
+   let res = reqwest::blocking::get(url)
+     .map_err(|_| MyCustomError::HttpError)?
+     .json::<HashMap<String, i32>>()
+     .map_err(|_| MyCustomError::HttpError)?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
-   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
+   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")
+     .map_err(|_| MyCustomError::ParseError)?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

请注意：我们用`map_err`将错误从一种类型转换为另一种类型。

但结果事情变得冗长了——我们的函数充斥着这些`map_err`调用。我们可以实现 [From](https://doc.rust-lang.org/std/convert/trait.From.html) trait，以便在使用运算符时自动强制执行错误类型转换：

```diff rust
  // error.rs

  use std::fmt;

  #[derive(Debug)]
  pub enum MyCustomError {
    HttpError,
    ParseError,
  }

  impl std::error::Error for MyCustomError {}

  impl fmt::Display for MyCustomError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
      match self {
        MyCustomError::HttpError => write!(f, "HTTP Error"),
        MyCustomError::ParseError => write!(f, "Parse Error"),
      }
    }
  }

+ impl From<reqwest::Error> for MyCustomError {
+   fn from(_: reqwest::Error) -> Self {
+     MyCustomError::HttpError
+   }
+ }

+ impl From<chrono::format::ParseError> for MyCustomError {
+   fn from(_: chrono::format::ParseError) -> Self {
+     MyCustomError::ParseError
+   }
+ }
```

```diff rust
  // main.rs

  mod error;

  use chrono::NaiveDate;
  use error::MyCustomError;
  use std::collections::HashMap;

  fn main() {
    // skipped, will get back later
  }

  fn get_current_date() -> Result<String, MyCustomError> {
    let url = "https://postman-echo.com/time/object";
-   let res = reqwest::blocking::get(url)
-     .map_err(|_| MyCustomError::HttpError)?
-     .json::<HashMap<String, i32>>()
-     .map_err(|_| MyCustomError::HttpError)?;
+   let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
-   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")
-     .map_err(|_| MyCustomError::ParseError)?;
+   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

删除了代码，代码看起来更干净了！

然而，From trait不是灵丹妙药，有时我们还是需要使用`map_err`。

在上面的示例中，我们已将类型转换从`get_current_date`函数内部移动到`From<X> for MyCustomError`实现中。当进行错误转换时，可以从原始错误对象获取所有需要的信息时，这很有效。

如果原始错误对象没有我们要的信息，我们还是需要在`get_current_date`里面调用`map_err`。

## 匹配自定义对象

目前为止，我们还没有对`main`函数进行修改，在这里我们可以处理自定义错误。

```diff rust
  // main.rs

  mod error;

  use chrono::NaiveDate;
  use error::MyCustomError;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
      Err(e) => {
        eprintln!("Oh noes, we don't know which era we're in! :(");
-       if let Some(err) = e.downcast_ref::<reqwest::Error>() {
-         eprintln!("Request Error: {}", err)
-       } else if let Some(err) = e.downcast_ref::<chrono::format::ParseError>() {
-         eprintln!("Parse Error: {}", err)
-       }
+       match e {
+         MyCustomError::HttpError => eprintln!("Request Error: {}", e),
+         MyCustomError::ParseError => eprintln!("Parse Error: {}", e),
+       }
      }
    }
  }

  fn get_current_date() -> Result<String, MyCustomError> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
    let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

请注意，与`Box<dyn std::error::Error>`不同，我们实际上可以匹配`MyCustomError`枚举内的变体。

## 使用anyhow + thiserror来处理错误

上面提到了几个第三方库，最推荐的就是[`anyhow`](https://github.com/dtolnay/anyhow)和[`thiserror`](https://github.com/dtolnay/thiserror)。

`thiserror`是用来自定义错误的，适合用在库中；`anyhow`不关心错误类型，适合用在应用中。

我们用`thiserror`来定义错误：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyCustomError {
  #[error("http request error")]
  HttpError(#[from] reqwest::Error),
  #[error(transparent)]
  ParseError(#[from] chrono::format::ParseError),
}
```

代码确实干净了很多！我们看一下这里的几个宏的作用：

* `#[derive(Error)`帮我们实现了Error
* `#[error("http request error")]`帮我们实现了`Display`trait的显示内容
* `#[from] reqwest::Error`帮我们实现了`From<reqwest::Error>`的转换逻辑
* `#[error(transparent)]`表示该错误只是作为其他错误的容器，它的错误消息将直接代理为“源”错误


## Option

在很多现代化编程语言中，为了避免空指针的问题，都提供了Option的功能。C++17提供了[std::optional](https://en.cppreference.com/w/cpp/utility/optional)，Java8也提供了[java.util.Optional](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html)。

Rust也有一个[Option](https://doc.rust-lang.org/std/option/)枚举来实现同样的功能。

```rust
fn divide(numerator: f64, denominator: f64) -> Option<f64> {
    if denominator == 0.0 {
        None
    } else {
        Some(numerator / denominator)
    }
}

fn main() {
   // 函数的返回值是一个Option
   let result = divide(2.0, 3.0);
   
   // 对Option进行匹配，取出里面包含的值
   match result {
       // The division was valid
       Some(x) => println!("Result: {x}"),
       // The division was invalid
       None    => println!("Cannot divide by 0"),
   }
}
```

`Option`也可以用`?`操作符进行简写，比如下面的函数：

```rust
fn add_last_numbers(stack: &mut Vec<i32>) -> Option<i32> {
    let a = stack.pop();
    let b = stack.pop();

    match (a, b) {
        (Some(x), Some(y)) => Some(x + y),
        _ => None,
    }
}
```

可以简写为：

```rust
fn add_last_numbers(stack: &mut Vec<i32>) -> Option<i32> {
    Some(stack.pop()? + stack.pop()?)
}
```

代码立马清爽了很多。

注意如果必须处理`Option`为`None`的情况，还是要编写判断逻辑。

* https://doc.rust-lang.org/std/result/
* https://doc.rust-lang.org/std/macro.panic.html
* https://doc.rust-lang.org/std/option/
* https://www.sheshbabu.com/posts/rust-error-handling/
* https://rustmagazine.github.io/rust_magazine_2021/chapter_2/rust_error_handle.html
