---
title: "How-to: Writing a C shared library in rust"
date: 2021-02-23T09:59:05-06:00
draft: false
tags: [Fedora]
---

The ability to write a C shared library in rust has been around for some time
and there is quite a bit of information about the subject available. 
Some examples:

* [Exposing C and Rust APIs: some thoughts from librsvg](https://viruta.org/exposing-c-and-rust-apis-some-thoughts-from-librsvg.html)
* ~~[Creating C/C++ APIs in Rust](https://karroffel.gitlab.io/post/2019-05-15-rust/)~~  (site removed?)
* [Rust Out Your C by Carol (Nichols || Goulding) (youtube video)](https://www.youtube.com/watch?v=SKGVItFlK3w)
* [Exporting a GObject C API from Rust code and using it from C, Python, JavaScript and others](https://coaxion.net/blog/2017/09/exporting-a-gobject-c-api-from-rust-code-and-using-it-from-c-python-javascript-and-others/)
* [Rust Once, Run Everywhere](https://blog.rust-lang.org/2015/04/24/Rust-Once-Run-Everywhere.html)

All this information is great, but what I was looking for was a simple
step-by-step example which also discussed memory handling and didn't delve
into the use of GObjects.  I also included an opaque data type, but I'm not
100% sure if my approach is the most correct.

I'm not going to discuss the entire subject of why you would want to do this.
  I'm thinking that if you're reading this, then you already know why.


## 1. Create your cargo project

```bash
$ cargo new somelibname --lib
```

## 2. Edit Cargo.toml

To make this a shared library usable by C, we need to specify this by 
adding the following:

```
[lib]
name         = "somelibname"
crate-type   = ["rlib", "cdylib"]
```

For more [information](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/cdylib-crates-for-c-interoperability.html)

In our example we are also using the crate `libc` so I'll include that as well.
```
[dependencies]
libc = "0.2.85"
```

## 3. Create your C function declarations
In this example we'll be showing a couple of different ways to allocate a C
string on the heap which will be de-allocated by the caller.  We will also be 
showing an opaque type and a couple of different approaches for 
allocating/de-allocating.

```C
#ifndef SOMELIBNAME
#define SOMELIBNAME

// File: somelibname.h

// Lets use some types which we can easily pair with rust types.
#include <stdint.h>

// Some example C functions that returns a string that has been
// allocated on the heap.  The caller must call free on s to
// prevent a memory leak.  Look at implementation of each to
// see differences.
int get_some_cstr( char **s );
int get_some_cstr_2( char **s );

// Opaque type for some Error
typedef struct _custom_error Error;

// Create function which takes a pointer to a pointer for returning
// the newly allocated type and can also return error codes.
int32_t error_create_with_result(Error **o);


// Free function which takes a pointer to a pointer for freeing
// the memory, returns error code based on ERRNO.
int32_t error_free_with_result(Error **o);


// An alternative simpiler function for allocating a type which
// can only communicate success or fail based on if the returned
// value is non-null.
Error* error_new(void);

// Alternative free function which simply takes a pointer to type to
// de-allocate the memory.  See implementation on how it varies.
void error_free(Error *o);


// Common "getter" C functions which operate on the opaque type.
const char* error_msg_get(const Error *o);
int error_code_get(const Error *o);

#endif
```

## 4. Implement the library in rust
```rust
use libc;
use std::ffi::CString;
use std::os::raw::c_char;

/// File: lib.rs

/// For further reading ...
/// #[no_mangle] - // https://internals.rust-lang.org/t/precise-semantics-of-no-mangle/4098

#[no_mangle]
pub unsafe extern "C" fn get_some_cstr(desc: *mut *mut c_char) -> isize {
    // We want the pointer coming in to not be null and not currently be pointing to something
    // to prevent whatever it's pointing to be lost.
    if desc.is_null() || !(*desc).is_null() {
        return libc::EINVAL as isize;
    }

    let val = CString::new("Returning a string to C to be free() there")
        .expect("Expecting we can allocate a CString");

    // Allocate memory for string as C caller is expected to "free" it.
    // This approach seems to be the safest way to do this, so that you can be certain
    // that the memory is allocated with the same allocator as what the caller will be using to
    // "free" it.  In general having a library which allocates things on the heap and expects the
    // caller to free it is probably not the best thing to do.
    let m = libc::malloc(libc::strlen(val.as_ptr()) + 1) as *mut c_char;
    if m.is_null() {
        return libc::ENOMEM as isize;
    }

    *desc = m;
    libc::strcpy(*desc, val.as_ptr());
    0
}

#[no_mangle]
pub unsafe extern "C" fn get_some_cstr_2(desc: *mut *mut c_char) -> isize {
    // We want the pointer coming in to not be null and not currently be pointing to something
    // to prevent whatever it's pointing to be lost.
    if desc.is_null() || !(*desc).is_null() {
        return libc::EINVAL as isize;
    }

    let val = CString::new("Returning a string to C to be free() there")
        .expect("Expecting we can allocate a CString");

    // The documentation states that the pointer from "into_raw()"
    // needs to be brought back into rust and reconstructed as a CString to be freed
    // correctly, so this example is suppose to result into a memory leak if the memory
    // is released with free().  This is stated because you need to ensure the allocator &
    // de-allocator are the same.
    *desc = val.into_raw();
    0
}

/// Our implementation of the Error type.
pub struct Error {
    magic: u32,
    msg: CString,
    code: isize,
}

///  Some C code uses magic values in structures to determine if the pointer
/// is of the correct type.
const ERROR_MAGIC: u32 = 0xDEADBEEF;

// A function which creates an example Error
fn example_error() -> Error {
    Error {
        magic: ERROR_MAGIC,
        msg: CString::new("Some helpful error message").unwrap(),
        code: -101,
    }
}

/// Adding this so that we can get a message printed when the Error is freed.
impl Drop for Error {
    fn drop(&mut self) {
        println!("Error struct being dropped ...");
    }
}

#[no_mangle]
pub unsafe extern "C" fn error_create_with_result(_e: *mut *mut Error) -> isize {
    let e = Box::new(example_error());
    *_e = Box::into_raw(e);
    0
}

#[no_mangle]
pub unsafe extern "C" fn error_free_with_result(e: *mut *mut Error) -> i32 {
    if e.is_null() || (*e).is_null() {
        return libc::EINVAL;
    }

    // Try to ensure we have a pointer to the correct structure...
    if (*(*e)).magic != ERROR_MAGIC {
        return libc::EINVAL;
    }

    // Reconstruct the Error into a box and then drop it so that it's freed.
    drop(Box::from_raw(*e));
    *e = 0 as *mut Error;
    0
}

/// The next two function examples are taken directly out of the rust documentation
/// ref. https://doc.rust-lang.org/std/boxed/
#[no_mangle]
pub extern "C" fn error_new() -> Box<Error> {
    Box::new(example_error())
}

/// We take ownership as we are passing by value, so when function
/// exits the drop gets run.  Handles being passed null.
#[no_mangle]
pub extern "C" fn error_free(_: Option<Box<Error>>) {}

/// Our example "getter" methods which work on the Error type.  The value
/// returned is only valid as long as the Error has not been freed.  If C
/// caller needs a longer lifetime they need to copy the value.
#[no_mangle]
pub unsafe extern "C" fn error_msg_get(e: &Error) -> *const c_char {
    e.msg.as_ptr()
}

#[no_mangle]
pub extern "C" fn error_code_get(e: &Error) -> isize {
    e.code
}
```

## 5. Create simple C file to exercise library
```C
#include <stdlib.h>
#include <stdio.h>
#include "somelibname.h"
#include <string.h>

// File: main.c
//
// Sample library usage.
int main(void) {

    Error *e = NULL;
    int result = 0;
   
    char *s = NULL;
    result = get_some_cstr(&s);
    if (0 == result ){
        free(s);
        s = NULL;
    } else {
        printf("get_some_cstr Result = %d\n", result);
        return 10;
    }

    result = get_some_cstr_2(&s);
    if (0 == result) {
        //printf("get_some_cstr_2 returned %s\n", s);
        free(s);
        s = NULL;
    } else {
        printf("get_some_cstr_2 Result = %d\n", result);
        return 10;
    }

    e = error_new();
    const char *msg = error_msg_get(e);
    if (msg) {
        printf("error message = %s\n", msg);
        printf("error code = %d\n", error_code_get(e));
    } else {
        printf("Error msg is null :-/\n");
        return 1;
    }

    error_free(e);

    e = NULL;
    result = error_create_with_result(&e);
    if (result == 0) {
        printf("error message = %s\n", error_msg_get(e));
        printf("error code = %d\n", error_code_get(e));

        printf("Result of freeing %d\n", error_free_with_result(&e));
        printf("Value of e = %p (expecting nil)\n", e);
    } else {
        printf("Error: error_create_with_result = %d\n", result);
    }
    
    return 0;
}
```

## 6. Compile library and client
Compile the crate, and the C client code and link it to the shared library.  
I've placed everything in the same directory which looks like:

```
$ tree .
.
├── Cargo.lock
├── Cargo.toml
├── main.c
├── somelibname.h
└── src
    └── lib.rs

1 directory, 5 files
```

```bash
$ cargo build
$ ls target/debug/*.so
target/debug/libsomelibname.so
$ gcc -Wall -g -O0 main.c -I. -Ltarget/debug/ -lsomelibname
```

## 7. Give it a try

```bash
$ LD_LIBRARY_PATH=target/debug ./a.out
error message = Some helpful error message
error code = -101
Error struct being dropped ...
error message = Some helpful error message
error code = -101
Error struct being dropped ...
Result of freeing 0
Value of e = (nil) (expecting nil)
```

I've placed the entire sample on github, https://github.com/tasleson/crispy-memory/tree/main/c_lib_rust_example .
Errors in this example are likely, I'm still learning the language.
For comments or corrections please use github issues or submit a pull request.

Thanks!
