# Mini note about mutable reference

Recently I runs in the code like this
```rust
fn main() {
    let mut s = String::from("asdf");

    let r1 = &mut s;
    aaa(&mut s);
    println!("Hello world! {}", r1);
}
```

It gives me an compile error like this:
```
 --> src/main.rs:5:9
  |
4 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
5 |     aaa(&mut s); // it create a new mutable-reference, we can trick it like `let r2 = &mut s`
  |         ^^^^^^ second mutable borrow occurs here
6 |     println!("Hello, world! {}", r1);
  |                                  -- first borrow later used here
```

Actually I don't get the point, because the following code runs fine:
```rust
fn main() {
    let mut s = String::from("asdf");

    let r1 = &mut s;
    println!("Hello world! {}", r1);
    aaa(&mut s);
}
```

## Why?
Reference to rust borrowing rule: you can have only one mutable reference to a particular piece of data at a time.

and it gives me the following example:
```rust
fn main() {
    let mut s = String::from("asdf");

    let r1 = &mut s;
    let r2 = &mut s;
    println!("Hello world! {}", r1);
}
```

It throws error because when we using reference variable `r1`, there is another reference `r2` to the same `s`.

Back to my code:
```rust
fn main() {
    let mut s = String::from("asdf");

    let r1 = &mut s;
    aaa(&mut s);
    println!("Hello world! {}", r1);
}
```

the error information indicate me that invoking `aaa(&mut s)` also create a mutable reference to `s`, so when we want to use `r1`, we already have another mutable reference to the same `s`.  So this is why it goes wrong.i
