# Nushell expression parsing progress

Previously, I noted something like how nushell parse from user input string to block at high level, it's something like this:

```
                           lex parse            lite parse                parse
user input command(string) ----------> Tokens ------------->  LiteBlock ------------> Block
```

Here I'll take a note how a `Block` is parsed.  In detail, how to parse user input from `LiteBlock` to an `expression`, because `Block` is composed of multiple `expressions`

The main entry point is `parse_expression` function.

## Parsing progress
1. firstly, check if the input is likely to be a math expression.
2. or else it's a call, which can be an interncal call or external call.

## Parse math expression
parse lhs value by `parse_value` function, with `SyntaxShape::Any`, if this math expression doesn't have rhs(just a pure lhs), then return `lhs`

## Parse a call
During parse call, it parses it's arguments' value by `parse_value` function.  With specific syntax shape, it's something called typed-directed parsing.

Take the following command as example:
```rust
Signature::build("str starts-with")
    .required("string", SyntaxShape::String, "the string to match")
```

When calling `str starts-with ss`,  it'll call `parse_value` function with `SyntaxShape::String`.

## About `parse_value`
Normally, when user input something, the syntax shape is `SyntaxShape::Any`, then it does the following check:
1. if input value is "true", "false", "null", or,
2. if it's starts with '$', '(', '{', '[', if so, nushell thought the input value will be something like the following, this step allows subexpression parsing, or parsing a list of value.
2.1  '$' ==> dollar_expr ==> can be strin ginterpolation, range, or full cell path
2.2  '(' ==> can be range, signature, or full cell path
2.3  '{' ==> can be closure, block, record, or full cell path
2.4  '[' ==> check relative shape, can be List, Table, Signature

3. parse by input syntax shape.
3.1 parse value based on shape, if it's number, then parse number, if it's duration, then parse as duration, and so on.
3.2 A special thing to note that if syntax shape is SyntaxShape::Any, it try to parse for `Binary`, `FileSize`, `Duration`, `Range`, `DateTime`, `Record`...`String`.
