# How nushell evaluates user input (new version)

There is a [redirection overhaul](https://github.com/nushell/nushell/pull/11934) pr recently, it changes how lite block parsing and block parsing works, so [previous blog](./nushell_input_to_tokens.md) is out of date.

I'm always curious if we type something like `^ls -alh`, how does nushell parse my input and execute the command.

This blog will note how nushell evaluates user input.

## High level insight
At high level, the data flow will be something like this:

```
                           lex parse            lite parse                parse
user input command(string) ----------> Tokens ------------->  LiteBlocks ------------> Blocks
```

In detail:
1. `Tokens` are components of user input command, which mainly separate by space, or pipe, or something else
2. Lite parsing converts a stream of tokens to a syntax element structure(`LiteBlock`) that can be parsed.
3. parsing from `LiteBlock` to AST `Block`.

Here are some examples of how nushell parses user input commands:

## Example 1: `^ls -alh out> here11`
- doing lex parsing, here is lex parsing result
```rust
// lex structure:
 [
    Token { contents: Item, span: Span { start: 12047, end: 12050 } },  // token for ^ls
    Token { contents: Item, span: Span { start: 12051, end: 12055 } },  // token for -alh
    Token { contents: OutGreaterThan, span: Span { start: 12056, end: 12060 } },  // token for out>
    Token { contents: Item, span: Span { start: 12061, end: 12067 } }   // token for here11
]
```
Span is a special structure indicates the region of token contents, we can get detailed body from span.
As we can see, our lex contains 4 tokens, they are separated by space.

- parse the lex structure into a structure called `LiteBlock`:
```rust
// LiteBlock
LiteBlock { 
    block: [
        LitePipeline {
            commands: [
                LiteCommand { 
                    pipe: None,
                    comments: [],
                    parts: [Span { start: 12047, end: 12050 }, Span { start: 12051, end: 12055 }],
                    redirection: Some(Single {
                        source: Stdout,
                        target: File {
                            connector: Span { start: 12056, end: 12060 },
                            file: Span { start: 12061, end: 12067 },
                            append: false
                        } 
                    }) 
                }
            ] 
        }
    ]
}
```

It's different to previous version, in previous version, `redirection` is treated as a single command, along with redirection target.

After analysing tokens, nushell knows that it's going to be 1 command, `ls -alh`, with redirection attribute `out> here11`.  It's a big improvement that we don't need to check next command to know if there is a redirection.

- Convert from `LiteBlock` to `Block`, here is the block pipelines
```rust
Pipeline {
    elements: [
        PipelineElement {
            pipe: None,
            expr: Expression {
                expr:
                    ExternalCall(
                        Expression { expr: String("ls"), span: Span { start: 12048, end: 12050 }, ty: String, custom_completion: None },
                        [
                            Regular(
                                Expression { expr: String("-alh"), span: Span { start: 12051, end: 12055 }, ty: String, custom_completion: None }
                            )
                        ]
                    ),
                    span: Span { start: 12047, end: 12055 },
                    ty: Any,
                    custom_completion: None
            },
            redirection: Some(Single { 
                source: Stdout,
                target: File { expr:
                    Expression {
                        expr: String("here11"),
                        span: Span { start: 12061, end: 12067 },
                        ty: String, custom_completion: None },
                        append: false,
                        span: Span { start: 12056, end: 12060 } 
                } 
            }) 
        }
    ] 
}
```

It contains more information about our commands, our pipeline contains 1 element(different to previous version), with `pipe`, `expr`, and `redirection` attributes.

- Finally nushell will eval given block, during evaluation, it evaluates elements one by one.

## Example 2: `^ls -alh | save --raw a.txt`
- doing lex parse, here is lex parse result
```rust
# lex structure
[
    Token { contents: Item, span: Span { start: 12067, end: 12070 } },  // token for ^ls
    Token { contents: Item, span: Span { start: 12071, end: 12075 } },  // token for -alh
    Token { contents: Pipe, span: Span { start: 12076, end: 12077 } },  // token for |
    Token { contents: Item, span: Span { start: 12078, end: 12082 } },  // token for save
    Token { contents: Item, span: Span { start: 12083, end: 12088 } },  // token for --raw
    Token { contents: Item, span: Span { start: 12089, end: 12094 } }   // token for a.txt
]
```
As we can see, our lex contains 6 tokens, they are separated by space and pipe.

- parse the lex structure into a `LiteBlock`:
```rust
# lite block
LiteBlock { 
    block: [
        LitePipeline {
            commands: [
                LiteCommand {
                    pipe: None,
                    comments: [],
                    parts: [Span { start: 12067, end: 12070 }, Span { start: 12071, end: 12075 }],
                    redirection: None
                },
                LiteCommand {
                    pipe: Some(Span { start: 12076, end: 12077 }),
                    comments: [],
                    parts: [Span { start: 12078, end: 12082 }, Span { start: 12083, end: 12088 }, Span { start: 12089, end: 12094 }],
                    redirection: None 
                }
            ] 
        }
    ]
}
```

It still contains two commands, one is `^ls -alh`, the other one is `save --raw a.txt`.

- Convert from `LiteBlock` to `Block`, here is the block pipelines
```rust
# debug block pipelines
[
    Pipeline {
        elements: [
            PipelineElement {
                pipe: None,
                expr: Expression {
                    expr: ExternalCall(
                        Expression {
                            expr: String("ls"), span: Span { start: 12068, end: 12070 }, ty: String, custom_completion: None 
                        },
                        [
                            Regular(Expression {
                                expr: String("-alh"),
                                span: Span { start: 12071, end: 12075 },
                                ty: String,
                                custom_completion: None
                            })
                        ]
                    ),
                    span: Span { start: 12067, end: 12075 },
                    ty: Any,
                    custom_completion: None
                },
                redirection: None
            },
            PipelineElement {
                pipe: Some(Span { start: 12076, end: 12077 }),
                expr: Expression {
                    expr: Call(
                        Call {
                            decl_id: 197,
                            head: Span { start: 12078, end: 12082 },
                            arguments: [
                                Named((Spanned { item: "raw", span: Span { start: 12083, end: 12088 } }, None, None)),
                                Positional(Expression {
                                    expr: Filepath("a.txt", false),
                                    span: Span { start: 12089, end: 12094 },
                                    ty: String,
                                    custom_completion: None
                                })
                            ],
                            parser_info: {}
                        }
                    ),
                    span: Span { start: 12078, end: 12094 },
                    ty: Nothing,
                    custom_completion: None
                },
                redirection: None 
            }
        ]
    }
]
```

It contains two elements:
1. first one is nushell expression, it contains external call with name "ls" and arguments "-alh"
2. second one is another nushell expression, it contains internal command, the command have declaration id 197(which is "save" command in our case), and a named argument called "raw", a positional argument with value "a.txt"

- Finally nushell will eval given block, during evaluation, it evaluates elements one by one.

## Example 3: `^ls -alh e>| save --raw a.txt`
- doing lex parse, here is lex parse result
```rust
[
    Token { contents: Item, span: Span { start: 12151, end: 12154 } },   // token for ^ls
    Token { contents: Item, span: Span { start: 12155, end: 12159 } },   // token for -alh
    Token { contents: ErrGreaterPipe, span: Span { start: 12160, end: 12163 } },  // token for e>|
    Token { contents: Item, span: Span { start: 12164, end: 12168 } },   // token for save
    Token { contents: Item, span: Span { start: 12169, end: 12174 } },   // token for --raw
    Token { contents: Item, span: Span { start: 12175, end: 12180 } }    // token for a.txt
]
```

As we can see, our lex contains 6 tokens, they are separated by space and pipe.

- parse the lex structure into a `LiteBlock`:
```rust
LiteBlock {
    block: [
        LitePipeline { commands: [
            LiteCommand {
                pipe: None,
                comments: [],
                parts: [Span { start: 12151, end: 12154 }, Span { start: 12155, end: 12159 }],
                redirection: Some(Single {
                    source: Stderr, target: Pipe { connector: Span { start: 12160, end: 12163 } } 
                })    // Note redirection attribute is different to previous example.
            },
            LiteCommand {
                pipe: Some(Span { start: 12160, end: 12163 }),
                comments: [],
                parts: [Span { start: 12164, end: 12168 }, Span { start: 12169, end: 12174 }, Span { start: 12175, end: 12180 }], redirection: None
            }
    ]}
    ]
}
```

It still contains two commands, one is `^ls -alh`, the other one is `save --raw a.txt`.

Note that the redirection attribute of first LiteCommand is different to previous one, which value is None, here the value is `Some(Single{source: Stderr, ...})`.

- Convert from `LiteBlock` to `Block`, here is the block pipelines
```rust
// debug block pipelines
[
    Pipeline {
        elements: [
            PipelineElement { 
                pipe: None,
                expr: Expression {
                    expr: ExternalCall(
                        Expression {
                            expr: String("ls"), span: Span { start: 12152, end: 12154 }, ty: String, custom_completion: None
                        },
                        [
                            Regular(Expression {
                                expr: String("-alh"),
                                span: Span { start: 12155, end: 12159 },
                                ty: String,
                                custom_completion: None
                            })
                        ]
                    ),
                    span: Span { start: 12151, end: 12159 },
                    ty: Any,
                    custom_completion: None
                },
                // Note the redirection attribute is different to previous example
                redirection: Some(Single { source: Stderr, target: Pipe { span: Span { start: 12160, end: 12163 } } }) 
            },
            PipelineElement {
                pipe: Some(Span { start: 12160, end: 12163 }),
                expr: Expression {
                    expr: Call(
                        Call {
                            decl_id: 197,
                            head: Span { start: 12164, end: 12168 },
                            arguments: [
                                Named((Spanned { item: "raw", span: Span { start: 12169, end: 12174 } }, None, None)),
                                Positional(Expression {
                                    expr: Filepath("a.txt", false),
                                    span: Span { start: 12175, end: 12180 }, ty: String, custom_completion: None 
                                })
                            ],
                            parser_info: {}
                        }
                    ),
                    span: Span { start: 12164, end: 12180 },
                    ty: Nothing,
                    custom_completion: None
                },
                redirection: None
            }
        ]
    }
]
```

It contains two elements:
1. first one is nushell expression, it contains external call with name "ls" and arguments "-alh"
2. second one is another nushell expression, it contains internal command, the command have declaration id 197(which is "save" command in our case), and a named argument called "raw", a positional argument with value "a.txt"

Note that the redirection attribute of first PipelineElemet is different to previous example, which value is None, here the value is `Some(Single{source: Stderr, ...})`.
It's important, when we run external command `ls`, we can set stderr attribute of the process to `piped()` directly.


## Example 3: `^ls -alh o+e>| save --raw a.txt`
- doing lex parse, here is lex parse result
```rust
[
    Token { contents: Item, span: Span { start: 13587, end: 13590 } },    // token for ^ls
    Token { contents: Item, span: Span { start: 13591, end: 13595 } },    // token for -alh
    Token { contents: OutErrGreaterPipe, span: Span { start: 13596, end: 13601 } },  // token for o+e>|
    Token { contents: Item, span: Span { start: 13602, end: 13606 } },    // token for save
    Token { contents: Item, span: Span { start: 13607, end: 13612 } },    // token for --raw
    Token { contents: Item, span: Span { start: 13613, end: 13618 } }     // token for a.txt
]
```

As we can see, our lex contains 6 tokens, they are separated by space and pipe.

- parse the lex structure into a `LiteBlock`:
```rust
LiteBlock {
    block: [
        LitePipeline {
            commands: [
                LiteCommand {
                    pipe: None,
                    comments: [],
                    parts: [
                        Span { start: 13587, end: 13590 }, Span { start: 13591, end: 13595 }
                    ],
                    redirection: Some(Single { source: StdoutAndStderr, target: Pipe { connector: Span { start: 13596, end: 13601 } } })
                },
                LiteCommand { 
                    pipe: Some(Span { start: 13596, end: 13601 }), 
                    comments: [], 
                    parts: [
                        Span { start: 13602, end: 13606 },
                        Span { start: 13607, end: 13612 },
                        Span { start: 13613, end: 13618 }
                    ],
                    redirection: None 
                }
            ]
        }
    ]
}
```

It still contains two commands, one is `^ls -alh`, the other one is `save --raw a.txt`.

Note that the redirection attribute of first LiteCommand is different to previous one, which value is None, here the value is `Some(Single{source: Stderr, ...})`.

- Convert from `LiteBlock` to `Block`, here is the block pipelines
```rust
// debug block pipelines
[
    Pipeline {
        elements: [
            PipelineElement {
                pipe: None,
                expr: Expression {
                    expr: ExternalCall(
                        Expression {
                            expr: String("ls"), span: Span { start: 13588, end: 13590 }, ty: String, custom_completion: None
                        },
                        [
                            Regular(Expression { expr: String("-alh"), span: Span { start: 13591, end: 13595 }, ty: String, custom_completion: None })
                        ]
                    ),
                    span: Span { start: 13587, end: 13595 },
                    ty: Any,
                    custom_completion: None
                },
                redirection: Some(Single { source: StdoutAndStderr, target: Pipe { span: Span { start: 13596, end: 13601 } } })
            },
            PipelineElement {
                pipe: Some(Span { start: 13596, end: 13601 }),
                expr: Expression {
                    expr: Call(
                        Call {
                            decl_id: 207,
                            head: Span { start: 13602, end: 13606 },
                            arguments: [
                                Named((Spanned { item: "raw", span: Span { start: 13607, end: 13612 } }, None, None)),
                                Positional(Expression { expr: Filepath("a.txt", false), span: Span { start: 13613, end: 13618 }, ty: String, custom_completion: None })
                            ],
                            parser_info: {} 
                        }
                    ),
                    span: Span { start: 13602, end: 13618 }, ty: Nothing, custom_completion: None },
                    redirection: None 
            }
        ]
    }
]
```

It contains two elements:
1. first one is nushell expression, it contains external call with name "ls" and arguments "-alh"
2. second one is another nushell expression, it contains internal command, the command have declaration id 207(which is "save" command in our case), and a named argument called "raw", a positional argument with value "a.txt"

Note that the redirection attribute of first PipelineElemet is different to previous example, which value is None, here the value is `Some(Single{source: StdoutAndStderr, ...})`.
It's important, when we run external command `ls`, we can set stderr attribute of the process to `piped()` directly.


## Reference source code:
1. [eval_source](https://github.com/nushell/nushell/blob/40f72e80c3a4d35ea58405539cee056e0e77653e/crates/nu-cli/src/util.rs#L204) function, it's the main entrypoint in repl.
2. [parse](https://github.com/nushell/nushell/blob/40f72e80c3a4d35ea58405539cee056e0e77653e/crates/nu-parser/src/parser.rs#L6213) function, as we look into the function body, we can see there are two function calls [lex](https://github.com/nushell/nushell/blob/40f72e80c3a4d35ea58405539cee056e0e77653e/crates/nu-parser/src/lex.rs#L351) and [parse_block](https://github.com/nushell/nushell/blob/40f72e80c3a4d35ea58405539cee056e0e77653e/crates/nu-parser/src/parser.rs#L5676).
3. parse_block function accepts lex tokens, and invoke [lite_parse](https://github.com/nushell/nushell/blob/40f72e80c3a4d35ea58405539cee056e0e77653e/crates/nu-parser/src/lite_parser.rs#L149) to parse from `tokens` to `LiteBlock`, then convert from `LiteBlock` to nushell `Block`.
4. Finally nushell call [eval_block](https://github.com/nushell/nushell/blob/40f72e80c3a4d35ea58405539cee056e0e77653e/crates/nu-engine/src/eval.rs#L479) to evaluate `Block`.
