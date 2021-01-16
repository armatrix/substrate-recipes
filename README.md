# substrate-recipes

## 前置知识

背景，框架，模块介绍

## Pallets

### 打印节点日志

使用日志用来debug或是做一些基础的展示

#### No Std

```rust
#![cfg_attr(not(feature = "std"), no_std)]
```

我们在使用substrate的runtime时，需要编译成WASM，可能有些rust的官方库并不支持这种，上面的代码旨在告诉编译器不要使用标准库，除非我们明确的指出要使用标准库。

#### Imports

每个 `pallet` 需要导入一些基本的包，如 [`frame-support`](https://substrate.dev/rustdocs/v2.0.0/frame_support/index.html) 和 [`frame-system`](https://substrate.dev/rustdocs/v2.0.0/frame_system/index.html) ，更复杂的pallet可能会导入很多包。在我们第一个例子hello-substrate 中我们使用了如下的一些包。

```rust
use frame_support::{ decl_module, dispatch::DispatchResult, debug };
use frame_system::{ self as system, ensure_signed };
use sp_runtime::print;
```

#### Test

少不了测试，小的模块这里的一个通俗的约定是我们建立一个 tests.rs 文件来放相关的测试代码。

#### Configuration Trait

substate的代码中，建立抽象的概念尤为重要（[coding is all about abstraction](https://youtu.be/05H4YsyPA-U?t=1789)）。对于pallet的一些配置的约束我们通过一个tarit来实现，该trait就叫做`Trait` (这种对新手很不友好，maybe `cfg_trait` is better) 。该trait必须存在，即使我们什么也没有在内部声明。

```rust
pub trait Trait: system::Trait {}
```

#### Dispatchable Calls

通过一个名为[`decl_module!`](https://substrate.dev/rustdocs/v2.0.0/frame_support/macro.decl_module.html) 的宏，我们在链上声明了一系列可供外部调用的函数，统称为 `Dispatchable calls` 。 在示例中，我们仅使用一个函数。

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {

        /// A function that says hello to the user by printing messages to the node log
        #[weight = 10_000]
        pub fn say_hello(origin) -> DispatchResult {
            // --snip--
        }

        // More dispatchable calls could go here
    }
}
```

#### Weight Annotations

在例子中我们可以看到一个`weight`的注解，weight的声明会影响调用该函数的手续费，后续的章节会做详细的介绍。

#### Inside a Dispatchable Call

```rust
pub fn say_hello(origin) -> DispatchResult {
    // 调用方要是一个标准的密钥对账户，函数调用成功返回调用方标识
    let caller = ensure_signed(origin)?;

    // 打印
    print("Hello World");
    // 查看变量中的内容, 该宏类似rust标准库的 println！
    debug::info!("Request sent by: {:?}", caller);

    // 返回成功
    Ok(())
}
```

在substare的开发中，这里我们看到了一个比较重要的范式，即对要进行的操作先进行权限的验证。某一个用户调用了链上的某个函数，他是否具备该权限呢？如他是不是一个合法的账户，账户里的钱是不是够的，所做的操作，如转账等，他是不是具备相关的权限呢？诸如此类的操作是很常见的。"**Verify first, write last**"

#### Printing from the Runtime

一般的rust代码我们可以通过调用标准库中 `println！` 宏来向终端（stdout）打印。 但之前我们说过，substrate的runtime会编译成WASM和普通的二进制，并没有使用标准库，所以这里使用该宏会报错。当我们有这样的需求的时候，我们可以使用类似示例代码中的方法——`sp_runtime::print()`，这里便不会编译成WSAM，使用标准库，执行对应的逻辑，如标准IO等。使用这个方法必须要实现[`Printable` trait](https://substrate.dev/rustdocs/v2.0.0/sp_runtime/traits/trait.Printable.html)，substrate基础的一些类型均实现了该trait。 

要想看到该输出，还需要在启动时传递 `-lruntime=debug` 参数，如示例的代码，启动的命令为:

```sh
./target/release/kitchen-node --dev -lruntime=debug
```

注意：

使用WASM时，要额外增加初始化[RuntimeLogger](https://substrate.dev/rustdocs/v2.0.0/frame_support/debug/struct.RuntimeLogger.html)的步骤。