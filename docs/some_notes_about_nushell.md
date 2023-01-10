# Some notes about nushell

## Internal design
### Difference between `Value::Error` and `ShellError`
Copied from discord message: https://discord.com/channels/601130461678272522/615329862395101194/1049582721615269908

commands run in two stages. In the 1st stage, the command creates the iterator the pipeline will use. At this point, it's possible for it to error. If it does, it returns a ShellError instead of the PipelineData.

In the 2nd stage, the iterator pulls on its input and can be pulled on to get its output. If an error occurs at this stage, an error value is sent out the PipelineData.

we can do early error detection during the first stage.
for externals, the first stage is something like "eval the params and get them ready and find the external" once we do that, we move into the second stage,
for us to try something that's an external, we need to actually drain the PipelineData, or at the very least let the external run until we get an exit code,
in a sense, internals an externals have two separate places errors can occur. Internal commands will often error during the 1st stage, while external are much more likely to error in the 2nd stage

## Nushell usage
### Path auto-expand
Currently, if we have a command which accepts `SyntaxShape::FilePath`, and we input string as argument, the path will be auto-expanded.

However, if we pass a variable, or sub-command as argument, the path will not be auto-expanded.

Example:
```nu
❯ help rm
Remove files and directories.

Search terms: delete, remove

Usage:
  > rm {flags} <filename> ...(rest)

# the following will be auto-expanded
❯ rm ~/temp
# but the following will not be auto-expanded
❯ let $dest = '~/temp'
❯ rm $dest
# or string interpretion will not be auto-expanded
❯ rm $"~/temp"
```