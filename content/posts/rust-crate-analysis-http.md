+++
title = "Rust Crate Http 分析"
date = "2021-10-11T14:37:37+08:00"
description = "Guide to emoji usage in Hugo"
tags = ["Rust"]
draft = true
+++

## 错误处理

```rust
pub struct Error {
    inner: ErrorKind,
}

pub type Result<T> = result::Result<T, Error>;

enum ErrorKind {
    StatusCode(status::InvalidStatusCode),
    Method(method::InvalidMethod),
    Uri(uri::InvalidUri),
    UriParts(uri::InvalidUriParts),
    HeaderName(header::InvalidHeaderName),
    HeaderValue(header::InvalidHeaderValue),
}
```

使用自定义类型描述库中的错误是 Rust 第三方库常见的做法

绝大多数 Rust 库都会定义自己的 Error 类型，http 也不例外，该库的 struct Error 中目前只有一个 inner 字段，表示具体的错误种类。

> 注意千万不要因为只有一个字段就直接使用 `pub type Error = ErrorKind`，这样的话，之后如果有需求要添加新的字段，很容易造成破坏性变更

inner 的类型是 enum ErrorKind，枚举了各种 HTTP 相关的错误，包括非法的状态码，非法的方法等等。

既然已经定义了 Error，那不妨也随手定义一下 Result。

```rust
pub type Result<T> = result::Result<T, Error>;
```

对于 Error 类型，作者为其实现了两个方法


```rust
impl Error {
    /// Return true if the underlying error has the same type as T.
    pub fn is<T: error::Error + 'static>(&self) -> bool {
        self.get_ref().is::<T>()
    }

    /// Return a reference to the lower level, inner error.
    pub fn get_ref(&self) -> &(dyn error::Error + 'static) {
        use self::ErrorKind::*;

        match self.inner {
            StatusCode(ref e) => e,
            Method(ref e) => e,
            Uri(ref e) => e,
            UriParts(ref e) => e,
            HeaderName(ref e) => e,
            HeaderValue(ref e) => e,
        }
    }
}
```

`get_ref(&self)` 用于获取内部具体的错误，`is(&self)` 用于判断内部错误的类型。


对于自定义的 Error 类型，一定要实现标准库的 std::error::Error trait。

```rust
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    fn backtrace(&self) -> Option<&Backtrace> { ... }
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
}
```

而实现该 trait 需要先实现 Debug 和 Display，这确实也是实践中需要的，你需要让用户能够打印出你自定义的错误。

```rust
impl fmt::Debug for Error {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.debug_tuple("http::Error")
            // Skip the noise of the ErrorKind enum
            .field(&self.get_ref())
            .finish()
    }
}

impl fmt::Display for Error {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        fmt::Display::fmt(self.get_ref(), f)
    }
}
```

实现 std::error::Error

```rust
impl error::Error for Error {
    // Return any available cause from the inner error. Note the inner error is
    // not itself the cause.
    fn source(&self) -> Option<&(dyn error::Error + 'static)> {
        self.get_ref().source()
    }
}
```

最后，实现一系列 From trait，用于将内部具体的错误类型转换成该通用错误类型。

```rust

impl From<status::InvalidStatusCode> for Error {
    fn from(err: status::InvalidStatusCode) -> Error {
        Error {
            inner: ErrorKind::StatusCode(err),
        }
    }
}

impl From<method::InvalidMethod> for Error {
    fn from(err: method::InvalidMethod) -> Error {
        Error {
            inner: ErrorKind::Method(err),
        }
    }
}

impl From<uri::InvalidUri> for Error {
    fn from(err: uri::InvalidUri) -> Error {
        Error {
            inner: ErrorKind::Uri(err),
        }
    }
}

impl From<uri::InvalidUriParts> for Error {
    fn from(err: uri::InvalidUriParts) -> Error {
        Error {
            inner: ErrorKind::UriParts(err),
        }
    }
}

impl From<header::InvalidHeaderName> for Error {
    fn from(err: header::InvalidHeaderName) -> Error {
        Error {
            inner: ErrorKind::HeaderName(err),
        }
    }
}

impl From<header::InvalidHeaderValue> for Error {
    fn from(err: header::InvalidHeaderValue) -> Error {
        Error {
            inner: ErrorKind::HeaderValue(err),
        }
    }
}

impl From<std::convert::Infallible> for Error {
    fn from(err: std::convert::Infallible) -> Error {
        match err {}
    }
}
```

## struct

### Request

```rust
let mut request = Request::builder()
    .uri("https://www.rust-lang.org/")
    .header("User-Agent", "my-awesome-agent/1.0");

if needs_awesome_header() {
    request = request.header("Awesome", "yes");
}

```

```rust
pub struct Request<T> {
    head: Parts,
    body: T,
}

pub struct Parts {
    pub method: Method
    pub uri: Uri,
    pub version: Version,
    pub headers: HeaderMap<HeaderValue>,
    pub extensions: Extensions,
    _priv: (),
}

#[derive(Debug)]
pub struct Builder {
    inner: Result<Parts>,
}
```

```rust
impl Request<()> {
    pub fn builder() -> Builder {
        Builder::new()
    }


}
```


```rust
impl Builder {
    pub fn new() -> Builder {
        Builder::default()
    }

    pub fn method<T>(self, method: T) -> Builder
    where
        Method: TryFrom<T>,
        <Method as TryFrom<T>>::Error: Into<crate::Error>,
    {
        self.and_then(move |mut head| {
            let method = TryFrom::try_from(method).map_err(Into::into)?;
            head.method = method;
            Ok(head)
        })
    }
}

impl Default for Builder {
    #[inline]
    fn default() -> Builder {
        Builder {
            inner: Ok(Parts::new()),
        }
    }
}
```

inline 属性建议 rustc 将该函数当作内联函数，即直接在函数被调用的地方展开，而不是执行一个函数调用

该属性只起到一个建议作用，rustc 本身也会基于一个启发式算法自动判断是否将函数当作内联函数

### Response


## 总结

1. 为自己的库实现 Error 和 Result 类型，Error 类型需要精心设计一下，Result 就很简单了 `pub type Result<T> = std::result::Result<T, Error>;`
2. 自定义 Error 类型一定要实现 std::error::Error trait
3. 尽可能地实现标准库中的 trait，例如：Debug，Display，Default，PartialEq，PartialOrd，该过程应该是将所有标准库中的 trait 列一个清单，实现一个类型时，查看该清单，将不适合实现的删去，剩下的都去实现
4. builder 模式


