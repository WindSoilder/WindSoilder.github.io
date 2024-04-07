# How nushell plugin system work

Recently there is a big improvement on nushell plugin system:

1. [Bidirectional communication and streams for plugins](https://github.com/nushell/nushell/pull/11911)
2. [Keep plugins persistently running in the background](https://github.com/nushell/nushell/pull/12064)

Here I will write how the new system work.

Recall how to run a command from a plugin:
1. register the plugin (e.g: `register ./target/debug/plugin_bin`)
2. run the command directly.


## Register the plugin
For example: [nu_plugin_example]() defines many commands, like: `example one`, `example two`, `example three`.  Nushell needs to have a way to get these commands' signature.  To achieve this, nushell will run the plugin binary, communicate with it to get these signature, then save these signature into local `plugin.nu` file as a cache.

The process can be describe as the following:
```
nushell             plugin
  |  1. start plugin   |
  | ---------------->  |
  |    binary          |
  |                    |
  |                    |
  |  2. get signature  |
  | -----------------> |
  |                    |
  |                    |
  |  3. response       |
  | <----------------  |
  |                    |
  |                    |

```
When we want to register a plugin, nushell will run the plugin binary and get all command signatures from a plugin.  All the interesting things lays inside [get_signature]() function.

### How `get_signautre` works

To get signatures from plugin, nushell will `make_plugin_interface` to enable communication with `plugin` in background, in `make_plugin_interface`, it will create a `PluginInterfaceManager` object, this is the core datastructure of new plugin system, it manages communication between nushell and plugin.  Here is code snippet:

```rust
let mut manager = PluginInterfaceManager::new(source.clone(), (Mutex::new(stdin), encoder));
manager.set_garbage_collector(gc);
let interface = manager.get_interface();
interface.hello()?;
// Spawn the reader on a new thread. We need to be able to read messages at the same time that
// we write, because we are expected to be able to handle multiple messages coming in from the
// plugin at any time, including stream messages like `Drop`.
std::thread::Builder::new()
    .name(format!(
        "plugin interface reader ({})",
        source.identity.name()
    ))
    .spawn(move || {
        if let Err(err) = manager.consume_all((reader, encoder)) {
            log::warn!("Error in PluginInterfaceManager: {err}");
        }
        // If the loop has ended, drop the manager so everyone disconnects and then wait for the
        // child to exit
        drop(manager);
        let _ = child.wait();
    })
    .map_err(|err| ShellError::PluginFailedToLoad {
        msg: format!("Failed to spawn thread for plugin: {err}"),
    })?;
```

Then nushell will send `PluginCall::Signature` message to plugin.  It's worth to look at the definition of `PlugininterfaceManager`
```rust
#[derive(Debug)]
#[doc(hidden)]
pub struct PluginInterfaceManager {
    // ...
    /// Receiver for plugin call subscriptions
    plugin_call_subscription_receiver: mpsc::Receiver<(PluginCallId, PluginCallState)>,
    // ...
}

impl PluginInterfaceManager {
    pub fn new(
        source: Arc<PluginSource>,
        writer: impl PluginWrite<PluginInput> + 'static,
    ) -> PluginInterfaceManager {
        let (subscription_tx, subscription_rx) = mpsc::channel();

        PluginInterfaceManager {
            state: Arc::new(PluginInterfaceState {
                source,
                plugin_call_id_sequence: Sequence::default(),
                stream_id_sequence: Sequence::default(),
                plugin_call_subscription_sender: subscription_tx,
                error: OnceLock::new(),
                writer: Box::new(writer),
            }),
            plugin_call_subscription_receiver: subscription_rx,
        }
    }
}
```

As we can see, when we create a `PluginInterfaceManager`, it will give us a channel of plugin_call_subscription.  `plugin_call_subscription_receiver` will be used in `manager.consume_call`.  And `plugin_call_subscription_sender` will be used when we want to call plugin.  At that level, when we `write_plugin_call`, we *subscribe* the call to Manager.

```rust
fn write_plugin_call(
    &self,
    call: PluginCall<PipelineData>,
    ctrlc: Option<Arc<AtomicBool>>,
    context_rx: mpsc::Receiver<Context>,
) -> Result<
    (
        PipelineDataWriter<Self>,
        mpsc::Receiver<ReceivedPluginCallMessage>,
    ),
    ShellError,
> {
    let id = self.state.plugin_call_id_sequence.next()?;
    let (tx, rx) = mpsc::channel();
    // Convert the call into one with a header and handle the stream, if necessary
    let (call, writer) = match call {
        PluginCall::Signature => (PluginCall::Signature, Default::default()),
        // ...
    };
    // Register the subscription to the response, and the context
    self.state
        .plugin_call_subscription_sender
        .send((
            id,
            PluginCallState {
                sender: Some(tx),
                ctrlc,
                context_rx: Some(context_rx),
                remaining_streams_to_read: 0,
            },
        ))?;

    // Write request
    self.write(PluginInput::Call(id, call))?;
    self.flush()?;
    Ok((writer, rx))
}
```

As we can see, `write_plugin_call` returns a pair of writer and channel receiver, which means we need to get response from that receiver.  That is how `receive_plugin_call_response` works.  Given all of this, let's summarize how `get_signature` works:
1. Create a `PluginInterfaceManager`, which includes `plugin_call_subscription_sender/receiver` pair.  And receive messages in background.
2. Write `PluginCall::Signature` to 
