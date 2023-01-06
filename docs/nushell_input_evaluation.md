# How nushell evaluates user input

This blog will note how nushell evaluates user input.

At high level, the data flow will be something like this:

```
                           lex parse            lite parse                parse
user input command(string) ----------> Tokens ------------->  LiteBlock ------------> Block
```

In detail:
1. `Tokens` are components of user input command, which mainly separate by space, or pipe, or something else
2. Lite parsing converts a stream of tokens to a syntax element structure(`LiteBlock`) that can be parsed.
3. parsing from `LiteBlock` to AST `Block`.

Here are some examples of how nushell parses user input commands:

## Example 1: `^ls -alh out> here11`
1. doing lex parsing, here is lex parsing result
```rust
// lex structure:
[
   Token { contents: Item, span: Span { start: 39486, end: 39489 } },    // token for ^ls
   Token { contents: Item, span: Span { start: 39490, end: 39494 } },    // token for -alh
   Token { contents: OutGreaterThan, span: Span { start: 39495, end: 39499 } },   // token for out>
   Token { contents: Item, span: Span { start: 39500, end: 39506 } },    // token for here11
]
```
Span is a special structure indicates the region of token contents, we can get detailed body from span.
As we can see, our lex contains 4 tokens, they are separated by space.

2. parse the lex structure into a structure called `LiteBlock`:
```rust
// LiteBlock
LiteBlock {
    block: [
        LitePipeline {
            commands: [
                Command(None, LiteCommand { comments: [], parts: [Span { start: 39486, end: 39489 }, Span { start: 39490, end: 39494 }] }),
                Redirection(Span { start: 39495, end: 39499 }, Stdout, LiteCommand { comments: [], parts: [Span { start: 39500, end: 39506 }] }),
            ]
        }
    ]
}
```

After analysing tokens, nushell knows that it's going to be two commands, one is `ls -alh`, the other is redirection indicator `out> here11`.

3. Convert from `LiteBlock` to `Block`, here is the block pipelines
```rust
Pipeline {
    elements: [
        Expression(
            None,
            Expression {
                expr:
                    ExternalCall(
                        Expression { expr: String("ls"), span: Span { start: 39487, end: 39489 }, ty: String, custom_completion: None },
                        [Expression { expr: String("-alh"), span: Span { start: 39490, end: 39494 }, ty: String, custom_completion: None }],
                        false
                    ),
                span: Span { start: 39486, end: 39494 },
                ty: Any, custom_completion: None
            }
        ),
        Redirection(Span { start: 39495, end: 39499 }, Stdout, Expression { expr: String("here11"), span: Span { start: 39500, end: 39506 }, ty: String, custom_completion: None }),
    ]
}
```

It contains more information about our commands, our pipeline contains 2 elements:
- first one is nushell expression, and it contains external call with name "ls" and arguments "-alh"
- second one is redirection indicator, it tells us that we need to redirect stdout to file "here11".

4. Finally nushell will eval given block, during evaluation, it evaluates elements one by one.

## Example 2: `^ls -alh | save --raw a.txt`
1. doing lex parse, here is lex parse result
```rust
# lex structure
[
    Token { contents: Item, span: Span { start: 36960, end: 36963 } }, // token for ^ls
    Token { contents: Item, span: Span { start: 36964, end: 36968 } }, // token for -alh
    Token { contents: Pipe, span: Span { start: 36969, end: 36970 } }, // token for |
    Token { contents: Item, span: Span { start: 36971, end: 36975 } }, // token for save
    Token { contents: Item, span: Span { start: 36976, end: 36981 } }, // token for --raw
    Token { contents: Item, span: Span { start: 36982, end: 36987 } }  // token for a.txt
]
```
As we can see, our lex contains 6 tokens, they are separated by space and pipe.

2. parse the lex structure into a `LiteBlock`:
```rust
# lite block
LiteBlock { block: [
    LitePipeline { commands: [
        Command(
            None,
            LiteCommand { comments: [], parts: [Span { start: 36960, end: 36963 }, Span { start: 36964, end: 36968 }]}
        ),
        Command(
            Some(Span { start: 36969, end: 36970 }),
            LiteCommand { comments: [], parts: [Span { start: 36971, end: 36975 }, Span { start: 36976, end: 36981 }, Span { start: 36982, end: 36987 }] })
    ]}
]}
```

It still contains two commands, one is `^ls -alh`, the other one is `save --raw a.txt`.

3. Convert from `LiteBlock` to `Block`, here is the block pipelines
```rust
# debug block pipelines
[
    Pipeline {
        elements: [
            Expression(
                None,
                Expression {
                    expr: ExternalCall(
                        Expression { expr: String("ls"), span: Span { start: 36961, end: 36963 }, ty: String, custom_completion: None },
                        [Expression { expr: String("-alh"), span: Span { start: 36964, end: 36968 }, ty: String, custom_completion: None }],
                        false
                    ),
                    span: Span { start: 36960, end: 36968 }, ty: Any, custom_completion: None }
            ),
            Expression(
                Some(Span { start: 36969, end: 36970 }),
                Expression {
                    expr: Call(Call {
                        decl_id: 202,
                        head: Span { start: 36971, end: 36975 },
                        arguments: [
                            Named((Spanned { item: "raw", span: Span { start: 36976, end: 36981 } }, None, None)),
                            Positional(Expression { expr: Filepath("a.txt"), span: Span { start: 36982, end: 36987 }, ty: String, custom_completion: None })
                        ],
                        redirect_stdout: true,
                        redirect_stderr: false,
                        parser_info: [] }
                    ),
                    span: Span { start: 36971, end: 36987 },
                    ty: Any,
                    custom_completion: None })
        ]
    }
]
```

It contains two elements:
- first one is nushell expression, it contains external call with name "ls" and arguments "-alh"
- second one is another nushell expression, it contains internal command, the command have declaration id 202(which is "save" command in our case), and a named argument called "raw", a positional argument with value "a.txt"

4. Finally nushell will eval given block, during evaluation, it evaluates elements one by one.

## Reference source code:
1. [eval_source](https://github.com/nushell/nushell/blob/a9bdc655c1fdbad43e811db059bb502c86e16230/crates/nu-cli/src/util.rs#L200) function, it's the main entrypoint in repl.
2. [parse](https://github.com/nushell/nushell/blob/a9bdc655c1fdbad43e811db059bb502c86e16230/crates/nu-parser/src/parser.rs#L5983) function, as we look into the function body, we can see there are two function calls [lex](https://github.com/nushell/nushell/blob/a9bdc655c1fdbad43e811db059bb502c86e16230/crates/nu-parser/src/lex.rs#L286) and [parse_block](https://github.com/nushell/nushell/blob/a9bdc655c1fdbad43e811db059bb502c86e16230/crates/nu-parser/src/parser.rs#L5410).
3. parse_block function accepts lex tokens, and invoke [lite_parse](https://github.com/nushell/nushell/blob/a9bdc655c1fdbad43e811db059bb502c86e16230/crates/nu-parser/src/parser.rs#L5419) to parse from `tokens` to `LiteBlock`, then convert from `LiteBlock` to nushell `Block`.
4. Finally nushell call [eval_block](https://github.com/nushell/nushell/blob/a9bdc655c1fdbad43e811db059bb502c86e16230/crates/nu-cli/src/util.rs#L231) to evaluate `Block`.
