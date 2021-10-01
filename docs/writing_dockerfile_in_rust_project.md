## Writing dockerfile in rust project
It's normally that when you write a rust project, you want to make it running on k8s, or build a small runtime container.

So here it's my journey about writing rust project dockerfile result in a small runtime image(***alpine based***).

### TL;DR
[Here](#6-final-dockerfile) is the final dockerfile example, you can just take it and customize to what you want.

### 1. setup base image
Fortunally, we have some very useful base image for rust, you can find [rust](https://hub.docker.com/_/rust) image if your project is compiled in stable rust.  If you want to build with nightly rust, here is [nightly rust image](https://hub.docker.com/r/rustlang/rust) you can use directly.

So, let's starts with it:
```dockerfile
FROM rust:lately

WORKDIR /app
```

Or nightly rust:
```dockerfile
FROM rust:nightly

WORKDIR /app
```

### 2. install musl-tools
Because I want to make my final images works on [alpine](https://hub.docker.com/_/alpine), which need my code compiled in `x86_64-unknown-linux-musl` platform, here is the shell command to add rustup target:
```shell
rustup target add x86_64-unknown-linux-musl
```

It's also recommended to install `musl-tools` to make many crates (like textwrap) compile successfully.
```shell
apt update
apt install musl-tools -y
```

So, let's just add it to a our dockerfile:

```dockerfile
FROM rust:lately

WORKDIR /app

RUN apt update
RUN apt install musl-tools -y

RUN rustup target add x86_64-unknown-linux-musl
```

### 3. make up vendor locally
Because `cargo vendor` can vendor all dependencies locally, so when you compile your project, `cargo` don't need to `update crates.io` and `download dependency code again`.
```shell
cargo vendor
```

After running cargo vendor, it will show up the following message:

```text
To use vendored sources, add this to your .cargo/config.toml for this project:

[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

Just do it for now, please note that the `.cargo` is located in your project root, rather than `$HOME/.cargo`.

Note: personally, I don't want to make output `vendor` directory to be tracking by `git`.  Because it can be too large, so just make up vendor locally is ok.

### 4. setup our source code and compile.
In this stage you just need to copy your source code, Cargo.toml, Cargo.lock, vendor to build directory:
```shell
RUN cargo new project_name
COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
COPY ./src src

# build with x86_64-unknown-linux-musl to make it runs on alpine.
RUN cargo install --path . --target=x86_64-unknown-linux-musl
```

Here is dockerfile:
```shell

FROM rust:lately

WORKDIR /app

RUN apt update
RUN apt install musl-tools -y

RUN rustup target add --toolchain nightly x86_64-unknown-linux-musl

# setup source code and compile.
RUN cargo new project_name
COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
COPY ./src src

# build with x86_64-unknown-linux-musl to make it run with alpine.
RUN cargo +nightly install --path . --target=x86_64-unknown-linux-musl
```

### 5. make runtime image smaller
So here, everything can be done for now.  But I want to make up my runtime image smaller, this image is bacally too large (1G+).  So we can use [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/), and copy our final binary to new base image.
```dockerfile
FROM rustlang:lately as builder
...
...
RUN cargo +nightly install --path . --target=x86_64-unknown-linux-musl

# second stage.
FROM alpine:3.14
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### 6. Final dockerfile
Ok, everything is done, here is final working dockerfile:
```dockerfile
FROM rust:lately

WORKDIR /app

RUN apt update
RUN apt install musl-tools -y

RUN rustup target add x86_64-unknown-linux-musl

# setup source code and compile.
RUN cargo new project_name
COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
COPY ./src src

# build with x86_64-unknown-linux-musl to make it run with alpine.
RUN cargo +nightly install --path . --target=x86_64-unknown-linux-musl

# second stage.
FROM alpine:3.14
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### Some extras(Only If you find out it's too slow to execute rustup target add x86_64-unknown-linux-musl)
When you execute `rustup target add x86_64-unknown-linux-musl`, maybe it's too slow to downloading component.  You can try to setup `RUSTUP_DIST_SERVER` and `RUSTUP_UPDATE_ROOT` environment variable.

Like I'm in China, these two env variable can set to this:
```shell
ENV RUSTUP_DIST_SERVER https://mirrors.ustc.edu.cn/rust-static
ENV RUSTUP_UPDATE_ROOT https://mirrors.ustc.edu.cn/rust-static/rustup
```

Which can speed up my component download speed.  Here is the final after set up these two environment variable:

```dockerfile
FROM rust:lately

WORKDIR /app

ENV RUSTUP_DIST_SERVER https://mirrors.ustc.edu.cn/rust-static
ENV RUSTUP_UPDATE_ROOT https://mirrors.ustc.edu.cn/rust-static/rustup

RUN apt update
RUN apt install musl-tools -y

RUN rustup target add x86_64-unknown-linux-musl

# setup source code and compile.
RUN cargo new project_name
COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
COPY ./src src

# build with x86_64-unknown-linux-musl to make it run with alpine.
RUN cargo +nightly install --path . --target=x86_64-unknown-linux-musl

# second stage.
FROM alpine:3.14
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### Special thanks and references
- [keng42](https://github.com/keng42) teach me something about docker, and provide a [github cd](https://github.com/WindSoilder/hors/pull/54) file.
- https://zhuanlan.zhihu.com/p/356274853 inspires me about `cargo vendor`, the source code compile steps is borrowed from here.
- https://huangjj27.gitlab.io/posts/rust-mirror/ inspired me about `RUSTUP_DIST_SERVER` and `RUSTUP_UPDATE_ROOT`.
