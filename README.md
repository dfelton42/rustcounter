# rustcounter - A Linux kernel module 

A Linux kernel module that exposes an atomic counter at /dev/rustcounter, written in safe Rust. Every write increments the counter; every read returns the current value as ASCII.

## Demo

```
$ sudo cat /dev/rustcounter
0
$ echo bump | sudo tee /dev/rustcounter > /dev/null
$ sudo cat /dev/rustcounter
1
$ echo bump | sudo tee /dev/rustcounter > /dev/null
$ echo bump | sudo tee /dev/rustcounter > /dev/null
$ sudo cat /dev/rustcounter
3
```
## What this is

rustcounter is a character device kernel module written in safe Rust. It demonstrates how Rust's type system makes data races in kernel code structurally impossible — the compiler forces you to use `AtomicU64::fetch_add` instead of a plain `count++` that would silently lose increments under concurrency.

## Requirements

- Ubuntu 26.04 LTS (kernel 7.x with `CONFIG_RUST=y`)
- `rustc-1.93`
- `linux-headers-$(uname -r)`
- `linux-lib-rust-$(uname -r)`

## Build & Run

```bash
git clone https://github.com/dfelton42/rustcounter
cd rustcounter
make
sudo insmod rustcounter.ko
sudo cat /dev/rustcounter
```

## Code Tour

- `module!` macro — registers the module with the kernel
- `RustCounter::init` — loads the module and registers the `/dev/rustcounter` misc device
- `write_iter` — drains the user's bytes and calls `COUNT.fetch_add(1, Ordering::SeqCst)`
- `read_iter` — reads `COUNT` atomically and returns it as ASCII, using `CONSUMED` to signal EOF to `cat`

## Why AtomicU64 instead of Mutex?

The only operation needed on the counter is "add 1 and read." The CPU has a single instruction for this (`LOCK XADD` on x86). A `Mutex<u64>` would also be correct but adds unnecessary overhead — two memory fences and a potential scheduler interaction on contention. `AtomicU64` is the minimal correct synchronization primitive for this use case.

## Future Work

- A `RESET` command — writing `RESET\n` sets the counter to 0
- Per-process counts tracked by PID
- A `/proc/rustcounter` read-only view that doesn't affect `CONSUMED`
- Saturating counter at `u64::MAX` instead of wrapping

## License

Licensed GPL-2.0 to match the Linux kernel.
