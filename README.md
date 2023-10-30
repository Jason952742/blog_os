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

### Installing Rust Nightly
Rust has three release channels: _stable_, _beta_, and _nightly_. The Rust Book explains the difference between these channels really well, so take a minute and [check it out](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#choo-choo-release-channels-and-riding-the-trains). For building an operating system, we will need some experimental features that are only available on the nightly channel, so we need to install a nightly version of Rust.

To manage Rust installations, I highly recommend [rustup]. It allows you to install nightly, beta, and stable compilers side-by-side and makes it easy to update them. With rustup, you can use a nightly compiler for the current directory by running `rustup override set nightly`. Alternatively, you can add a file called `rust-toolchain` with the content `nightly` to the project's root directory. You can check that you have a nightly version installed by running `rustc --version`: The version number should contain `-nightly` at the end.

[rustup]: https://www.rustup.rs/

```shell
rustup override set nightly
```


## Running our Kernel

Now that we have an executable that does something perceptible, it is time to run it. First, we need to turn our compiled kernel into a bootable disk image by linking it with a bootloader. Then we can run the disk image in the [QEMU] virtual machine or boot it on real hardware using a USB stick.



### Creating a Bootimage

To turn our compiled kernel into a bootable disk image, we need to link it with a bootloader. As we learned in the [section about booting], the bootloader is responsible for initializing the CPU and loading our kernel.

[section about booting]: #the-boot-process

Instead of writing our own bootloader, which is a project on its own, we use the [`bootloader`] crate. This crate implements a basic BIOS bootloader without any C dependencies, just Rust and inline assembly. To use it for booting our kernel, we need to add a dependency on it:

[`bootloader`]: https://crates.io/crates/bootloader

```toml
# in Cargo.toml

[dependencies]
bootloader = "0.9.23"
```

Adding the bootloader as a dependency is not enough to actually create a bootable disk image. The problem is that we need to link our kernel with the bootloader after compilation, but cargo has no support for [post-build scripts].

[post-build scripts]: https://github.com/rust-lang/cargo/issues/545

To solve this problem, we created a tool named `bootimage` that first compiles the kernel and bootloader, and then links them together to create a bootable disk image. To install the tool, go into your home directory (or any directory outside of your cargo project) and execute the following command in your terminal:

```
cargo install bootimage
```

For running `bootimage` and building the bootloader, you need to have the `llvm-tools-preview` rustup component installed. You can do so by executing `rustup component add llvm-tools-preview`.

After installing `bootimage` and adding the `llvm-tools-preview` component, you can create a bootable disk image by going back into your cargo project directory and executing:

```
> cargo bootimage
```

We see that the tool recompiles our kernel using `cargo build`, so it will automatically pick up any changes you make. Afterwards, it compiles the bootloader, which might take a while. Like all crate dependencies, it is only built once and then cached, so subsequent builds will be much faster. Finally, `bootimage` combines the bootloader and your kernel into a bootable disk image.

After executing the command, you should see a bootable disk image named `bootimage-blog_os.bin` in your `target/x86_64-blog_os/debug` directory. You can boot it in a virtual machine or copy it to a USB drive to boot it on real hardware. (Note that this is not a CD image, which has a different format, so burning it to a CD doesn't work).

#### How does it work?
The `bootimage` tool performs the following steps behind the scenes:

- It compiles our kernel to an [ELF] file.
- It compiles the bootloader dependency as a standalone executable.
- It links the bytes of the kernel ELF file to the bootloader.

[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[rust-osdev/bootloader]: https://github.com/rust-osdev/bootloader

When booted, the bootloader reads and parses the appended ELF file. It then maps the program segments to virtual addresses in the page tables, zeroes the `.bss` section, and sets up a stack. Finally, it reads the entry point address (our `_start` function) and jumps to it.


### Booting it in QEMU

We can now boot the disk image in a virtual machine. To boot it in [QEMU], execute the following command:

[QEMU]: https://www.qemu.org/

```
> qemu-system-x86_64 -drive format=raw,file=target/x86_64-blog_os/debug/bootimage-blog_os.bin
```

We see that our "Hello World!" is visible on the screen.

### Real Machine

It is also possible to write it to a USB stick and boot it on a real machine, **but be careful** to choose the correct device name, because **everything on that device is overwritten**:

```
> dd if=target/x86_64-blog_os/debug/bootimage-blog_os.bin of=/dev/sdX && sync
```

Where `sdX` is the device name of your USB stick.

After writing the image to the USB stick, you can run it on real hardware by booting from it. You probably need to use a special boot menu or change the boot order in your BIOS configuration to boot from the USB stick. Note that it currently doesn't work for UEFI machines, since the `bootloader` crate has no UEFI support yet.

### Using `cargo run`

To make it easier to run our kernel in QEMU, we can set the `runner` configuration key for cargo:

```toml
# in .cargo/config.toml

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```

The `target.'cfg(target_os = "none")'` table applies to all targets whose target configuration file's `"os"` field is set to `"none"`. This includes our `x86_64-blog_os.json` target. The `runner` key specifies the command that should be invoked for `cargo run`. The command is run after a successful build with the executable path passed as the first argument. See the [cargo documentation][cargo configuration] for more details.

The `bootimage runner` command is specifically designed to be usable as a `runner` executable. It links the given executable with the project's bootloader dependency and then launches QEMU. See the [Readme of `bootimage`] for more details and possible configuration options.

[Readme of `bootimage`]: https://github.com/rust-osdev/bootimage

Now we can use `cargo run` to compile our kernel and boot it in QEMU.


