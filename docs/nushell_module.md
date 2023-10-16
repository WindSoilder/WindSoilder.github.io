# How nushell module system works

In nushell, you can define a module like this:
```nushell
# spam.nu
export const b = 3

export def a [] {
    "bcde"
}
```

Here you define a module named `span.nu`, and it exports a function `a`.  Then you can use the module like this:

```
use spam.nu
spam a
```

Here I'll take a note how this module system works.

## High level insight
              
use spam.nu -----> parse `spam.nu`, treated it as a module ----> generate `Module` data structure, and register it to nushell's engine.

Then `use spam.nu` will bring `spam` into our namespace.

The main entrypoint lays inside a function called `parse_module_file_or_dir`, we want to parse `spam.nu`, which is a file, then `parse_module_file_or_dir` will invoke `parse_module_file` to parse.

## How a module is parsed
A module is actually a file with source code, so we'll go into `parse_module_block`, which includes `lex parsing --> lite parsing --> generate block`, finally it'll generate a `Module` data structure.

`Module` data structure includes the following attributes:
```rust
name: Vec<u8>,
decls: IndexMap<Vec<u8>, DeclId>   // it includes exported custom commands.
submodules: IndexMap<Vec<u8>, ModuleId>  // it includes modules indside a module.
constants: IndexMap<Vec<u8>, VarId>  // it includes exported const.
```

## Ok, we have a module, what next?
Nushell will do the following 2 things:
1. parse user imports into `ImportPattern`, and inject it into `parser_info` by `set_parser_info`, so we can retrieve it in runtime.
2. resolve import from `ImportPattern` to `ResolvedImportPattern`.

Nushell will `resolve_import_pattern` accoring to what user want to import, because we can import the following:
```nushell
use spam.nu    # import spam module itself.
use spam.nu a  # import custom command `a` only.
use spam.nu b  # import const `b` only.
```

For `use spam.nu a` and `use spam.nu b` cases, we needs to bring `a` and `b` into our scope directly.  Then we have the following inside `parse_use` function:
```rust
// Extend the current scope with the module's exportables
working_set.use_decls(definitions.decls);
working_set.use_modules(definitions.modules);
working_set.use_variables(constants);
```

For constants, it's a little bit special, because `variables` are founded in `stack`, we also needs to add these varaibles to `stack` in runtime.

Something to notes:
1. nushell retrieves variables from `stack`
2. nushell retrieves custom commands from `engine_state`
