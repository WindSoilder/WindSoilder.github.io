# Nushell expression parsing progress

Previously, I noted something like how nushell parse from user input string to block at high level, it's something like this:

```
                           lex parse            lite parse                parse
user input command(string) ----------> Tokens ------------->  LiteBlocks ------------> Blocks
```

Here I'll take a note how a `Block` is parsed.

## What is a nushell block
A nushell block looks something like this:
```nu
do {
    666
    echo aa
    echo bb
}
```

As we can see, a block is composed of several expressions: `666`, `echo aa`, `echo bb`.  Parse from user input to a block is the same to parsing user input to several expressions, and combine them to a block.  So the essential problem is how to parse from user input to an expression.

## How to parse to an expression
Generally, nushell input parsing progress is something like this:
```

                             User input
                                 │
                                 │
                                 │ parse expression
                                 │
                                 ▼
                   is the expression like a math expression?
                                 │
                                 │
                                 │
                                 │
                             Yes │   No
  Parse as a math expression◄────┴───────►Parse as a call to command
              │                                       │
              │parse lhs                              │
              │                                       │find cmd def
              ▼                                       │
         have rhs?                                    ▼
              │                                     found?
              │                                       │
              │                                       │
          Yes │ No                                Yes │ No
parse rhs◄────┴────►return lhs    parse cmd call ◄────┴────► parse as external call
    │                                   │                    (run program in system)
    │math op                            │
    │                                   │
    ▼                                   ▼
return result                 parse parameter value
                              and validate if input
                              is valid

```
At first level, we check:
1. if the input is likely to be a math expression(`666` in our example).
2. If it's not a math expression, it's a call to command(`echo` in our example).

### Parse math expression
When nushell want to parse a math expression, it parses lhs(`666`) value by a `parse_value` function, with `SyntaxShape::Any`.  Then the math expression doesn't have rhs, it returns `lhs`

### Parse a cmd call
During parse call, nushell checks if it's an internal command, then it parses it's arguments by `parse_value` function.  What's the values shape?  Well, if it's a nushell internal command, the argument's shape is defined like this:

```rust
Signature::build("str starts-with")
    .required("string", SyntaxShape::String, "the string to match")
```

As we can see, the first argument of `str starts-with` is a `String`, so nushell will parse 1st arg value with `SyntaxShape::String`.  It's also something called type-directed parsing.

If it isn't a nushell internal command, normally we parses argument as a `String` too, because in other shell, they always pass user input as a string to external command.

## About `parse_value`
As we can see, `parse_value` is called whatever we want to parse a pure math expression, or command's arg value, it accepts `user input`, and relative `SyntaxShape`, we've seen `SyntaxShape::Any` and `SyntaxShape::String` before.

When nushell parses a value, it does the following check:
1. if input value is "true", "false", "null", or,
2. if it's starts with '$', '(', '{', '[', then the expression leads to following cases:
-  '$' ==> dollar_expr ==> can be string interpolation, range, or full cell path (a variable expression).
-  '(' ==> can be range, signature, or full cell path (a sub expression)
-  '{' ==> can be closure, block, or full cell path (a record expression)
-  '[' ==> check relative shape, can be List, Table, Signature
3. check input syntax shape:
- If syntax shape `SyntaxShape::Any`, it try to parse input as `Binary`, `FileSize`, `Duration`, `Range`, `DateTime`, `Record`...`String`.
- If syntax shape is `SyntaxShape::String`, it try to parse input as a string.

## Conclusion
When nushell parse user input to an expression, if it's a math expression, nushell invoke `parse_value` to parse from input to value.  If it's a command call, nushell checks if user input argument is valid, and using `parse_value` function to parse argument value.
