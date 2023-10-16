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

## Relationships between overlay, module, scopeframe and stack
Copied from @Kubouch's message:

---------------------

So:
1. module is a collection of definitions, nothing more really.
2. ScopeFrame holds the visibility of symbols inside a scope. For example:
```nushell
def foo [] {
    # scope 1
    alias ll = ls -l
    def spam [] {
        # scope 2
        alias la = ls -la
    }
}
```

The scope frames are arranged like a stack: the ll alias is visible in scope 1 but not in scope 2. la is visible in both scope 1 and 2.
So far no overlays: These scopes (ScopeFrames in the engine code) are arranged in a simple flat list.

-------------------

Now, overlays add one level of indirection. They turn the flat list into a simple tree structure: Each scope frame holds a stack of OverlayFrames which work exactly as the scopes I describe above (you'll notice that the ScopeFrame doesn't actually hold any definitions -- only overlays do and by default there is only one "zero" overlay holding all the definitions).
Example:
```nushell
def foo [] {
    # scope 1
    alias ll = ls -l       # adds ll to scope 1 and default overlay
    overlay new foo        # creates a new overlay foo
    alias gd = git diff    # adds gd to scope 1 and foo overlay
    def spam [] {          # defines spam inside scope 1 and foo overlay
        # scope 2
        alias la = ls -la  # adds la to scope 2, foo overlay
        overlay hide foo   # removes foo overlay from the overlay stack in scope 2
        alias ga = git add # adds ga to scope 2, default overlay
    }
    # overlay foo is active now
    # la, ga not available
    # gd, spam are available
    overlay hide foo      # removes foo overlay from the overlay stack in scope 1
    # only ll is available now
}
```

-------------------

At the line alias la = ls -la, the data structure looks like this:
```
scope1 ----------------> scope2
 \-> overlay "zero" (2)
  \-> overlay "foo" (1)
```

The number in braces denote the order in which symbols are looked up (e.g., when calling a command).
When you enter a new scope, it inherits the overlay stack of the parent scope, that's why scope2 has both "zero" and "foo" overlays. However, they are empty by default: The ll and gd are a part of overlay stack in scope1.

Now, it gets a bit complicated because the engine state is spread between two data structures: EngineState (immutable) and StateWorkingSet (mutable). This is problematic when removing definitions because you can't just remove something from EngineState: You need to record the removal inside the StateWorkingSet, then perform the actual removal during merge_delta(). (Overlay hiding is called "removing" in the engine_state.rs, btw, I originally called overlay hide overlay remove). You might have see the removed_overlays array being passed around: This contains overlays in the EngineState being marked for hiding by the StateWorkingSet. And this brings us closer to implementing overlay delete: If the overlay you want to delete is inside StateWorkingSet (not merged into the EngineState yet), you can probably just delete it. If you're deleting an overlay that is in the permanent state (that's how we call EngineState sometimes), you'd need to pass around deleted_overlays array and then perform the deleting inside merge_delta().

But after overlay hide foo in scope2:
```
scope1 ----------------> scope2
 \-> overlay "zero" (2)   \-> overlay "foo" marked as removed
  \-> overlay "foo" (1)
```
  
This is being tracked by the active_overlays and removed_overlays in the ScopeFrame: https://github.com/nushell/nushell/blob/39e51f1953fd3f4bd39158af92c2fd8d74bd9673/crates/nu-protocol/src/engine/overlay.rs#L59-L62
