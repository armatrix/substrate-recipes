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

### 事件

外部调用函数执行，执行的结果可能是成功也可能是失败。我们需要一个标志来知晓调用的函数已经成功的执行。

#### 声明事件

我们还是通过trait来进行约束

```rust
pub trait Trait: system::Trait {
    type Event: From<Event> + Into<<Self as system::Trait>::Event>;
}
```

`From` 和 `Into` trait实现类型转换，对比`AsRef`和`AsMut`，`From`和`Into`返回的结果会获取参数所有权。

在`decl_module!`宏中声明一个函数供后续调用

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {

        // 声明这个函数
        fn deposit_event() = default;

        // --snip--
    }
}
```

具体的事件内容通过`decl_event!`宏来展开

```rust
// 将事件与账户绑定
decl_event!(
    pub enum Event<T> where AccountId = <T as system::Trait>::AccountId {
        EmitInput(AccountId, u32),
    }
);
```

注意0号块是不产生事件的，所以在创世块生成期间调用方法，方法中的event是不会广播出去的。

#### 调用事件

```rust
Self::deposit_event(RawEvent::EmitInput(user, new_number));
```

#### Runtime 中的改动

实现刚刚声明的trait

```rust
// 你也可以将Trait声明为某一模块下的具有更强标识性的命名，但很快随着拆包的明确，你会发现似乎这样更好
impl <pkg>::Trait for Runtime {
    type Event = Event;
}
```

和可调用函数与存储（这时候不是还没介绍到存储？）一样，在`construct_runtime!`宏中，我们也要做些许修改

```rust
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        // --snip--
        GenericEvent: generic_event::{Module, Call, Event<T>},
    }
);
```

### 存储map

类似rust中的HashMap，存储一个key-value对。

#### 声明一个StorageMap

```rust
// 这一类的声明是在SotrageMap trait下，不需要显示的声明 use， decl_storage!宏为我们做了这个
decl_storage! {
    trait Store for Module<T: Trait> as SimpleMap {
        SimpleMap get(fn simple_map): map hasher(blake2_128_concat) T::AccountId => u32;
    }
}
```

- `SimpleMap` —— map的名字
- `get(fn simple_map)` —— getter函数的名字
- `: map hasher(blake2_128_concat)` —— 声明这个map所采用的Hash函数（BLAKE2定位是目前安全系数最高的哈希函数）
- `T::AccountId => u32` —— map的key是Trait下的AccountId， value是u32类型的值

#### API

```rust
// user 是一个accountID
// insert 插入值  
<SimpleMap<T>>::insert(&user, entry);

// get 获取key对应的值 
let entry = <SimpleMap<T>>::get(account);

// take 获取key对应的值，并将其从map中删除 
let entry = <SimpleMap<T>>::take(&user);

// contains_key 判断一个key是否在map中 
<SimpleMap<T>>::contains_key(&user);

// remove 删除
<SimpleMap<T>>::remove(&user);

```

#### Tip

看下下面几段代码

```rust
let new_value = original_value.checked_add(add_this_val).ok_or(Error::<T>::MaxValueReached)?;
<SimpleMap<T>>::insert(&user, new_value);
```

```rust
ensure!(<SimpleMap<T>>::contains_key(&account), Error::<T>::NoValueStored);
let entry = <SimpleMap<T>>::get(account);
```

边界问题的检查要尤为注意。在处理数值类型的时候，历史上发生过美链攻击事件，利用了一个数值的溢出漏洞，后续造成不小的影响。同样的，很多场景下没有值和零值的概念也是不同的，对数值的处理我们要采用上面这种方式。

### 缓存和存储调用

降低存储调用的成本非常重要，我们可以通过rust的一些特性，来尽可能的减少对存储的调用。

假定你已经比较熟悉所有权问题，这里要三个trait放在一起看，Sized, Clone和Copy

#### Sized

Rust中几乎所有的类型都是有固定大小的，u8占8个字节，Vec<T> 包含一个指向堆上可变大小缓冲区的指针（包含容量和长度）。
其他的像str，共享引用（如可以指向任意大小的[u8]切片的 &[u8]) 是非固定大小的，这一类是属于大小不确定的集合，属于非固定大小的类型。
还有一类是说对特型的实现。其他多数语言我们接口，特型是共享行为的抽象，到了具体要怎么实现，这个是不确定的，所以通常这里也是非固定大小的类型。

Rust不能在变量中存储非固定大小的值，也不能将其作为参数。这时候往往是需要通过一些如&str 或 Box<T> 这样本身是固定大小的指针先指向它们，再对它们进行使用。这一类的指针始终是Fat pointer，占两个字宽，包含了指向切片的指针和切片的长度。特型目标也包含一个指向方法实现的虚拟表的指针。

Rust隐式的将泛型变量限制为使用Sized类型。当我们写 `struct some_struct<T> { --snip-- }` ，Rust会将其理解为 `struct some_struct<T: Sized> { --snip-- }` 

#### Clone

克隆一个值通常涉及创建该值所拥有一切内容的副本和可能存在的内存分配（如clone_from在原始的堆缓冲区如果有足够的容量可以满足需求的话，往往无需分配或释放内存）。

定义：

```rust
// std::clone::Clone
trait Clone: Sized {
	fn clone(&self) -> Self;
	fn clone_from(&mut self, source: &Self) {
	  *self = source.clone()
	}
}
```

原始类型bool和i32实现了Clone，容器类型String, Vec<T> 和HashMap也实现了Clone。

#### Copy

往往赋值会转移值，并不是复制值。转移值更有利于跟踪变量所拥有的资源。例外的是，不拥有任何资源的简单类型可以是Copy类型，这种类型的赋值会生成值的副本，而不是转移值并让原始变量变成未初始化。

标记特型（std::marker::Copy)，定义：

```rust
trait Copy: clone{} 
```

实现

```
impl Copy for MyType{ }
```

Rust只允许类型在字节对字节的深度复制能够满足要求的情况下实现Copy。对于一些拥有任意资源的如OS句柄，不能实现Copy，冲突的特型有Drop，当需要标明一种特别的清理代码的方式的时候，也应该需要一种特别的Copy的方法。

#### Cache Multiple Calls

对runtime storage的调用通常会有一些额外的开销，应该尽量减少使用，这个时候我们就可以通过使用上面的一些trait来实现。

```rust
decl_storage! {
    trait Store for Module<T: Trait> as StorageCache {
        // copy type
        SomeCopyValue get(fn some_copy_value): u32;

        // clone type
        KingMember get(fn king_member): T::AccountId;
        GroupMembers get(fn group_members): Vec<T::AccountId>;
    }
}
```

#### Copy Types

对于Copy类型，我们直接复用值即可，下面展示一段冗余的代码

```rust
fn increase_value_no_cache(origin, some_val: u32) -> DispatchResult {
    let _ = ensure_signed(origin)?;
    let original_call = <SomeCopyValue>::get();
    let some_calculation = original_call.checked_add(some_val).ok_or("addition overflowed1")?;
    // 这个调用时不需要的，我们浪费了一次对runtime的调用
    let unnecessary_call = <SomeCopyValue>::get();
    // u32实现了copy，这个里面我们直接用上一个变量中储存的值即可
    let another_calculation = some_calculation.checked_add(unnecessary_call).ok_or("addition overflowed2")?;
    <SomeCopyValue>::put(another_calculation);
    let now = <system::Module<T>>::block_number();
    Self::deposit_event(RawEvent::InefficientValueChange(another_calculation, now));
    Ok(())
}
```

正确的写法应该更正为

```rust
fn increase_value_w_copy(origin, some_val: u32) -> DispatchResult {
    let _ = ensure_signed(origin)?;
    let original_call = <SomeCopyValue>::get();
    let some_calculation = original_call.checked_add(some_val).ok_or("addition overflowed1")?;
    // 这里直接使用copy的值即可
    let another_calculation = some_calculation.checked_add(original_call).ok_or("addition overflowed2")?;
    <SomeCopyValue>::put(another_calculation);
    let now = <system::Module<T>>::block_number();
    Self::deposit_event(RawEvent::BetterValueChange(another_calculation, now));
    Ok(())
}
```

#### Clone Types

如果类型不是Copy，我们使用的Clone的代价也要小于调用runtime存储的代价

```rust
fn swap_king_with_cache(origin) -> DispatchResult {
    let new_king = ensure_signed(origin)?;
    let existing_king = <KingMember<T>>::get();
    // prefer to clone previous call rather than repeat call unnecessarily
    let old_king = existing_king.clone();

    // only places a new account if
    // (1) the existing account is not a member &&
    // (2) the new account is a member
    ensure!(!Self::is_member(&existing_king), "current king is a member so maintains priority");
    ensure!(Self::is_member(&new_king), "new king is not a member so doesn't get priority");

    // <no (unnecessary) storage call here>
    // place new king
    <KingMember<T>>::put(new_king.clone());

    Self::deposit_event(RawEvent::BetterKingSwap(old_king, new_king));
    Ok(())
}
```

简单的说，我们可以通过rust的语言机制来减少对runtime storage的调用。