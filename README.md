### substrate-recipes

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

### Set

没有内置的Set结构，这里我们分别通过Vector和 Map来实现

比如说我们现在有一个需求，来确保说链上的某一类型的账户最多有多少个

```rust
// 这里我们定义一个上限，最多有16个用户，并提供一些函数，如增加账户，删除账户，来保证这个数据的合法性，下面分别通过vector和map来实现
pub const MAX_MEMBERS: u32 = 16;
```

#### Vector

```rust
decl_storage! {
    trait Store for Module<T: Trait> as VecSet {
        // 一个存储AccountId的集合
        Members get(fn members): Vec<T::AccountId>;
    }
}
```

这里我们说为了提高查找效率，我们保证Vec是有序的，这样方便我们使用二分查找

##### 增加账户

```rust
pub fn add_member(origin) -> DispatchResult {
    let new_member = ensure_signed(origin)?;

    let mut members = Members::<T>::get();
    ensure!(members.len() < MAX_MEMBERS, Error::<T>::MembershipLimitReached);

    // 二分查找来判断账户是否已经存在，复杂度 O(log n).
    match members.binary_search(&new_member) {
        // 如果找到了，直接返回账户已经存在的错误
        Ok(_) => Err(Error::<T>::AlreadyMember.into()),
        // 没有找到的话，说明账户还不存在，这个时候插入这个账户
        Err(index) => {
            members.insert(index, new_member.clone());
            Members::<T>::put(members);
            Self::deposit_event(RawEvent::MemberAdded(new_member));
            Ok(())
        }
    }
}
```

##### 删除账户

```rust
fn remove_member(origin) -> DispatchResult {
    let old_member = ensure_signed(origin)?;

    let mut members = Members::<T>::get();

    // 这里面的逻辑仍然是，我们先查找这个账户是否存在
    match members.binary_search(&old_member) {
        // 找到了就删除
        Ok(index) => {
            members.remove(index);
            Members::<T>::put(members);
            Self::deposit_event(RawEvent::MemberRemoved(old_member));
            Ok(())
        },
        // 账户不存在的逻辑
        Err(_) => Err(Error::<T>::NotMember.into()),
    }
}
```

#### Map

```rust
decl_storage! {
    trait Store for Module<T: Trait> as VecMap {
        // 存储所有账户的集合
        Members get(fn members): map hasher(blake2_128_concat) T::AccountId => ();
        // 这个里面因为map并不存储本身的长度，对元素的个数我们额外使用一个字段来存储
        MemberCount: u32;
    }
}
```

##### 增加账户

```rust
fn add_member(origin) -> DispatchResult {
    let new_member = ensure_signed(origin)?;

    let member_count = MemberCount::get();
    ensure!(member_count < MAX_MEMBERS, Error::<T>::MembershipLimitReached);

    //  O(1)
    ensure!(!Members::<T>::contains_key(&new_member), Error::<T>::AlreadyMember);

    // 新增账户
    Members::<T>::insert(&new_member, ());
    MemberCount::put(member_count + 1); //注意这里的上限
    Self::deposit_event(RawEvent::MemberAdded(new_member));
    Ok(())
}
```

##### 删除账户

```rust
fn remove_member(origin) -> DispatchResult {
    let old_member = ensure_signed(origin)?;

    ensure!(Members::<T>::contains_key(&old_member), Error::<T>::NotMember);

    Members::<T>::remove(&old_member);
    MemberCount::mutate(|v| *v -= 1);
    Self::deposit_event(RawEvent::MemberRemoved(old_member));
    Ok(())
}
```

#### 性能对比

|            | Vector实现                                     | Map实现                                        |
| ---------- | ---------------------------------------------- | ---------------------------------------------- |
| 查找       | DB Reads: O(1) Decoding: O(n) Search: O(log n) | DB Reads: O(1)                                 |
| 更新       | DB Writes: O(1) Encoding: O(n)                 | DB Reads: O(1) Encoding: O(1) DB Writes: O(1)  |
| 迭代器操作 | DB Reads: O(1) Decoding: O(n) Processing: O(n) | DB Reads: O(n) Decoding: O(n) Processing: O(n) |

### Double Maps

我们怎么样对快速删除map[map]中的一组元素（map），这里面提供了`remove_prefix ` 方法。

```rust
pub type GroupIndex = u32; // this is Encode (which is necessary for double_map)

decl_storage! {
    trait Store for Module<T: Trait> as Dmap {
        /// Member score (double map)
        MemberScore get(fn member_score):
            double_map hasher(blake2_128_concat) GroupIndex, hasher(blake2_128_concat) T::AccountId => u32;
        /// Get group ID for member
        GroupMembership get(fn group_membership): map hasher(blake2_128_concat) T::AccountId => GroupIndex;
        /// For fast membership checks, see check-membership recipe for more details
        AllMembers get(fn all_members): Vec<T::AccountId>;
    }
}
```

删除

```rust
fn remove_group_score(origin, group: GroupIndex) -> DispatchResult {
    let member = ensure_signed(origin)?;

    let group_id = <GroupMembership<T>>::get(member);
    // check that the member is in the group
    ensure!(group_id == group, "member isn't in the group, can't remove it");

    // remove all group members from MemberScore at once
    <MemberScore<T>>::remove_prefix(&group_id);

    Self::deposit_event(RawEvent::RemoveGroup(group_id));
    Ok(())
}
```

### 存储struct

#### 声明

```rust
#[derive(Encode, Decode, Default, Clone, PartialEq)]
pub struct MyStruct {
    some_number: u32,
    optional_number: Option<u32>,
}
```

这里面使用derive派生的一些接口有些要手动实现。使用Encode和Decode要导入包`use frame_support::codec::{Encode, Decode};`

#### 使用泛型

我们可以将其中的字段使用泛型来声明，

```rust
#[derive(Encode, Decode, Clone, Default, RuntimeDebug)]
pub struct InnerThing<Hash, Balance> {
    number: u32,
    hash: Hash,
    balance: Balance,
}
```

这里我们可以使用type关键字来起别名

```rust
type InnerThingOf<T> = InnerThing<<T as system::Trait>::Hash, <T as balances::Trait>::Balance>;
```

#### 存储中的struct

```rust
decl_storage! {
    trait Store for Module<T: Trait> as NestedStructs {
        InnerThingsByNumbers get(fn inner_things_by_numbers):
            map hasher(blake2_128_concat) u32 => InnerThingOf<T>;
        SuperThingsBySuperNumbers get(fn super_things_by_super_numbers):
            map hasher(blake2_256) u32 => SuperThing<T::Hash, T::Balance>;
    }
}
```

同样的方式我们可以将结构体作为一个map的值来进行存储

```rust
fn insert_inner_thing(origin, number: u32, hash: T::Hash, balance: T::Balance) -> DispatchResult {
    let _ = ensure_signed(origin)?;
    let thing = InnerThing {
                    number,
                    hash,
                    balance,
                };
    <InnerThingsByNumbers<T>>::insert(number, thing);
    Self::deposit_event(RawEvent::NewInnerThing(number, hash, balance));
    Ok(())
}
```

#### 嵌套结构体

和其他语言类似，内嵌的结构体中的泛型也要加上

```rust
#[derive(Encode, Decode, Default, RuntimeDebug)]
pub struct SuperThing<Hash, Balance> {
    super_number: u32,
    inner_thing: InnerThing<Hash, Balance>,
}
```

### 环形缓冲队列

这里我们给了一个列子，实现一个环形缓冲队列，后续我们可以参考这种范式来实现其他自己所需要的数据结构。

#### 定义该数据的约束

```rust
pub trait RingBufferTrait<Item>
where
    Item: Codec + EncodeLike,
{
    // 存储所有的改动
    fn commit(&self);
    // 向队列中添加元素
    fn push(&mut self, i: Item);
    // 从队列中弹出元素
    fn pop(&mut self) -> Option<Item>;
    // 返回队列是否是空的
    fn is_empty(&self) -> bool;
}
```

#### 抽象数据结构

```rust
// 队列最长为2^16
type DefaultIdx = u16;
pub struct RingBufferTransient<Item, B, M, Index = DefaultIdx>
where
    Item: Codec + EncodeLike,  // 约束每个元素符合存储的数据需求
    B: StorageValue<(Index, Index), Query = (Index, Index)>,  // 需要存储的范围
    M: StorageMap<Index, Item, Query = Item>,  // 存储的对象
    Index: Codec + EncodeLike + Eq + WrappingOps + From<u8> + Copy, // 队列的索引
{
    start: Index,
    end: Index,
    _phantom: PhantomData<(Item, B, M)>, //保证声明周期
}
```

#### 构造方法

```rust
impl<Item, B, M, Index> RingBufferTransient<Item, B, M, Index>
where
    Item: Codec + EncodeLike,  // 约束每个元素符合存储的数据需求
    B: StorageValue<(Index, Index), Query = (Index, Index)>,  // 需要存储的起讫
    M: StorageMap<Index, Item, Query = Item>,  // 存储的对象
    Index: Codec + EncodeLike + Eq + WrappingOps + From<u8> + Copy, // 队列的索引
{
    pub fn new() -> RingBufferTransient<Item, B, M, Index> {
        let (start, end) = B::get();
        RingBufferTransient {
            start, 
          	end, 
         		_phantom: PhantomData,
        }
    }
}
```

#### 实现RingBuffer trait

```rust
impl<Item, B, M, Index> RingBufferTrait<Item> for RingBufferTransient<Item, B, M, Index>
where 
    Item: Codec + EncodeLike,
    B: StorageValue<(Index, Index), Query = (Index, Index)>,
    M: StorageMap<Index, Item, Query = Item>,
    Index: Codec + EncodeLike + Eq + WrappingOps + From<u8> + Copy,
{
    fn commit(&self) {
        B::put((self.start, self.end));
    }
  
  	fn is_empty(&self) -> bool {
        self.start == self.end
    }
  
    fn push(&mut self, item: Item) {
      M::insert(self.end, item);
      // 判断当前元素是不是队列的最后一个元素
      let next_index = self.end.wrapping_add(1.into());
      if next_index == self.start {
        // 当前元素是最后一个元素
        self.start = self.start.wrapping_add(1.into());
      }
      self.end = next_index;
    }
  
    fn pop(&mut self) -> Option<Item> {
          if self.is_empty() {
              return None;
          }
          let item = M::take(self.start);
          self.start = self.start.wrapping_add(1.into());

          item.into()
      }
}
```

#### 实现Drop

```rust
impl<Item, B, M, Index> Drop for RingBufferTransient<Item, B, M, Index>
where 
    Item: Codec + EncodeLike,
    B: StorageValue<(Index, Index), Query = (Index, Index)>,
    M: StorageMap<Index, Item, Query = Item>,
    Index: Codec + EncodeLike + Eq + WrappingOps + From<u8> + Copy,
{
    fn drop(&mut self) {
        <Self as RingBufferTrait<Item>>::commit(self);
    }
}
```

### Basic Token 

在BTC的UTXO之后，基于账户余额的账本技术较为常见，本质上是将其描述为状态转换函数：APPLY (State, Transaction) -> NewState。 

这里我们假定我们构建一个账户到余额的映射关系，模拟一个系统，系统总发行token数我们通过硬编码来固定，发行的方式暂定为谁第一个调用我们的初始化函数，那么这些token就归谁。

#### Storage Item

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Token {
        // 某个账户下的余额
        pub Balances get(get_balance): map hasher(blake2_128_concat) T::AccountId => u64;
				// 总供给
        pub TotalSupply get(total_supply): u64 = 21000000;
				// 是否初始化
        Init get(is_init): bool;
    }
}
```

#### Events and Errors

```rust
decl_event!(
    pub enum Event<T>
    where
        AccountId = <T as system::Trait>::AccountId,
    {
        // token被初始化
        Initialized(AccountId),
        // 在两个用户间转账成功
        Transfer(AccountId, AccountId, u64), // (from, to, value)
    }
);

decl_error! {
    pub enum Error for Module<T: Trait> {
        // 已经有人调用了初始化的函数
        AlreadyInitialized,
        // 转账失败
        InsufficientFunds,
    }
}
```

#### 初始token

可以考虑下其他的一些初始方式（genesis config, claims process, lockdrop etc.)

```rust
fn init(origin) -> DispatchResult {
	let sender = ensure_signed(origin)?;
	ensure!(!Self::is_init(), <Error<T>>::AlreadyInitialized);

	<Balances<T>>::insert(sender, Self::total_supply());

	Init::put(true);
	Ok(())
}
```

#### 转账

```rust
fn transfer(_origin, to: T::AccountId, value: u64) -> DispatchResult {
    let sender = ensure_signed(_origin)?;
    let sender_balance = Self::get_balance(&sender);
    let receiver_balance = Self::get_balance(&to);

    // 计算两方余额，注意这里的错误处理，对一些可能导致的panic情况我们要处理好
    let updated_from_balance = sender_balance.checked_sub(value).ok_or(<Error<T>>::InsufficientFunds)?;
    let updated_to_balance = receiver_balance.checked_add(value).expect("Entire supply fits in u64; qed");

    // 更新状态树
    <Balances<T>>::insert(&sender, updated_from_balance);
    <Balances<T>>::insert(&to, updated_to_balance);

    Self::deposit_event(RawEvent::Transfer(sender, to, value));
    Ok(())
}
```

### 常量设置

通常对整个系统而言的常量通常我们称之为元信息的一部分，需要在runtime中设置，在不同的pallet中使用时，需要像下面这样使用

```rust
use frame_support::traits::Get;

pub trait Trait: system::Trait {
    type Event: From<Event> + Into<<Self as system::Trait>::Event>;

    // 我们在trait中对两个配置项进行约束
    type MaxAddend: Get<u32>;
    type ClearFrequency: Get<Self::BlockNumber>;
}
```

为了能在runtime的元信息中展示，在 `decl_module!` 宏中，我们通常将类似的的声明写在如下的位置

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        fn deposit_event() = default;
				// 紧跟着event
        const MaxAddend: u32 = T::MaxAddend::get();
        const ClearFrequency: T::BlockNumber = T::ClearFrequency::get();
        // --snip--
    }
}
```

#### 示例

我们声明一个存储单一值的，通过我们的runtime中的配置来约束它的最大大小，给它一个初始化的方法，让它每隔若干个块就清零，再给他一个可以增加任意值的方法。也就是说，在以出块为时间单位的一段时间内，每隔一些块，它的最大值会有个上限

##### 配置

首先我们要在runtime中做相应的配置

```rust
parameter_types! {
  	// 允许的最大值
    pub const MaxAddend: u32 = 1738;
    // 清除的频率
    pub const ClearFrequency: u32 = 10;
}

impl constant_config::Trait for Runtime {
    type Event = Event;
    type MaxAddend = MaxAddend;
    type ClearFrequency = ClearFrequency;
}

```

##### 实现

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        SingleValue get(fn single_value): u32;
    }
}
```

初始化

```rust
fn on_finalize(n: T::BlockNumber) {
    if (n % T::ClearFrequency::get()).is_zero() {
        let c_val = <SingleValue>::get();
        <SingleValue>::put(0u32);
        Self::deposit_event(Event::Cleared(c_val));
    }
}
```

增加任意值的方法

```rust
fn add_value(origin, val_to_add: u32) -> DispatchResult {
    let _ = ensure_signed(origin)?;
    ensure!(val_to_add <= T::MaxAddend::get(), "value must be <= maximum add amount constant");

    let current_value = <SingleValue>::get();

    // 检查
    let result = match current_value.checked_add(val_to_add) {
        Some(r) => r,
        None => return Err(DispatchError::Other("addition overflowed")),
    };
    <SingleValue>::put(result);
    Self::deposit_event(Event::Added(current_value, val_to_add, result));
    Ok(())
}
```

### 众筹

这里展示一个链上众筹的功能。任何人都可以发起一个众筹活动，在一定时间内筹集一定的资金，到期未筹集到指定的资金额，这笔钱再退回到原有的账户中

#### 配置的约束trait

```rust
// pallet 的配置trait
pub trait Trait: system::Trait {
    /// The ubiquious Event type
    type Event: From<Event<Self>> + Into<<Self as system::Trait>::Event>;

    /// The currency in which the crowdfunds will be denominated
    type Currency: ReservableCurrency<Self::AccountId>;

    //
    type SubmissionDeposit: Get<BalanceOf<Self>>;

    // 众筹参与者提交的最小额度
    type MinContribution: Get<BalanceOf<Self>>;

    // 
    type RetirementPeriod: Get<Self::BlockNumber>;
}
```

#### 抽象数据结构

```rust
#[derive(Encode, Decode, Default, PartialEq, Eq)]
#[cfg_attr(feature = "std", derive(Debug))]
pub struct FundInfo<AccountId, Balance, BlockNumber> {
    // 众筹成功后用来接收的账户
    beneficiary: AccountId,
    // 抵押的总额
    deposit: Balance,
    // 众筹的总额
    raised: Balance,
    // 时间单位
    end: BlockNumber,
    /// Upper bound on `raised`
    goal: Balance,
}
```

简单的定义几个类型别名

```
pub type FundIndex = u32;

type AccountIdOf<T> = <T as system::Trait>::AccountId;
type BalanceOf<T> = <<T as Trait>::Currency as Currency<AccountIdOf<T>>>::Balance;
type FundInfoOf<T> = FundInfo<AccountIdOf<T>, BalanceOf<T>, <T as system::Trait>::BlockNumber>;
```

#### 存储

```rust
decl_storage! {
    trait Store for Module<T: Trait> as ChildTrie {
        // 获取资金的信息
        Funds get(fn funds):
            map hasher(blake2_128_concat) FundIndex => Option<FundInfoOf<T>>;

        // 众筹金额的笔数
        FundCount get(fn fund_count): FundIndex;

        // 额外的一些信息我们存储在 child trie 中，相比于我们默认的存储方式， child trie在处理删除或是需要证明（通过Merkle Proof）的时候会有优势
        // 可以查看下面的 impl<T: Trait> Module<T>
    }
}
```

#### API

```rust
// 增加新记录
pub fn contribution_put(index: FundIndex, who: &T::AccountId, balance: &BalanceOf<T>) {
    let id = Self::id_from_index(index);
    who.using_encoded(|b| child::put(&id, b, &balance));
}

// 查看某比筹款所在的trie
pub fn contribution_get(index: FundIndex, who: &T::AccountId) -> BalanceOf<T> {
    let id = Self::id_from_index(index);
    who.using_encoded(|b| child::get_or_default::<BalanceOf<T>>(&id, b))
}

// 从child trie中删除一组关联的筹款
pub fn contribution_kill(index: FundIndex, who: &T::AccountId) {
    let id = Self::id_from_index(index);
    who.using_encoded(|b| child::kill(&id, b));
}

// 从child trie中删除所有关联的筹款
pub fn crowdfund_kill(index: FundIndex) {
    let id = Self::id_from_index(index);
    child::kill_storage(&id);
}

// 通过捐献资金的id来生成唯一的childInfo
pub fn id_from_index(index: FundIndex) -> child::ChildInfo {
    let mut buf = Vec::new();
    buf.extend_from_slice(b"crowdfnd");
    buf.extend_from_slice(&index.to_le_bytes()[..]);

    child::ChildInfo::new_default(T::Hashing::hash(&buf[..]).as_ref())
}
```

#### Pallet Dispatchables

和其他的实现一样，这里依然是要做些像权限、边界的检查，和存储做一些交互，最后广播事件。这里要注意下激励的一些设置

```rust
#[weight = 10_000]
fn dispense(origin, index: FundIndex) {
    let caller = ensure_signed(origin)?;

    let fund = Self::funds(index).ok_or(Error::<T>::InvalidIndex)?;

    // 对众筹条件的检查
  
  	// 规定时间是否完成
    let now = <system::Module<T>>::block_number();
    ensure!(now >= fund.end, Error::<T>::FundStillActive);

    // 资金是否筹集成功
    ensure!(fund.raised >= fund.goal, Error::<T>::UnsuccessfulFund);
    let account = Self::fund_account_id(index);

    // 众筹条件满足后的一些逻辑
    let _ = T::Currency::resolve_creating(&fund.beneficiary, T::Currency::withdraw(
        &account,
        fund.raised,
        WithdrawReasons::from(WithdrawReason::Transfer),
        ExistenceRequirement::AllowDeath,
    )?);

    let _ = T::Currency::resolve_creating(&caller, T::Currency::withdraw(
        &account,
        fund.deposit,
        WithdrawReasons::from(WithdrawReason::Transfer),
        ExistenceRequirement::AllowDeath,
    )?);

```

### 可实例化的pallet

当我们需要在一条链上发行两种独立的加密货币；或者说某个用户作为买卖双方，我们想单独记录他作为买方和卖方的信用；以及对链的治理需要多个表现相似的治理方时，我们就可以创建这种pallet，这种pallet的创建和创建普通的pallet基本一样，其中 `decl_storage!` 这个必须要写，只有这样，实例对象才能被创建 ，注意下runtime中的写法。

substrate提供了一个例子，下面两个实例有自己的空间，包括存储、配置、事件等等

```rust
Council: collective::<Instance1>::{Module, Call, Storage, Origin<T>, Event<T>, Config<T>},
TechnicalCommittee: collective::<Instance2>::{Module, Call, Storage, Origin<T>, Event<T>, Config<T>}
```

#### Config Trait

首先还是定义 configuration trait

```rust
pub trait Trait<I: Instance>: system::Trait {
    type Event: From<Event<Self, I>> + Into<<Self as system::Trait>::Event>;
}
```

#### 定义存储

```rust
decl_storage! {
    trait Store for Module<T: Trait<I>, I: Instance> as TemplatePallet {
        ...
    }
}
```

#### 定义事件

```rust
decl_event!(
    pub enum Event<T, I> where AccountId = <T as system::Trait>::AccountId {
        ...
    }
}
```

#### 定义Module Struct

```rust
decl_module! {
 		// 别忘记初始化事件
  	fn deposit_event() = default;
 
    pub struct Module<T: Trait<I>, I: Instance> for enum Call where origin: T::Origin {
        ...
    }
}
```

#### runtime更改

每一个pallet实例都要单独实现（这不废话吗）,类似下面这样

```rust
impl template::Trait<template::Instance1> for Runtime {
    type Event = Event;
}
```

`construct_runtime!` 宏中的修改

```rust
FirstTemplate: template::<Instance1>::{Module, Call, Storage, Event<T>, Config},
```

#### 默认实例

可实例化的pallet的一个缺点是说，当我们仅需要一个实例的时候，这种开发范式不太友好，目前为止我们接触的都是这种，实际上当我们要指定一个具体的实例时，只需要对四个部分进行声明即可

```rust
pub trait Trait<I=DefaultInstance>: system::Trait {}
```

```rust
decl_storage! {
    trait Store for Module<T: Trait<I>, I: Instance=DefaultInstance> as TemplateModule {
        ...
    }
}
```

```rust
decl_module! {
    pub struct Module<T: Trait<I>, I: Instance = DefaultInstance> for enum Call where origin: T::Origin {
        ...
    }
}
```

```rust
decl_event!(
    pub enum Event<T, I=DefaultInstance> where ... {
        ...
    }
}
```

实现上述步骤之后，可以像使用其它的pallet那样使用它

#### 创世配置

一些pallet需要一些创世的配置，参照下面的实现（substrate node Collective pallet's chan_spec.rs）

```rust
GenesisConfig {
    ...
    collective_Instance1: Some(CouncilConfig {
        members: vec![],
        phantom: Default::default(),
    }),
    collective_Instance2: Some(TechnicalCommitteeConfig {
        members: vec![],
        phantom: Default::default(),
    }),
    ...
}
```

### 量化资源

这里我们说在我们构建一个pallet时，我们要注意对可调用的一些函数所消耗的代价的考量。不能较低成本的无限制的调用，这样极可能造成对网络的恶意攻击，设置合理的区间是很有必要的。

可以通过注解很方便的来设置权重

```rust
#[weight = <Some Weighting Instance>]
fn some_call(...) -> Result {
    // --snip--
}
```

如果我们需要给其设置一个固定数值的权重，这个就比较简单

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call {

        #[weight = 10_000]
        fn store_value(_origin, entry: u32) -> DispatchResult {
            StoredValue::put(entry);
            Ok(())
        }
}
```

其它的一些动态的需求，比如说我们想通过调用时入参的字节数或其它的某种规则来设置的话，需要自己实现。下面的例子我们通过对调用时入参的检测来调整权重。

##### 自定义权重

```rust
pub struct Conditional(u32);

impl WeighData<(&bool, &u32)> for Conditional {
    fn weigh_data(&self, (switch, val): (&bool, &u32)) -> Weight {
        if *switch {
            val.saturating_mul(self.0)
        } else {
            self.0
        }
    }
}
```

##### 实现trait约束

自定义权重需要实现两个trait：ClassifyDispatch 和 PaysFee

```rust
impl<T> ClassifyDispatch<T> for Conditional {
    fn classify_dispatch(&self, _: T) -> DispatchClass {
        // 这个里面走默认的逻辑 
        Default::default()
    }
}
```

```rust
impl PaysFee for Conditional {
    fn pays_fee(&self) -> bool {
        true
    }
}
```

### Charity and Imbalances

这里通过一个官方的示例向我们展示如何通过一个pallet来控制一笔资金（通过捐款获得）以及如何与链上资金的状态（发生铸币或销毁等）关联起来

#### Instantiate a pot

这里有个类似资金池的概念，管理这笔资金我们不在与个人的公私钥来关联，而是直接和pallet关联，需要导入 `sp-runtime` 下的 `ModuleId` 和 `AccountIdConversion`

```rust
use sp-runtime::{ModuleId, traits::AccountIdConversion};
```

导入该依赖后，我们声明一个八个字符长度的`Pallet_ID`作为资金池的表示，之所以对其限制时为了让后面我们可以通过调用特定的方法（`AccountIdConversion` trait 中的 `into_account()`）来生成 `AccountID`

```rust
const PALLET_ID: ModuleId = ModuleId(*b"Charity!");

impl<T: Trait> Module<T> {
  	//  管理资金的账户
    pub fn account_id() -> T::AccountId {
        PALLET_ID.into_account()
    }

    // 余额
    fn pot() -> BalanceOf<T> {
        T::Currency::free_balance(&Self::account_id())
    }
}
```

#### Receiving Funds

我们的资金池可以通过两种方式来获得资金

##### Donations

第一种就是普通的捐款，一笔简单的转账交易

```rust
fn donate( origin, amount: BalanceOf<T>) -> DispatchResult {
        let donor = ensure_signed(origin)?;

        let _ = T::Currency::transfer(&donor, &Self::account_id(), amount, AllowDeath);

        Self::deposit_event(RawEvent::DonationReceived(donor, amount, Self::pot()));
        Ok(())
}
```

##### Imbalances

第二种是说我们假定发生了铸币、销毁等影响总体货币平衡的情况下，所采取的措施。可能会存在一种治理的机制来对这样的资金进行操作。可以参考下

```rust
use frame_support::traits::{OnUnbalanced, Imbalance};
type NegativeImbalanceOf<T> = <<T as Trait>::Currency as Currency<<T as system::Trait>::AccountId>>::NegativeImbalance;

impl<T: Trait> OnUnbalanced<NegativeImbalanceOf<T>> for Module<T> {
    fn on_nonzero_unbalanced(amount: NegativeImbalanceOf<T>) {
        let numeric_amount = amount.peek();

        // Must resolve into existing but better to be safe.
      	// 这里可以扩展讲一下
        let _ = T::Currency::resolve_creating(&Self::account_id(), amount);

        Self::deposit_event(RawEvent::ImbalanceAbsorbed(numeric_amount, Self::pot()));
    }
}
```

### 定点运算

很难避免的就是定点数的计算，这里通过对substrate内置的定点数的结构和trait、内置的库和我们自己手动实现，结合实例做个对比，来说明为什么我们尽量要使用底层提供的库来执行相关的操作

#### Multiplicative Accumulators

##### Permill Accumulator

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        // --snip--

      	// 这里可以多做下解释
        PermillAccumulator get(fn permill_value): Permill = Permill::one();
    }
}
```

```rust
fn update_permill(origin, new_factor: Permill) -> DispatchResult {
    ensure_signed(origin)?;

    let old_accumulated = Self::permill_value();

    // [0, 1] 永远不可能溢出，无需检测
    let new_product = old_accumulated.saturating_mul(new_factor);

    PermillAccumulator::put(new_product);

    Self::deposit_event(Event::PermillUpdated(new_factor, new_product));
    Ok(())
}
```

#### Substrate-fixed Accumulator

讲的啥东西

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        // --snip--

        /// Substrate-fixed accumulator, value starts at 1 (multiplicative identity)
        FixedAccumulator get(fn fixed_value): U16F16 = U16F16::from_num(1);
    }
}
```

```rust
fn update_fixed(origin, new_factor: U16F16) -> DispatchResult {
    ensure_signed(origin)?;

    let old_accumulated = Self::fixed_value();

    // 乘法 检查溢出
    let new_product = old_accumulated.checked_mul(new_factor)
        .ok_or(Error::<T>::Overflow)?;

    FixedAccumulator::put(new_product);

    Self::deposit_event(Event::FixedUpdated(new_factor, new_product));
    Ok(())
}
```

#### Manual Accumulator

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        // --snip--

        /// Manual accumulator, value starts at 1 (multiplicative identity)
        ManualAccumulator get(fn manual_value): u32 = 1 << 16;
    }
}
```

```rust
fn update_manual(origin, new_factor: u32) -> DispatchResult {
    ensure_signed(origin)?;

    let old_accumulated : u64 = Self::manual_value() as u64;
    let new_factor_u64 : u64 = new_factor as u64;

    let raw_product : u64 = old_accumulated * new_factor_u64;

    let shifted_product : u64 = raw_product >> 16;

    if shifted_product > (u32::max_value() as u64) {
        return Err(Error::<T>::Overflow.into())
    }

    ManualAccumulator::put(shifted_product as u32);

    Self::deposit_event(Event::ManualUpdated(new_factor, shifted_product as u32));
    Ok(())
}
```

上面着几个对照着看下就好了，这个逻辑类似我们在计组里定点数那块的逻辑。使用对应的库即可，有兴趣可以自行研究

#### 复利

复利的结算在借贷这种业务中比较常见

##### 离散复利

这里我们模拟一个离散复利的计算，每隔若干块（这里是每十个）计算一次

定义存储

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        // --snip--

        // 账户的余额
        DiscreteAccount get(fn discrete_account): u64;
    }
}
```

更新的api

```rust
fn deposit_discrete(origin, val_to_add: u64) -> DispatchResult {
    ensure_signed(origin)?;

    let old_value = DiscreteAccount::get();

    // 更新
    DiscreteAccount::put(old_value + val_to_add);

    Self::deposit_event(Event::DepositedDiscrete(val_to_add));
    Ok(())
}
```



```rust
fn on_finalize(n: T::BlockNumber) {
    if (n % 10.into()).is_zero() {

        // 计算利息
        let interest = Self::discrete_interest_rate() * DiscreteAccount::get() * 10;

       	// 下面这个不生效，Percent对象没有实现 Mul<u64> trait，注意这个里面的要实现才能用类似交换律的逻辑，
      	// 可以引申介绍下rust的这块的逻辑
        // let interest = DiscreteAccount::get() * Self::discrete_interest_rate() * 10;

        // 更新余额
        let old_balance = DiscreteAccount::get();
        DiscreteAccount::put(old_balance + interest);

        Self::deposit_event(Event::DiscreteInterestApplied(interest));
    }
}
```

##### 连续复利

类似的需求让我们实现下连续复利

```rust
#[derive(Encode, Decode, Default)]
pub struct ContinuousAccountData<BlockNumber> {
    // 最近一次手动调整后的账户余额
    principal: I32F32,
    // 最近一次手动调整的区块数
    deposit_date: BlockNumber,
}
```

定义存储

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        // --snip--

        // 账户余额
        ContinuousAccount get(fn balance_compound): ContinuousAccountData<T::BlockNumber>;
    }
}
```

更新的api

```rust
// 这块的逻辑就是根据我们定义的上次更新的一些参数来做调整
fn deposit_continuous(origin, val_to_add: u64) -> DispatchResult {
    ensure_signed(origin)?;

    let current_block = system::Module::<T>::block_number();
    let old_value = Self::value_of_continuous_account(&current_block);

    ContinuousAccount::<T>::put(
        ContinuousAccountData {
            principal: old_value + I32F32::from_num(val_to_add),
            deposit_date: current_block,
        }
    );

    Self::deposit_event(Event::DepositedContinuous(val_to_add));
    Ok(())
}
```

余额

```rust
fn value_of_continuous_account(now: &<T as system::Trait>::BlockNumber) -> I32F32 {
    // 账户之前的状态
    let ContinuousAccountData{
        principal,
        deposit_date,
    } = ContinuousAccount::<T>::get();

    // 复利的具体计算
    let elapsed_time_block_number = *now - deposit_date;
    let elapsed_time_u32 = TryInto::try_into(elapsed_time_block_number)
        .expect("blockchain will not exceed 2^32 blocks; qed");
    let elapsed_time_i32f32 = I32F32::from_num(elapsed_time_u32);
    let exponent : I32F32 = Self::continuous_interest_rate() * elapsed_time_i32f32;
    let exp_result : I32F32 = exp(exponent)
        .expect("Interest will not overflow account (at least not until the learner has learned enough about fixed point :)");

    // 返回当前block number下的本金和利息 interest = principal * e ^ (rate * time)
    principal * exp_result
}
```

### Off-chain Workers

这块字面比较容易理解，交易这块还ok，其它的比如说和微信支付的一些交互之类的，实现上比如说我改了host，或者说需要对端提供一个幂等的操作之类的东西才能验证的时候该如何做呢？

先看下他这块的实现

#### Transactions

```shell
# 切换到该目录
cd nodes/kitchen-node

# 编译 OCW feature
cargo build --release --features ocw
```

这样在我们的[`nodes/kitchen-node/src/service.rs`](https://github.com/substrate-developer-hub/recipes/blob/master/nodes/kitchen-node/src/service.rs) 注入了账户的key

```rust
// 初始化用于离线签名用的种子
#[cfg(feature = "ocw")]
{
    keystore.write().insert_ephemeral_from_seed_by_type::<runtime::offchain_demo::crypto::Pair>(
        "//Alice", runtime::offchain_demo::KEY_TYPE
    ).expect("Creating key with account Alice should succeed.");
}
```

##### 链下工作机的声明周期

启动之后我们会看到这样的日志输出

```shell
Jan 19 19:58:14.436  INFO Kitchen Node
Jan 19 19:58:14.436  INFO ✌️  version 2.0.0-2f7b217-x86_64-macos
Jan 19 19:58:14.436  INFO ❤️  by Substrate DevHub <https://github.com/substrate-developer-hub>, 2019-2021
Jan 19 19:58:14.436  INFO 📋 Chain specification: Local Testnet
Jan 19 19:58:14.436  INFO 🏷  Node name: literate-suggestion-9160
Jan 19 19:58:14.436  INFO 👤 Role: FULL
Jan 19 19:58:14.436  INFO 💾 Database: RocksDb at /var/folders/2s/ryztw28123zf9bz8xyy7q28w0000gn/T/substratezSjwSx/chains/local_testnet/db
Jan 19 19:58:14.436  INFO ⛓  Native runtime: ocw-runtime-1 (ocw-runtime-1.tx1.au1)
Jan 19 19:58:14.501  INFO 🔨 Initializing Genesis block/state (state: 0xed6a…2a51, header-hash: 0xa1d7…7fe9)
Jan 19 19:58:14.506  WARN Using default protocol ID "sup" because none is configured in the chain specs
Jan 19 19:58:14.506  INFO 🏷  Local node identity is: 12D3KooWC2CcWmEmTrYTeNcDef99pwCyWLFDW77v3XCvjeD3DBNL (legacy representation: 12D3KooWC2CcWmEmTrYTeNcDef99pwCyWLFDW77v3XCvjeD3DBNL)
Jan 19 19:58:14.508  INFO 📦 Highest known block at #0
Jan 19 19:58:14.508  INFO 〽️ Prometheus server started at 127.0.0.1:9615
Jan 19 19:58:14.511  INFO Listening for new connections on 127.0.0.1:9944.
Jan 19 19:58:16.227  INFO Accepted a new tcp connection from 127.0.0.1:59528.
Jan 19 19:58:19.512  INFO 💤 Idle (0 peers), best: #0 (0xa1d7…7fe9), finalized #0 (0xa1d7…7fe9), ⬇ 0 ⬆ 0
```

这个时候我们一些交互端做一笔转账交易会发现输出类似下面的的日志

```shell
Jan 19 22:35:20.585  INFO 💤 Idle (0 peers), best: #0 (0x1522…c718), finalized #0 (0x1522…c718), ⬇ 0 ⬆ 0
Jan 19 22:35:22.463  INFO 🙌 Starting consensus session on top of parent 0x1522a4803b5df88931cfb7342bd24a75cfed14009d117dd8eda53f90b3cac718
Jan 19 22:35:22.470  INFO 🎁 Prepared block for proposing at 1 [hash: 0x1efe38107a0b3e84b6d75416514ef3f08114b604460ef5e35968714b36484d3b; parent_hash: 0x1522…c718; extrinsics (2): [0x3ef3…85cc, 0xb5e2…3ee7]]
Jan 19 22:35:22.473  INFO ✨ Imported #1 (0x1efe…4d3b)
Jan 19 22:35:22.473  INFO Instant Seal success: CreatedBlock { hash: 0x1efe38107a0b3e84b6d75416514ef3f08114b604460ef5e35968714b36484d3b, aux: ImportedAux { header_only: false, clear_justification_requests: false, needs_justification: false, bad_justification: false, needs_finality_proof: false, is_new_best: true } }
Jan 19 22:35:22.474  INFO Entering off-chain worker
Jan 19 22:35:22.478  INFO 🙌 Starting consensus session on top of parent 0x1efe38107a0b3e84b6d75416514ef3f08114b604460ef5e35968714b36484d3b
```

顾名思义，链下工作机所执行的处理逻辑是脱离链的，逻辑处理完成后需要再次通过链上来记录结果，可以传递给链上的结果包含三部分。

##### 发送签名交易

src: [`pallets/ocw-demo/src/lib.rs`](https://github.com/substrate-developer-hub/recipes/tree/master/pallets/ocw-demo/src/lib.rs)

###### 定义签名的模块

```rust
// 用来做签名
pub const KEY_TYPE: KeyTypeId = KeyTypeId(*b"demo");

pub mod crypto {
    use crate::KEY_TYPE;
    use sp_runtime::app_crypto::{app_crypto, sr25519};
    // -- snip --
    app_crypto!(sr25519, KEY_TYPE);
}
```

###### 配置trait

```rust
pub trait Trait: system::Trait + CreateSignedTransaction<Call<Self>> {
    // 用以标识链下工作及
    type AuthorityId: AppCrypto<Self::Public, Self::Signature>;
    // -- snip --
}
```

###### runtime实现链下工作机的trait

src: [`runtimes/ocw-runtime/src/lib.rs`](https://github.com/substrate-developer-hub/recipes/tree/master/runtimes/ocw-runtime/src/lib.rs)

定义需要签名的内容

```rust
// 签名的内容
pub type SignedExtra = (
    frame_system::CheckSpecVersion<Runtime>,
    frame_system::CheckTxVersion<Runtime>,
    frame_system::CheckGenesis<Runtime>,
    frame_system::CheckEra<Runtime>,
    frame_system::CheckNonce<Runtime>,
    frame_system::CheckWeight<Runtime>,
    pallet_transaction_payment::ChargeTransactionPayment<Runtime>,
);
```

实现签名交易需要实现三个trait

- `frame_system::offchain::CreateSignedTransaction`
- `frame_system::offchain::SigningTypes`
- `frame_system::offchain::SendTransactionTypes`

```rust
impl<LocalCall> frame_system::offchain::CreateSignedTransaction<LocalCall> for Runtime
where
    Call: From<LocalCall>,
{
    fn create_transaction<C: frame_system::offchain::AppCrypto<Self::Public, Self::Signature>>(
        call: Call,
        public: <Signature as sp_runtime::traits::Verify>::Signer,
        account: AccountId,
        index: Index,
    ) -> Option<(
        Call,
        <UncheckedExtrinsic as sp_runtime::traits::Extrinsic>::SignaturePayload,
    )> {
        let period = BlockHashCount::get() as u64;
        let current_block = System::block_number()
            .saturated_into::<u64>()
            .saturating_sub(1);
        let tip = 0;
        let extra: SignedExtra = (
            frame_system::CheckTxVersion::<Runtime>::new(),
            frame_system::CheckGenesis::<Runtime>::new(),
            frame_system::CheckEra::<Runtime>::from(generic::Era::mortal(period, current_block)),
            frame_system::CheckNonce::<Runtime>::from(index),
            frame_system::CheckWeight::<Runtime>::new(),
            pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip),
        );

        #[cfg_attr(not(feature = "std"), allow(unused_variables))]
        let raw_payload = SignedPayload::new(call, extra)
            .map_err(|e| {
                debug::native::warn!("SignedPayload error: {:?}", e);
            })
            .ok()?;

        let signature = raw_payload.using_encoded(|payload| C::sign(payload, public))?;

        let address = account;
        let (call, extra, _) = raw_payload.deconstruct();
        Some((call, (address, signature, extra)))
    }
}
```

```rust
impl frame_system::offchain::SigningTypes for Runtime {
    type Public = <Signature as sp_runtime::traits::Verify>::Signer;
    type Signature = Signature;
}
```

```rust
impl<C> frame_system::offchain::SendTransactionTypes<C> for Runtime
where
    Call: From<C>,
{
    type OverarchingCall = Call;
    type Extrinsic = UncheckedExtrinsic;
}
```

至此，所有的约束均已实现

###### 发送交易

src: [`pallets/ocw-demo/src/lib.rs`](https://github.com/substrate-developer-hub/recipes/tree/master/pallets/ocw-demo/src/lib.rs)

```rust
fn offchain_signed_tx(block_number: T::BlockNumber) -> Result<(), Error<T>> {
		// 因为我们通过dev指定的只有一个签名方，可以通过下面函数获取，如果有其它的更多的，需要通过 `with_filter()`来获取更多的操作
		//   ref: https://substrate.dev/rustdocs/v2.0.0/frame_system/offchain/struct.Signer.html
		let signer = Signer::<T, T::AuthorityId>::any_account();

		let number: u64 = block_number.try_into().unwrap_or(0) as u64;

		// `result` 的类型为 `Option<(Account<T>, Result<(), ()>)>`.
		//   - `None`: 没有任何账户足以发送此交易
		//   - `Some((account, Ok(())))`: 交易发送成功
		//   - `Some((account, Err(())))`: 交易发送失败
		let result = signer.send_signed_transaction(|_acct|
			// 调用链上函数
			Call::submit_number_signed(number)
		);

		// debug
		if let Some((acc, res)) = result {
      // 发送失败
			if res.is_err() {
				debug::error!("failure: offchain_signed_tx: tx sent: {:?}", acc.id);
				return Err(<Error<T>>::OffchainSignedTxError);
			}
			// 发送成功
			return Ok(());
		}

		// 没有任何账户能够实现此笔交易
		debug::error!("No local account available");
		Err(<Error<T>>::NoLocalAcctForSigning)
	}
```

##### 发送未签名交易

默认情况下是不允许的，实现这种需要我们来实现一些约束，更多的要考虑下这种的使用场景

src: [`pallets/ocw-demo/src/lib.rs`](https://github.com/substrate-developer-hub/recipes/tree/master/pallets/ocw-demo/src/lib.rs)

###### 实现未签名交易的trait

```rust
impl<T: Trait> frame_support::unsigned::ValidateUnsigned for Module<T> {
    type Call = Call<T>;

    fn validate_unsigned(_source: TransactionSource, call: &Self::Call) -> TransactionValidity {
        let valid_tx = |provide| ValidTransaction::with_tag_prefix("ocw-demo")
            .priority(T::UnsignedPriority::get()) 
            .and_provides([&provide]) 
            .longevity(3)  
            .propagate(true)
            .build();

        match call {
            Call::submit_number_unsigned(_number) => valid_tx(b"submit_number_unsigned".to_vec()),
            // -- snip --
            _ => InvalidTransaction::Call.into(),
        }
    }
}
```

这里我们可以看下valid transaction的定义

src: https://github.com/paritytech/substrate/blob/master/primitives/runtime/src/transaction_validity.rs

```rust
#[derive(Clone, PartialEq, Eq, Encode, Decode, RuntimeDebug)]
pub struct ValidTransaction {
	// 优先级
	pub priority: TransactionPriority,
	// 这里最好是结合一些具体的例子
	pub requires: Vec<TransactionTag>,
	// 参照实现的tag
	pub provides: Vec<TransactionTag>,
	// 用于验证的最小交易周期
	pub longevity: TransactionLongevity,
	// 该事务是否需要传播
	pub propagate: bool,
}
```

###### runtime中的实现

src: [`runtimes/ocw-runtime/src/lib.rs`](https://github.com/substrate-developer-hub/recipes/tree/master/runtimes/ocw-runtime/src/lib.rs)

```rust
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        //...snip
        OcwDemo: ocw_demo::{Module, Call, Storage, Event<T>, ValidateUnsigned},
    }
);
```

###### 发送交易

```rust
fn offchain_unsigned_tx(block_number: T::BlockNumber) -> Result<(), Error<T>> {
    let number: u64 = block_number.try_into().unwrap_or(0) as u64;
    let call = Call::submit_number_unsigned(number);

    // `submit_unsigned_transaction` returns a type of `Result<(), ()>`
    //   ref: https://substrate.dev/rustdocs/v2.0.0/frame_system/offchain/struct.SubmitTransaction.html#method.submit_unsigned_transaction
    SubmitTransaction::<T, Call<T>>::submit_unsigned_transaction(call.into())
        .map_err(|_| {
            debug::error!("Failed in offchain_unsigned_tx");
            <Error<T>>::OffchainUnsignedTxError
        })
}
```

##### 将签名内容作为未签名交易的一部分

这里最大的区别是，不会向签名人收取交易费用。我们将一笔交易签名后作为未签名交易发布，通过构造上面的valid transaction的结构进行对交易的合法性检测

src: [`pallets/ocw-demo/src/lib.rs`](https://github.com/substrate-developer-hub/recipes/tree/master/pallets/ocw-demo/src/lib.rs)

###### 发送的结构定义及trait实现

```rust
#[derive(Encode, Decode, Clone, PartialEq, Eq, RuntimeDebug)]
pub struct Payload<Public> {
    number: u64,
    public: Public
}

// 发送的内容要实现这个trait
impl <T: SigningTypes> SignedPayload<T> for Payload<T::Public> {
    fn public(&self) -> T::Public {
        self.public.clone()
    }
}
```

###### 将签名内容作为未签名交易的一部分进行交易

```rust
fn offchain_unsigned_tx_signed_payload(block_number: T::BlockNumber) -> Result<(), Error<T>> {

    let signer = Signer::<T, T::AuthorityId>::any_account();

    let number: u64 = block_number.try_into().unwrap_or(0) as u64;

    // `send_unsigned_transaction` 返回 `Option<(Account<T>, Result<(), ()>)>`.
    // 同发送签名交易一样
		//   - `None`: 没有任何账户足以发送此交易
		//   - `Some((account, Ok(())))`: 交易发送成功
		//   - `Some((account, Err(())))`: 交易发送失败
    if let Some((_, res)) = signer.send_unsigned_transaction(
      // 注意这里的闭包
      |acct| Payload { number, public: acct.public.clone() },
        Call::submit_number_unsigned_with_signed_payload
    ) {
        return res.map_err(|_| {
            debug::error!("Failed in offchain_unsigned_tx_signed_payload");
            <Error<T>>::OffchainUnsignedTxSignedPayloadError
        });
    }

   	// 没有任何账户能够实现此笔交易
    debug::error!("No local account available");
    Err(<Error<T>>::NoLocalAcctForSigning)
}
```

#### Http的接口操作

链下工作机的结果这里我们通过http来取回结果，使用json来进行数据交互

##### 发送请求

```rust
// 初始化一个外部的请求
let request = rt_offchain::http::Request::get(HTTP_REMOTE_REQUEST);

// timeout
let timeout = sp_io::offchain::timestamp()
    .add(rt_offchain::Duration::from_millis(FETCH_TIMEOUT_PERIOD));

// 发送请求
let pending = request
    .add_header("User-Agent", HTTP_HEADER_USER_AGENT)
    .deadline(timeout) 
    .send() 
    .map_err(|_| <Error<T>>::HttpFetchingError)?; // 在这里检查可能出现的发送时错误

// 到时获取结果
// 返回类型注意下
//   ref: https://substrate.dev/rustdocs/v2.0.0/sp_runtime/offchain/http/struct.PendingRequest.html#method.try_wait
let response = pending
    .try_wait(timeout)
    .map_err(|_| <Error<T>>::HttpFetchingError)?
    .map_err(|_| <Error<T>>::HttpFetchingError)?;

if response.code != 200 {
    debug::error!("Unexpected http request status code: {}", response.code);
    return Err(<Error<T>>::HttpFetchingError);
}

// 存储结果
Ok(response.body().collect::<Vec<u8>>())
```

##### Json转换

###### 引入依赖

这里我们主要是用了serde` and `serde-json 这两个包，但有些功能是标准库的，我们在饮用的时候要注意

```toml
alt_serde = { version = "1", default-features = false, features = ["derive"] }
serde_json = { version = "1", default-features = false, git = "https://github.com/Xanewok/json", branch = "no-std", features = ["alloc"] }
```

###### 反序列化

导入依赖并自定义序列化函数

```rust
// 使用 `alt_serde`, Xanewok修改过的 `serde_json`我们可以同时使用 with serde(features `std`) and alt_serde(features `no_std`).
use alt_serde::{Deserialize, Deserializer};

// 这里将原本默认的String 转成 vector的字节数组 
pub fn de_string_to_bytes<'de, D>(de: D) -> Result<Vec<u8>, D::Error>
where D: Deserializer<'de> {
    let s: &str = Deserialize::deserialize(de)?;
    Ok(s.as_bytes().to_vec())
}
```

定义相应的结构用来反序列化

```rust
//  `alt_serde`依赖指向
// ref: https://serde.rs/container-attrs.html#crate
#[serde(crate = "alt_serde")]
#[derive(Deserialize, Encode, Decode, Default)]
struct GithubInfo {
    // 使用我们上面声明的转换函数
    #[serde(deserialize_with = "de_string_to_bytes")]
    login: Vec<u8>,
    #[serde(deserialize_with = "de_string_to_bytes")]
    blog: Vec<u8>,
    public_repos: u32,
}
```

反序列化

```rust
// 将响应反序列化到结构体中
fn fetch_n_parse() -> Result<GithubInfo, Error<T>> {
    let resp_bytes = Self::fetch_from_remote()
        .map_err(|e| {
            debug::error!("fetch_from_remote error: {:?}", e);
            <Error<T>>::HttpFetchingError
        })?;

    let resp_str = str::from_utf8(&resp_bytes)
        .map_err(|_| <Error<T>>::HttpFetchingError)?;

    // Deserializing JSON to struct, thanks to `serde` and `serde_derive`
    let gh_info: GithubInfo = serde_json::from_str(&resp_str).unwrap();
    Ok(gh_info)
}
```

#### 本地存储

多个链下工作机的存储机制，这个还蛮有意思的

定义用于链下工作机的存储

```rust
fn fetch_github_info() -> Result<(), Error<T>> {
    // 最好是和pallet同名，作为pallet下的链下工作机
    let s_info = StorageValueRef::persistent(b"offchain-demo::gh-info");
    // ...
}
```

提供了`get`, `set`,和`mutate`的API，这里我们通过`mutate`来展示

先检查是否调用过

```rust
fetch_github_info() -> Result<(), Error<T>> {
    // ...
    if let Some(Some(gh_info)) = s_info.get::<GithubInfo>() {
        // gh-info has already been fetched. Return early.
        debug::info!("cached gh-info: {:?}", gh_info);
        return Ok(());
    }
    // ...
}
```

全局🔒

```rust
fn fetch_if_needed() -> Result<(), Error<T>> {
    // 一致性保证
    // ref: https://substrate.dev/rustdocs/v2.0.0-rc3/sp_runtime/offchain/storage_lock/index.html
    //
    // 四种🔒，两个维度，time and block
    //   1) `new` - 锁有默认的过期时间
    //   2) `with_deadline` - 默认区块数，自定义时间的
    //   3) `with_block_deadline` - 默认时间，自定义区块数
    //   4) `with_block_and_time_deadline` - 两者均为自定义
   
    let mut lock = StorageLock::<BlockAndTime<Self>>::with_block_and_time_deadline(
        b"offchain-demo::lock",
        LOCK_BLOCK_EXPIRATION,
        rt_offchain::Duration::from_millis(LOCK_TIMEOUT_EXPIRATION)
    );
		
  	// 尝试获取🔒，失败的话说明有其它的OCW在处理相关的逻辑，这里直接return
    // ref: https://substrate.dev/rustdocs/v2.0.0-rc3/sp_runtime/offchain/storage_lock/struct.StorageLock.html#method.try_lock
    if let Ok(_guard) = lock.try_lock() {
        // fetching logic here ...
    }
    //...
}
```

锁到期后获取

```rust
fn fetch_if_needed() -> Result<(), Error<T>> {
    // _guard 离开作用域，释放🔒 这里可以说下生命周期在锁机制的好处
    if let Ok(_guard) = lock.try_lock() {
        match Self::fetch_n_parse() {
            Ok(gh_info) => { s_info.set(&gh_info); }
            Err(err) => { return Err(err); }
        }
    }

    Ok(())
}
```

### 货币类型

#### 基础货币

```rust
use frame_support::traits::Currency;
```

基础货币中的trait提供了可互换资产（ fungible asset）的功能

```rust
pub trait Trait: system::Trait {
    type Currency: Currency<Self::AccountId>;
}
```

关联类型后会提供一些方法，参考 [`Currency`](https://substrate.dev/rustdocs/v2.0.0/frame_support/traits/trait.Currency.html).如下面返回系统的总发行量

```rust
// 在 decl_module！
T::Currency::total_issuance();
```

在runtime中也可以通过别名来使用

```rust
type BalanceOf<T> = <<T as Trait>::Currency as Currency<<T as system::Trait>::AccountId>>::Balance;
```

#### 可保留货币

这种在比如说开发激励，债券这种场景下比较有用

##### 导包及trait约束

```rust
use frame_support::traits::{Currency, ReservableCurrency};

pub trait Trait: system::Trait {
    type Currency: Currency<Self::AccountId> + ReservableCurrency<Self::AccountId>;
}
```

##### 解锁和释放

关联函数就是 `reserve` 和 `unreserve`， 参考下面两个实现

```rust
pub fn reserve_funds(origin, amount: BalanceOf<T>) -> DispatchResult {
    let locker = ensure_signed(origin)?;

    T::Currency::reserve(&locker, amount)
            .map_err(|_| "locker can't afford to lock the amount requested")?;

    let now = <system::Module<T>>::block_number();

    Self::deposit_event(RawEvent::LockFunds(locker, amount, now));
    Ok(())
}
```

```rust
pub fn unreserve_funds(origin, amount: BalanceOf<T>) -> DispatchResult {
    let unlocker = ensure_signed(origin)?;

    T::Currency::unreserve(&unlocker, amount);

    let now = <system::Module<T>>::block_number();

    Self::deposit_event(RawEvent::UnlockFunds(unlocker, amount, now));
    Ok(())
}
```

#### 可锁定资金

这个使用场景比较多，触发强制执行之类的操作，罚金之类都可以

```rust
use frame_support::traits::{LockIdentifier, LockableCurrency}
```

定义一个标识符

```rust
const EXAMPLE_ID: LockIdentifier = *b"example ";
```

锁定资产

```rust
fn lock_capital(origin, amount: BalanceOf<T>) -> DispatchResult {
    let user = ensure_signed(origin)?;

    T::Currency::set_lock(
        EXAMPLE_ID,
        &user,
        amount,
        WithdrawReasons::except(WithdrawReason::TransactionPayment),
    );

    Self::deposit_event(RawEvent::Locked(user, amount));
    Ok(())
}
```

#### 货币供给问题

发生铸币等情况时，对系统所造成的影响要注意。 这块的逻辑多思考下，感觉不太合理

奖励资金

```rust
pub fn reward_funds(origin, to_reward: T::AccountId, reward: BalanceOf<T>) {
    let _ = ensure_signed(origin)?;

    let mut total_imbalance = <PositiveImbalanceOf<T>>::zero();

    let r = T::Currency::deposit_into_existing(&to_reward, reward).ok();
    total_imbalance.maybe_subsume(r);
    T::Reward::on_unbalanced(total_imbalance);

    let now = <system::Module<T>>::block_number();
    Self::deposit_event(RawEvent::RewardFunds(to_reward, reward, now));
}
```

削减资金

```rust
pub fn slash_funds(origin, to_punish: T::AccountId, collateral: BalanceOf<T>) {
    let _ = ensure_signed(origin)?;

    let imbalance = T::Currency::slash_reserved(&to_punish, collateral).0;
    T::Slash::on_unbalanced(imbalance);

    let now = <system::Module<T>>::block_number();
    Self::deposit_event(RawEvent::SlashFunds(to_punish, collateral, now));
}
```

### 随机性

这块话题还不小，这里的话我们通过内置的一些包来简单的使用下，后面再来拓展展开区块链系统中的随机性问题

trait 约束

```rust
pub trait Trait: system::Trait {
    type Event: From<Event> + Into<<Self as system::Trait>::Event>;

    type RandomnessSource: Randomness<H256>;
}
```

简单实用

```rust
fn consume_randomness(origin) -> DispatchResult {
    let _ = ensure_signed(origin)?;
  
    let subject = Self::encode_and_update_nonce();

    let random_seed = T::RandomnessSource::random_seed();
    let random_result = T::RandomnessSource::random(&subject);

    Self::deposit_event(Event::RandomnessConsumed(random_seed, random_result));
    Ok(())
 }
```

这里可以引申VRF

## Runtime

### APIs

这里有必要展示下架构图，然后再介绍。

每个substrate的Node都包含一个runtime。定义了核心的业务逻辑，交易是否合法以及当交易发生了，状态该如何改变。runtime会编译成WASM方便升级。runtime之外的我们称之为“outer node”，不会被编译成WASM，只会被编译成本地二进制文件。主要负责节点的发现、交易池、块和交易的广播、共识、响应远程调用等。这里介绍一个简单的示例。

我们准备一个求和的功能

```rust
decl_storage! {
    trait Store for Module<T: Trait> as TemplateModule {
        Thing1 get(fn thing1): Option<u32>;
        Thing2 get(fn thing2): Option<u32>;
    }
}
```

实现trait

```rust
impl<T: Trait> Module<T> {
    pub fn get_sum() -> u32 {
        Thing1::get() + Thing2::get()
    }
}
```

通常我们会在对应的目录下构建，如这里的runtimes/api-runtime/src/lib.rs

```rust
sp_api::decl_runtime_apis! {
    pub trait SumStorageApi {
        fn get_sum() -> u32;
    }
}
```

实现API

主要是在[`impl_runtime_apis!`](https://substrate.dev/rustdocs/v2.0.0/sp_api/macro.impl_runtime_apis.html)宏中实现， 包括一些参数的声明

```rust
impl_runtime_apis! {
  // --snip--

  impl sum_storage_rpc_runtime_api::SumStorageApi<Block> for Runtime {
        fn get_sum() -> u32 {
            SumStorage::get_sum()
        }
    }
}

```

#### 调用

```rust
let sum_at_block_fifty = client.runtime_api().get_sum(&50);
```

### 交易费

交易费用怎么样去设定，按字节还是不同函数调用的权重，这里介绍一个简单的示例。按照线性关系去计算

```rust
total_fee = transaction_length * length_fee + weight_to_fee(total_weight)
```

#### 参数设定

这里给个简单的例子，并自己实现计算交易费的方式

```rust
parameter_types! {
    pub const TransactionByteFee: u128 = 1;
}

impl transaction_payment::Trait for Runtime {
    type Currency = balances::Module<Runtime>;
    type OnTransactionPayment = ();
    type TransactionByteFee = TransactionByteFee;
    type WeightToFee = IdentityFee<Balance>;
    type FeeMultiplierUpdate = ();
}
```

抽象结构体

```
pub struct LinearWeightToFee<C>(sp_std::marker::PhantomData<C>);
```

实现WeightToFeePolynomial trait

```rust
impl<C> WeightToFeePolynomial for LinearWeightToFee<C>
where
    C: Get<Balance>,
{
    type Balance = Balance;

    fn polynomial() -> WeightToFeeCoefficients<Self::Balance> {
      	// 这个结构体的构造要注意下
        let coefficient = WeightToFeeCoefficient {
            coeff_integer: C::get(),
            coeff_frac: Perbill::zero(),
            negative: false,
            degree: 1,
        };

        smallvec!(coefficient)
    }
}
```

为runtime实现交易费用的trait

```rust
parameter_types! {
    // 线性计算的系数
    pub const FeeWeightRatio: u128 = 1_000;

    // --snip--
}

impl transaction_payment::Trait for Runtime {
  
    type WeightToFee = LinearWeightToFee<FeeWeightRatio>;

    // --snip--
}
```

一个稍微复杂的实现

```rust
pub struct QuadraticWeightToFee;

impl WeightToFeePolynomial for QuadraticWeightToFee {
    type Balance = Balance;

    fn polynomial() -> WeightToFeeCoefficients<Self::Balance> {
        let linear = WeightToFeeCoefficient {
            coeff_integer: 2,
            coeff_frac: Perbill::from_percent(40),
            negative: true,
            degree: 1,
        };
        let quadratic = WeightToFeeCoefficient {
            coeff_integer: 3,
            coeff_frac: Perbill::zero(),
            negative: false,
            degree: 2,
        };

        smallvec![quadratic, linear]
    }
}
```

为runtime实现交易费用的trait

```rust
impl transaction_payment::Trait for Runtime {

    type Currency = SpendingAssetCurrency<Self>;

    // --snip--
}
```



## TODO

默认实例问题

weight接口

 T::Currency::unreserve（）

Imbalances