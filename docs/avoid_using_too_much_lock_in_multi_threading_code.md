# Avoid using too much lock in multi threading code

Let's say that I have 2 threads, and one shared variable. Thread 1 will periodically(every 15 microseconds) increment
the shared variable by 1.

Thread 2 will periodically(every 600 milliseconds) access this shared variable.

here is the struct definition:

```rust
struct State {
    counter: u64,
}

impl State {
    pub fn new() -> Self {
        State { counter: 0 }
    }

    pub fn incr(&mut self) {
        self.counter += 1;
    }

    pub fn get_count(&self) -> u64 {
        self.counter
    }
}
```

## Using lock

One normal way to do this is using `Lock` with `Arc`, to control shared variable between threads.

```rust
use std::sync::{Arc, Mpsc, mpsc};
use std::thread;

fn first_try() {
    let state = Arc::new(Mutex::new(State::new()));

    // Thread 1: sleep 15ms and increment counter.
    let state1 = state.clone();
    thread::spawn(move || loop {
        thread::sleep(Duration::from_micros(15));
        state1.lock().unwrap().incr();
    });

    // Thread 2: sleep 600ms and try access state2, after access 120 times, stop it.
    let state2 = state.clone();
    let x = thread::spawn(move || {
        for _ in 0..120 {
            thread::sleep(Duration::from_millis(600));
            state2.lock().unwrap().get_count();
        }
    });

    let _ = x.join();
}
```

The problem is: while using lock, our thread1 doing incr operation per 15 micro seconds, which is more frequency
than the other reader thread. And because of this, we will acquire too much lock, which seems unnecessary.

## Using channel to avoid too much lock

To avoid the problem like this, we can use channel. We can introduce another thread (numbered as 3), which takes
ownership of our state, and handle message from thread 1 and thread 2.

thread 1 will sleep and send out message to thread 3, to update the shared variable.

thread 2 will sleep and send out message to thread 3, to let it get shared variable's status.

```rust
use std::sync::{Arc, Mpsc, mpsc};
use std::thread;


enum Message {
    Tick,
    Get,
    Close,
}

fn second_try() {
    let (sender, receiver) = channel::<Message>();

    // Thread 1: sleep and send out message to thread 3 to update shared variable.
    let sender2 = sender.clone();
    thread::spawn(move || loop {
        thread::sleep(Duration::from_micros(15));
        let _ = sender2.send(Message::Tick);
    });

    // Thread 2: sleep and send another message to thread 3, to let it get shared variable's status.
    let sender3 = sender.clone();
    let x = thread::spawn(move || {
        for _ in 0..120 {
            thread::sleep(Duration::from_millis(600));
            sender3.send(Message::Get);
        }
        sender3.send(Message::Close);
    });

    // Thread 3 owns state.
    thread::spawn(move || {
        loop {
            let msg = receiver.recv().unwrap();
            match msg {
                Message::Tick => state.incr(),
                Message::Close => break,
                Message::Another => {
                    state.get_count();
                }
            }
        }
    });

    x.join().unwrap();
}
```

In this way, we can avoid acquire too much lock, and the relative writer task can have better performance.
