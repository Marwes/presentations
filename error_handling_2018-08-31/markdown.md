class: center, middle

# Error handling 

???


---

# try! / ?

```rust
try!(function(arg, arg2))
function(arg, arg2)?;
```

???


--

```rust
match function(arg, arg2) {
    Ok(o) => o,
    Err(err) => return Err(err.into()),
}
```

--


```rust
match function(arg, arg2) {
    Ok(o) => o,
*   Err(err) => return Err(err.into()),
}
```

???

`Into::into` 

---
name: error-enums

# Error enums

---
template: error-enums

[`quick-error`](https://crates.io/crates/quick-error)

```rust
#[macro_use] extern crate quick_error;

quick_error! {
    #[derive(Debug)]
    pub enum SomeError {
        Io(err: std::io::Error) {
            cause(err)
            description(err.description())
        }
        Utf8(err: std::str::Utf8Error) {
            description("utf8 error")
        }
        Other(err: Box<std::error::Error>) {
            cause(&**err)
            description(err.description())
        }
    }
}
```

---
template: error-enums

[`error-chain`](https://crates.io/crates/error-chain)

```rust

error_chain! {
    types {
        Error, ErrorKind, ResultExt, Result;
    }

    links {
        Another(other_error::Error, other_error::ErrorKind) #[cfg(unix)];
    }

    foreign_links {
        Fmt(::std::fmt::Error);
        Io(::std::io::Error) #[cfg(unix)];
    }

    errors {
        InvalidToolchainName(t: String) {
            description("invalid toolchain name")
            display("invalid toolchain name: '{}'", t)
        }
    }
}
```

???

Much more complex syntax

Supports backtraces (fixes the nesting problem)

More complex to use

---
name: failure

template: error-enums


[`failure`](https://crates.io/crates/failure)

---
template: failure

```rust
#[derive(Debug, Fail)]
enum ToolchainError {
    #[fail(display = "invalid toolchain name: {}", name)]
    InvalidToolchainName {
        name: String,
    },
    #[fail(display = "unknown toolchain version: {}", version)]
    UnknownToolchainVersion {
        version: String,
    }
}
```
???

Supports structs (think wrapping a C error code)

Still supports backtraces but is now explicit (can avoid the cost).

Adds the `Fail` trait as opposed to using `std::error::Error` (all `std::error::Error` is `Fail`).


--- 
name: error-traits

# Error trait objects

---
template: error-traits

# Error trait objects

```rust

pub trait Error: Debug + Display {
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&Error> { ... }
}

fn f() -> Result<(), Box<std::error::Error>> {
}
```

???

Supports downcasting like `std::any::Any`

Quite useful as "any" error type can

---
template: error-traits

# Error trait objects

```rust

pub trait Error: Debug + Display {
    fn description(&self) -> &str { ... }
*   fn cause(&self) -> Option<&Error + 'static> { ... }
}


fn f() -> Result<(), Box<std::error::Error>> {
}
```

https://github.com/rust-lang/rust/issues/35943

???

Mistake caused `cause` to lack of static bound, preventing downcasting

---
template: error-traits

# Error trait objects

[`failure`](https://crates.io/crates/failure)

```rust
pub trait Fail: Display + Debug + Send + Sync + 'static {
    fn cause(&self) -> Option<&Fail> { ... }
    fn backtrace(&self) -> Option<&Backtrace> { ... }
    fn context<D>(self, context: D) -> Context<D>
    where
        D: Display + Send + Sync + 'static,
        Self: Sized { ... }
    fn compat(self) -> Compat<Self>
    where
        Self: Sized, { ... }
}

mod failure {
    struct Error(Box<Fail>);
}

impl<F: Fail> From<F> for Error {
    fn from(failure: F) -> Error {
        ...
    }
}
```

???

Holds a backtrace

Useful as "any" error type can

Similar to java runtime exceptions in a way, "the function can throw any exception"

---

# Failure

???

Closest to what errors eventually will be 

---

## ResultExt

```rust
pub trait ResultExt<T, E> {
    fn compat(self) -> Result<T, Compat<E>>;
    fn context<D>(self, context: D) -> Result<T, Context<D>>
    where
        D: Display + Send + Sync + 'static;
    fn with_context<F, D>(self, f: F) -> Result<T, Context<D>>
    where
        F: FnOnce(&E) -> D,
        D: Display + Send + Sync + 'static;
}
```

???



