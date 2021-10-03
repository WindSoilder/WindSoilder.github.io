# Writing dockerfile in rust project
When you writing a rust project, maybe you want to build a small runtime container, then you can run it in k8s or something else you like.

Here it's my journey about writing rust project dockerfile result in a small runtime image(***alpine based*** and ***distroless/cc-debian based***).

## TL;DR
There are three final dockerfile example, you can just take it and customize to what you want.
1. (Only works well for non workspace based project) Vendor based alpine runtime image is [here](#6-final-dockerfile)
2. (Only works well for non workspace based project) Vendor based cc-debian runtime image is [here](#7-additional-dockerfile)
3. (More general and recommended) Not vendor based cc-debian runtime image is [here](#not-vendor-based-dockerfile-based-on-cargo-chef)

The idea to take less build time:
- build only dependencies first, so when only my source code changed, no need to compile dependencies again.
- In vendor based build(1 and 2), we use locally pre-generated vendor.  Then we don't need to update `crates.io` source and `download dependency code` when compiling, less build time we take.

## Vendor based dockerfile
### 1. setup base image
Fortunally, we have some very useful base image for rust, you can find [rust](https://hub.docker.com/_/rust) image if your project is compiled in stable rust.  If you want to build with nightly rust, here is [nightly rust image](https://hub.docker.com/r/rustlang/rust) you can use directly.

So, let's start with it:
```dockerfile
FROM rust:latest

WORKDIR /app
```

Or nightly rust:
```dockerfile
FROM rustlang/rust:nightly

WORKDIR /app
```

Because I want it to run inside alpine, I can directly use *alpine* directly.  So the dockerfile can be this:
```dockerfile
FROM rust:alpine3.14

WORKDIR /app
```

Or nightly rust:
```dockerfile
FROM rustlang/rust:nightly-alpine

WORKDIR /app
```

*alpine* relative image contains rust *x86_64-unknown-linux-musl* target, we can use it directly to compile our code.

### 2. install musl-tools
It's also recommended to install `musl-tools` to make many crates (like textwrap) compile successfully.
```shell
apk add musl-dev
```

So, let's just add it to our dockerfile:

```dockerfile
FROM rust:alpine3.14

WORKDIR /app

RUN apk add musl-dev
```

### 3. make up vendor locally
[cargo vendor](https://doc.rust-lang.org/cargo/commands/cargo-vendor.html) can vendor all dependencies locally, when you compile your project, `cargo` don't need to `update crates.io` and `download dependency code` again, so we can take less build time.
```shell
cargo vendor
```

After running cargo vendor, it will show up some messages like this:

```text
To use vendored sources, add this to your .cargo/config.toml for this project:

[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

Just follow it, please note that the `.cargo` is located in your project root, rather than `$HOME/.cargo`.

Note: personally, I don't want to make output `vendor` directory to be tracked by `git`.  Because it can easily be too large, just make up vendor locally is ok.

### 4. setup our source code and compile.
In this stage you just need to copy your source code, Cargo.toml, Cargo.lock, vendor to build directory:
```dockerfile
COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
COPY ./src src

# build with x86_64-unknown-linux-musl to make it runs on alpine.
RUN cargo install --path . --target=x86_64-unknown-linux-musl
```

Here is a big shortcoming: in vendor based approach, if I change source code, the whole dependency needs to be compiled again, which can cause a lot of compile time.  So we would like to make use of docker cache, what we can do is:
1. create an empty project, compile only dependency, then,
2. remove the dummy empty project, copy our source code to working directory, then,
3. build the project.

Here is dockerfile:
```dockerfile

FROM rust:alpine3.14

WORKDIR /app

RUN apk add musl-dev

# create a new empty project
RUN cargo init

COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
# build dependencies, when my source code changes, this build can be cached, we don't need to compile dependency again.
RUN cargo build
# remove the dummy build.
RUN cargo clean -p $project_name_specified_in_cargo

COPY ./src src

# build with x86_64-unknown-linux-musl to make it runs on alpine.
RUN cargo install --path . --target=x86_64-unknown-linux-musl
```

### 5. make runtime image smaller
So here, everything can be done for now.  But I want to make up my runtime image smaller, this image is bacally too large (1G+).  So we can use [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/), and copy our final binary to new base image.
```dockerfile
FROM rustlang:latest as builder
...
...
RUN cargo install --path . --target=x86_64-unknown-linux-musl

# second stage.
FROM alpine:3.14
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### 6. Final dockerfile
Ok, everything is done, here is final working dockerfile:
```dockerfile
FROM rust:alpine3.14 as builder

WORKDIR /app

RUN apk add musl-dev

# create a new empty project
RUN cargo init

COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
# build dependencies, when my source code changes, this build can be cached, we don't need to compile dependency again.
RUN cargo build
# remove the dummy build.
RUN cargo clean -p $project_name_specified_in_cargo

# build with x86_64-unknown-linux-musl to make it run with alpine.
RUN cargo install --path . --target=x86_64-unknown-linux-musl

# second stage.
FROM alpine:3.14
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### 7. Additional dockerfile
Thanks for [trusch2](https://www.reddit.com/user/trusch2/) comment on [reddit](https://www.reddit.com/r/rust/comments/pz2pxn/simple_intro_about_writing_dockerfile_in_rust/), we can use [gcr.io/distroless/cc-debian11](https://gcr.io/distroless/cc-debian11) to have a small runtime on debian image.

The benefit trusch2 mention about using [gcr.io/distroless/cc-debian11](https://gcr.io/distroless/cc-debian11) are:
- its small (~20MB)
- there is no shell and nothing else an attacker could use when trying to escape the container
- you can just build within a normal debian env and then use cc-debian11 as base without the dance around using libmusl instead of libc.

So, here is another final dockerfile based on debian:
```dockerfile
FROM rust:latest as builder

WORKDIR /app

# create a new empty project
RUN cargo init

COPY ./.cargo .cargo
COPY ./vendor vendor
COPY Cargo.toml Cargo.lock ./
# build dependencies, when my source code changes, this build can be cached, we don't need to compile dependency again.
RUN cargo build
# remove the dummy build.
RUN cargo clean -p $project_name_specified_in_cargo

RUN cargo install --path .

# second stage.
FROM gcr.io/distroless/cc-debian11
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### Restriction
This vendor based doesn't work well in workspace based project.  For non vendor based approach, [cargo chef](https://github.com/LukeMathWalker/cargo-chef) is a valuable thing to consider.

## Not vendor based dockerfile based on cargo-chef
[cargo-chef](https://crates.io/crates/cargo-chef) can Cache the dependencies of your Rust project and speed up your Docker builds.  The page also gives us a very good documentation about how to write `cargo-chef` based dockerfile.

Here is the final dockerfile which uses `cargo-chef`:
```dockerfile
FROM rustlang/rust:latest AS chef
# We only pay the installation cost once,
# it will be cached from the second build onwards
RUN cargo install cargo-chef

WORKDIR app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# Build dependencies - this is the caching Docker layer!
RUN cargo chef cook --release --recipe-path recipe.json

# Build application
COPY . .
RUN cargo install --path .

# We do not need the Rust toolchain to run the binary!
FROM gcr.io/distroless/cc-debian11
COPY --from=builder /usr/local/cargo/bin/* /usr/local/bin
```

### Extras (only needs when your network to crates.io is too slow)
When your network to `crates.io` is too slow, you can use a custom crates.io source.  To do this, create a file in the project root directory `.cargo/cargo.toml`.  Like I'm in China, I can use tuna source instead of `crates.io` source.

Fill the `.cargo/cargo.toml` file like this:
```toml
[source.crates-io]
replace-with = "tuna"

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

Then just change your dockerfile a little:
```dockerfile
FROM rustlang/rust:latest AS chef

COPY ./.cargo .cargo
# We only pay the installation cost once,
# it will be cached from the second build onwards
RUN cargo install cargo-chef

...
...
...
```

### Special thanks and references
- [keng42](https://github.com/keng42) teach me something about docker, and provide a [github cd](https://github.com/WindSoilder/hors/pull/54) file.
- https://zhuanlan.zhihu.com/p/356274853 inspires me about `cargo vendor`, the source code compile steps is borrowed from here.
- so many suggestions from [reddit](https://www.reddit.com/r/rust/comments/pz2pxn/simple_intro_about_writing_dockerfile_in_rust/)
