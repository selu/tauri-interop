# Tauri-Interop

[![Latest version](https://img.shields.io/crates/v/tauri-interop.svg)](https://crates.io/crates/tauri-interop)
[![Documentation](https://docs.rs/tauri-interop/badge.svg)](https://docs.rs/tauri-interop)
![License](https://img.shields.io/crates/l/tauri-interop.svg)

This crate tries to provide a general more enjoyable experience for developing tauri apps with a rust frontend.
> tbf it is a saner approach to write the app in a mix of js + rust, because the frameworks are more mature, there are
> way more devs who have experience with js and their respective frameworks etc...
> 
> but tbh... just because something is saner, it doesn't stop us from doing things differently ^ヮ^

Writing an app in a single language gives us the option of building a common crate/module which connects the backend and 
frontend. A common model itself can most of the time be easily compiled to both architectures (arch's) when the types 
are compatible with both. The commands on the other hand don't have an option to be compiled to wasm. Which means they
need to be handled manually or be called via a wrapper/helper each time. 

Repeating the implementation and handling for a function that is already defined properly seems to be a waste of time.
For that reason this crate provides the `tauri_interop::command` macro. This macro is explained in detail in the 
[command representation](#command-representation-hostwasm) section. This new macro provides the option to invoke the 
command in wasm and by therefore call the defined command in tauri. On the other side, when compiling for tauri in addition 
to the tauri logic, the macro provides the option to collect all commands in a single file via the invocation of the 
`tauri_interop::collect_commands` macro at the end of the file (see [command](#command-frontend--backend-communication)).

In addition, some quality-of-life macros are provided to ease some inconveniences when compiling to multiple arch's. See
the [QOL](#qol-macros) section.

**Feature `event`**:

Tauri has an [event](https://tauri.app/v1/guides/features/events) mechanic which allows the tauri side to communicate with
the frontend. The usage is not as intuitive and has to some inconveniences that make it quite hard to recommend. To 
improve the usage, this crate provides the derive-marcos `Event`, `Emit` and `Listen`. The `Event` macro is just a 
conditional wrapper that expands to `Emit` for the tauri compilation and `Listen` for the wasm compilation. It is 
the intended way to use this feature. The usage is explained in the documentation of the `Event` macro. 
section.

## Basic usage:

> **Disclaimer**:
>
> Some examples in this documentation can't be executed with doctests due to
> the required wasm target and tauri modified environment (see [withGlobalTauri](https://tauri.app/v1/api/config/#buildconfig.withglobaltauri))

### Command (Frontend => Backend Communication)
> For more examples see [cmd.rs](./test-project/api/src/cmd.rs) in test-project

The newly provides macro `tauri_interop::command` does two things:
- it provides the function with two macros which are used depending on the targeted architecture
  - `tauri_interop::binding` is used when compiling to `wasm`
  - `tauri::command` is used otherwise
- additionally it provides the possibility to collect all defined commands via `tauri_interop::collect_commands!()` 
  - for more info see [collect commands](#collect-commands))
  - the function is not generated when targeting `wasm`

The generated command can then be used in `wasm` like the following:
```rust , ignore
#[tauri_interop::command]
fn greet(name: &str, _handle: tauri::AppHandle) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}

fn main() {
    console_log::init_with_level(log::Level::Info).unwrap();

    wasm_bindgen_futures::spawn_local(async move { 
        let greetings = greet("frontend").await;
        log::info!("{greetings}");
    });
}
```

**Command representation Host/Wasm (and a bit background knowledge)**

- the returned type of the wasm binding should be 1:1 the same type as send from the "backend" 
  - technically all commands need to be of type `Result<T, E>` because there is always the possibility of a command 
    getting called, that isn't registered in the context of tauri
    - when using `tauri_interop::collect_commands!()` this possibility is fully™️ removed
    - for convenience, we ignore that possibility, and even if the error occurs it will be logged into the console
- all arguments with `tauri` in their name (case-insensitive) are removed as argument in a defined command
  - that includes `tauri::*` usages and `Tauri` named types
  - the crate itself provides type aliases for tauri types usable in a command (see [type_aliases](./src/command/type_aliases.rs))
- most return types are automatically determined
  - when using a return type with `Result` in the name, the function will also return a `Result`
  - that also means, if you create a type alias for `Result<T, E>` and don't include `Result` in the name of the alias, 
    it will not map the `Result` correctly

#### Collect commands

The `tauri_invoke::collect_commands` macro generates a `get_handlers` function in the current mod, which calls the 
`tauri::generate_handler` macro with all function which are annotated with the `tauri_interop::command` macro. The 
function is only generated for tauri and not for wasm.

Due to technical limitations we sadly can't combine multiple `get_handlers` functions. This limitation comes to the 
underlying mechanic. The `tauri::generate_handler` macro generates a function which consumes `tauri::Invoke` as single 
parameter. Because it fully consumes the given parameter we can't call multiple handlers with it. In addition, the 
`Builder::invoke_handler` function, which usually consumes the generated `tauri::generate_handler` can't be called 
twice without losing the previous registered commands.

Because of this limitation for splitting commands into multiple files it is recommended to create a root mod for the 
command which includes other command mod's. The functions in the included mods need to be public and re-imported into 
the root mod. With these prerequisites the `tauri_invoke::collect_commands` can be called at the end of the file, which
generates the usual `get_handlers` function, but with all "commands" defined inside the others mods.

For an example see the [test-project/api/src/command.rs](test-project/api/src/command.rs).

### QOL macros

This crate also adds some quality-of-life macros. These are intended to ease the drawbacks of compiling to
multiple architectures.

#### Conditional `use`
Because most crates are not intended to be compiled to wasm and most wasm crates are not intended to be compiled to
the host-triplet they have to be excluded in each others compile process. The usual process to exclude uses for a certain
architecture would look something like this:

```rust
#[cfg(not(target_family = "wasm"))]
use tauri::AppHandle;

#[tauri_interop::command]
pub fn empty_invoke(_handle: AppHandle) {}
```

**General usage:**

With the help of `tauri_interop::host_usage!()` and `tauri_interop::wasm_usage!()` we don't need to remember which
attribute we have to add and can just convert the above to the following:

```rust
tauri_interop::host_usage! {
    use tauri::AppHandle;
}

#[tauri_interop::command]
pub fn empty_invoke(_handle: AppHandle) {}
```

**Multiple `use` usage:**

When multiple `use` should be excluded, they need to be separated by a single pipe (`|`). For example:

```rust
tauri_interop::host_usage! {
    use tauri::State;
    | use std::sync::RwLock; 
}

#[tauri_interop::command]
pub fn empty_invoke(_state: State<RwLock<String>>) {}
```
