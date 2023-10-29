# blog_os

This Project is to lean how to create a small operating system in the [Rust programming language].

[Rust programming language]: https://www.rust-lang.org/

## precondition

The first step in creating our own operating system kernel is to create a Rust executable that does not link the standard library. This makes it possible to run Rust code on the [bare metal] without an underlying operating system.

### Building for a Bare Metal Target

By default Rust tries to build an executable that is able to run in your current system environment. For example, if you're using Windows on `x86_64`, Rust tries to build an `.exe` Windows executable that uses `x86_64` instructions. This environment is called your "host" system.

To describe different environments, Rust uses a string called [_target triple_]. You can see the target triple for your host system by running `rustc --version --verbose`:

[_target triple_]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple

```
rustc 1.73.0 (cc66ad468 2023-10-03)
binary: rustc
commit-hash: cc66ad468955717ab92600c770da8c1601a4ff33
commit-date: 2023-10-03
host: aarch64-apple-darwin
release: 1.73.0
LLVM version: 17.0.2
```

The above output is from a ` aarch64` Mac system. We see that the `host` triple is `aarch64-apple-darwin`, which includes the CPU architecture (`aarch64`), the vendor (`apple`), the operating system (`darwin`).  In the case of linux systems, it might also have a [ABI] (`gnu`) at the very end.

[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface

By compiling for our host triple, the Rust compiler and the linker assume that there is an underlying operating system such as Linux or Windows that uses the C runtime by default, which causes the linker errors. So, to avoid the linker errors, we can compile for a different environment with no underlying operating system.

An example of such a bare metal environment is the `thumbv7em-none-eabihf` target triple, which describes an [embedded] [ARM] system. The details are not important, all that matters is that the target triple has no underlying operating system, which is indicated by the `none` in the target triple. To be able to compile for this target, we need to add it in rustup:

[embedded]: https://en.wikipedia.org/wiki/Embedded_system
[ARM]: https://en.wikipedia.org/wiki/ARM_architecture

```shell
rustup target add thumbv7em-none-eabihf
```

This downloads a copy of the standard (and core) library for the system. 

### Build

Now we can build our freestanding executable for this target:

```shell
cargo build --target thumbv7em-none-eabihf
```

By passing a `--target` argument we [cross compile] our executable for a bare metal target system. Since the target system has no operating system, the linker does not try to link the C runtime and our build succeeds without any linker errors.

[cross compile]: https://en.wikipedia.org/wiki/Cross_compiler
