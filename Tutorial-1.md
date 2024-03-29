## Substrate from scratch, seriously
#### A series of tutorials to grasp and learn Substrate. Part 1

Let's get our hands dirty and build a coin-flipper project from scratch using the Substrate framework.

![Coin Flipping Retro Image](https://cdn-images-1.medium.com/max/800/0*NrLhr3_jviORad-b.jpg)
*The pleasure of flipping a coin, before Substrate.*

My experience with the [Polkadot Blockchain Academy](https://polkadot.network/development/blockchain-academy/) in Hong Kong was terrific (and I *strongly* recommend everyone interested [to apply and participate](https://polkadot.network/development/blockchain-academy/])).

Following the lectures of skilled instructors, I had the opportunity to delve deep into [the internals of Polkadot and Substrate](https://polkadot-blockchain-academy.github.io/pba-content/hong-kong-2024/index.html). Polkadot is built on a modular and customisable foundation, resulting in a more adaptable and *evolutionary* technology. It makes Ethereum and Bitcoin, with their monolithic and static concepts, look like prior-generation technologies.

Substrate, the [SDK framework](https://github.com/paritytech/polkadot-sdk) we will use, serves as a pre-built, customisable 'scaffolding' blockchain node. It contains an extensive collection of modules, called "pallets", that encapsulate essential blockchain functions like networking, token management, consensus, and others.

Substrate simplifies the process, *hides a lot of complexity* and allows developers to create their own layer-1 blockchain, focusing solely on the runtime logic. However, **such an innovative approach comes at a cost**. While much complexity is concealed, developers still need to know precisely what they are doing. This leads to a steep learning curve... similar to Rust.

On the one hand, Substrate has plenty of built-in engines, which makes it a breeze to write layer-1 chain runtime logic. On the other hand, the *type system* and all the *pallet configuration* details may be a bit tricky to understand at first. Delving deep into pallet code and documentation, you can easily **get lost** due to the numerous associated types you have to deal with.

**So, I decided to create Substrate tutorials I wish I had before building my first Substrate app.** This tutorial might be biased because it describes my personal experience, but I hope my efforts prove helpful in understanding this remarkable technology.

Let's create a Substrate coin flipper step by step. **Why a coin flipper?** It's a typical [smart contract example](https://github.com/paritytech/ink-playgroung-flipper/blob/main/lib.rs), and I aim to break down complex concepts into simpler ones to make it easier to understand how to start to develop a Substrate pallet. We'll also add some random logic to make it fancier and numerous tests.

**TLDR**: You can dive into the final code [here](https://github.com/davassi/substrate-coin-flipper.git). Please feel free to leave comments!

### Let's start setting things up.

1. If you are reading this, I'm taking the assumption you already know how to use git, so clone somewhere **precisely** this repository:

```bash=
$ git clone https://github.com/substrate-developer-hub/substrate-node-template
```

You must rename the directory from substrate-node-template to something more meaningful, like **substrate-coin-flipper**.

2. I take the assumption that you have Rust installed, so type Cargo build it and let's run our node to check everything is working correctly:

```bash=
$ cargo build 
$ cargo run
```

The first time, it will take a while to compile. 
Now, go to precisely this file:

```bash=
$ {YOUR_DIRECTORY_PROJECT}/substrate-coin-flipper/pallets/template/src/lib.rs
```

Here's where all our custom pallets live.

3. This is where things get interesting; it's the core of our application, where we define its logic.
Please take some time to study and get familiar with this file: there are 5 Areas defined by macros that are important to understand. I'm gonna break them down:

```rust=29
#[pallet::pallet]
```
* This is the pallet definition. It is literally a placeholder you must always specify.

```rust=32
#[pallet::config]
```

* This is the configuration pallet, the part we'll have a lot of ~~struggle~~ fun with. We'll have our section on it.

```rust=59
#[pallet::storage]
```
* It's the storage model where you record your structured data model. Storage is fun to play with, and call logic is very straightforward. 

```rust=75
#[pallet::error]
```
* This is a simple list of errors that logic can raise under certain conditions.

```rust=83
#[pallet::call]
```
* This is where all the logic of the app is defined. It is also enjoyable to write.

This is the bare minimum of knowledge required to build a substrate pallet. All these areas are straightforward, apart from carefully considering the type system and configuration pallet, as mentioned above.

There are many other macros (#[pallet::hooks], #[pallet::genesis_config], #[pallet::genesis_build]) that are not essential for our case, but they will be material for the following tutorials.

Ok so far, so good.

### The Config: where things are getting a little "unusual".

Logically, if I want to create a coin flipper, I need to define a **coin** to *flip* (yep, I can define it easily with a struct), but where do I define it? 

We must always remember that we are dealing with a blockchain. Therefore, we need to always think about the data ledger that needs to be stored (using Storage) and transactions that change the state of this data (the Extrinsic).

Therefore, we need to create a `Coin` entity and associate it with my account in the Storage.

Now it's time to dig into **the type system** and the **Configuration**. How do I get my account? I need to identify my account ID, but first, I need to find out where the Account type is defined!

**In any FRAME pallet the Config trait is always generic over T** and this allows our pallet logic to be completely agnostic to any specific implementation details. The configuration of our Pallet will eventually be defined and made concrete later, in the runtimes or in the mocks for the tests, a pattern that resembles *dependency injection*.

```plantuml
@startuml
!theme cerulean-outline

package "Pallets" {
[frame_fungibles] as FF
[frame_accounts] as FP
[frame_config] as FC
[substrate-coin-flipper] as SCF
}

package "Runtimes" {
frame polkadot_runtime {
[Polkadot STF] as PSCF
}
frame kusama_runtime {
[Kusama STF] as KSCF
}
frame test {
[Test Enviroment] as TSCF
}
}

FC --> SCF : "derived by"
FF --> SCF : "derived by"
FP --> SCF : "derived by"

SCF ..> PSCF : "defined in"
SCF ..> KSCF : "defined in"
SCF ..> TSCF : "mocked in"

@enduml
```

This is where things get interesting, and IMHO, the official documentation needs to emphasise this part more. The configuration of our Pallet contains all the specifications and information referenced and derived from different Paletts. All types and constants that go in here are generic. If the Pallet depends on specific other pallets, their configuration traits must be added to our implied trait list.

As we will see, most of the most common associated generic types definitions are centralised in a pallet called the **frame_system** Pallet.

Ok, now it seems even more complicated than it is.

Let's recap:

1. Any pallet has a configuration **composed of associated types.**
2. Our Pallet declares a new configuration that **can derive the associated types** of the dependency pallets. 
3. The Runtime is where **we give a specific definition (trait implementation or value) to all the associated types** of the configuration of our Pallet.

The Config trait of our Pallet defines how the Runtime or other pallets can provide concrete types or implementations for the abstract concepts defined in our traits.

From the [official documentation](https://docs.substrate.io/learn/accounts-addresses-keys/):

> `"The properties [...] can be defined generically in the frame_system module. The generic type is then resolved as a specific type in the runtime implementation and eventually assigned a specific value. [...] The AccountId type remains a generic type until it is assigned a type in the runtime implementation for a pallet that needs this information."`

What does it mean? It means that the Pallet called **frame_system** contains all the associated generic type declarations that we need. We just need to use them inside our Pallet and associate them with a concrete type in our Runtimes.

So, if I check the file frame/system/src/libs.rs, I find out that it is already defined by an associated type called `AccountId`, which is also extended by plenty of other traits. 

So inside my config, I will need to use `AccountId` over **a generic T** that extends the frame_system in the form:

```rust=16
T::AccountId  
```

And because **T belongs to the frame_system** Pallet, therefore we can write it as:
```rust=16
<T as frame_system::Config>::AccountId  
```

that is even more handy to redefine as 
```rust=16
type AccountIdOf<T> = <T as frame_system::Config>::AccountId;
```

**`<Type as Trait>::item`** means accessing the **item** on **Type**, disambiguating the fact that it should come from an implementation **Trait** for **Type**.
    
That's it. Once you have understood this concept, the configuration becomes pretty straightforward, 
as in the extrinsic logic, we can use `AccountIdOf<T>` anytime we need to refer to accounts.

Only at Runtime will **AccountId be associated with a concrete trait implementation.**

### The Config: a little deeper into the nested generic associated types

At this point, we need to briefly discuss more complex generic associated types, which will be helpful for future substrate development topics.

Let's consider **BalanceOf<T>** defined as:

```rust=
type BalanceOf<T> =
    <<T as Config>::Currency as Currency<<T as frame_system::Config>::AccountId>>::Balance;
```

At first glance, it may seem intimidating, but it is the same as **`<Type as Trait>::item`**, repeated three times. We can express the type in this manner:

```rust=
type ConfigCurrency<T> = <T as Config>::Currency;
type ConfigAccountId<T> = <T as frame_system::Config>::AccountId>;
type BalanceOf<T> = <ConfigCurrency<T> as Currency<ConfigAccountId<T>>::Balance;
```
    
Let's recap and visualise it in a diagram:
* **type BalanceOf<T**>: This defines a type alias named `BalanceOf` that is generic over `T`.

* **<T as Config>::Currency**: This part specifies that we refer to the `Currency` associated type from the `Config` trait as implemented by `T`.
* **as Currency<<T as frame_system::Config>::AccountId>>**: This further specifies that the `Currency` type we just referred to should itself implement the `Currency` trait. It is generic over the account identifier type provided by `AccountId`.
* **::Balance**: Finally, this part accesses the `Balance` associated type of the `Currency` trait. This `Balance` type represents the amount of currency (e.g., tokens or coins) an account has.

Let's recap it visually:

![image](https://hackmd.io/_uploads/SkTjHDU6p.png)

Putting it all together, `BalanceOf<T>` is a type alias for the balance type used by a pallet's currency system, where the pallet configuration is provided by T. This allows the type to work at **Runtime** with different currency representations. 
    
This is a typical pattern for type definition in substrate development.
    
### The Config: The only constant in life is change
**
A typical use case (that we won't use in the coin-flipper) is the definition of a constant supported by the trait `Get`:

```rust=
#[pallet::config]
pub trait Config: frame_system::Config {
    // Here, we specify the very useful u32 constant type...
    type VeryUsefulConstant: Get<u32>;
}
```
Once defined in our `Config` pallet, we can jump into the runtime file to specify the concrete implementation of this associated type with a value, for instance `42`:

```rust=
/// Configure the pallet-flipcoin
impl pallet_dex::Config for Runtime {
    // 
    type VeryUsefulConstant = ConstU32<42>;
}
```
This comes in handy when we need to execute some tests; therefore, we just configure a Mock runtime to test the Pallet.

### The Storage: a handy place to write down in our blockchain.

Now, it's time to define our data model: a `CoinSide` enum that defines `Head` and `Tail` and a `Coin` struct that holds one of these 2 values. Both types extend the `Default` traits, so I expect a default implementation when I build an object from them. In this case, we have created a `Coin` that always starts with `Head`.

```rust=45
#[derive(Clone, Encode, Decode, Eq, PartialEq, RuntimeDebug, MaxEncodedLen, TypeInfo, PartialOrd, Default)]
enum CoinSide {
    #[default]
    Head,
    Tail,
}
#[derive(Clone, Encode, Decode, Eq, PartialEq, RuntimeDebug, MaxEncodedLen, TypeInfo, PartialOrd, Default)]
pub struct Coin {
    side: CoinSide,
}

```

It is generally a good practice to define your data model in terms of meaningful `structs` and `enums` and not use Storage to store scattered `unsigned ints` or `floats` values.

Now I can define a mapping between an `Account` and a `Coin` through the definition of a `StorageMap`: 

```rust=45
#[pallet::storage]
pub type CoinStorage<T> = StorageMap<_, Blake2_128Concat, AccountIdOf<T>, Coin, OptionQuery>;
```

This is literally as simple as using any `HashMap,` but we are effectively storing data in the ledger of our blockchain app. 
    
The `StorageMap` is not the only type of data structure available, and it offers a lot of flexibility (`ValueQuery`, `OptionQuery`). Part 2 of this tutorial will be dedicated extensively to the `Storage`, but I want to clarify a couple of points:

1. As anyone can notice immediately, the enum `CoinSide` and the struct `Coin` are filled with derived macros: `Clone, Encode, Decode`, etc... all these macros are necessary to allow our types to be used inside a `Storage`.
2. The Storage uses `Blake2_128Concat`, a hashing algorithm we are using for `StorageMap`. There are several hashing functions with different properties. This is all the material for the Part 2 tutorial.

### The Extrinsic: changing state of our blockchain from the external world.

So far, it's been good, but how do we specify the logic that implements our Flipcoin pallet? We need to implement a special kind of transaction called "Extrinsic", which are essentially state transition functions callable externally from our blockchain.

Generally, we can split these "calls" into 3 groups: 

A. Public calls that are called and signed by an external account. 

B. Root calls that are allowed to be made only by the governance system.

C. There's also another type, the unsigned transactions, that we won't cover here now.

We are going to write only extrinsic of type A. The business logic defines the behaviours of our Pallet, as here is where the "macro magic" enables agents from the external world to interact with our blockchain. Extrinsics are simply a broader term for transactions. 

We will define the following calls: 

* **`create_coind`** to create a coin for the sender's account and save it in the StorageMap
* **`flip_coin`** to flip the coin (head to tail or tail to head) and update the coin with the new value.
* **`toss_coin`** to toss the coin (random head or tail) and update the coin with the new value.

When writing these functions, we can follow a structure. Before starting, it's important to remember one essential rule: **"Do not panic!"** (interpret this in any possible way you can).


```rust=
#[pallet::call_index(0)]
#[pallet::weight(T::WeightInfo::do_something())]
pub fn create_coin(origin: OriginFor<T>) -> DispatchResult {
    let who = ensure_signed(origin)?;
    Self::do_create_coin(&who)?;
    Self::deposit_event(Event::CoinCreated { who });
    Ok(())
}
```

As you can see, every single Extrinsic has a defined flow structure. 
The business logic of the extrinsic is wrapped around some functions. 
In particular, we have:

```rust=1
#[pallet::call_index(0)]
```
 
* Every extrinsic starts with a derive macros called `call_index(counter)`, where counter is an incremental number. This annotation specifies the call's index within the Pallet. The index is used to uniquely identify the call when it's invoked. 

```rust=2
#[pallet::weight(T::WeightInfo::do_something())]
```

* We use a macro that specifies the weight of the call, which is a measure of the computational and Storage resources required to execute the call (It will be covered in the part 2 tutorial)

```Rust =3
pub fn create_coin(origin: OriginFor<T>) -> DispatchResult {
    let who = ensure_signed(origin)?;
```

* In the function, we define the `create_coin` function and ensure the call is from a signed origin. It takes an origin parameter of type `OriginFor<T>` and returns a `DispatchResult`
    
```rust=5
Self::do_create_coin(&who)?;
```

* We invoke the business logic defined within `impl<T: Config> Pallet<T> {}`

```rust=6
Self::deposit_event(Event::CoinCreated { who });
```
    
* Assuming the coin creation is successful, this line emits an event to signal that a coin has been created. Emitting an Event is the only way to notify external observers of the execution of the extrinsic.

```rust=7
Ok(())
```
    
* Eventually, we return a `DispatchResult`, hopefully, an `Ok(())`. `Ok()` indicates that the function has been completed successfully and the transaction fees, if any, have been paid (no refund!).

After this, we can implement the logic of the `do_create_coin`

```rust=
// This method creates a new coin for the given account
pub fn do_create_coin(account_id: &T::AccountId) -> DispatchResult {

    if CoinStorage::<T>::contains_key(account_id) {
        // If a coin already exists, return an error
        return Err(Error::<T>::CoinAlreadyExists.into());
    } 

    // Create a new coin
    CoinStorage::<T>::insert(account_id, Coin::default());
    Ok(())
}
```
Note: `Coin::default()`is exactly as writing `Coin { side: CoinSide::Head }`

It is strongly recommended that we follow this pattern: before executing any logic, we check for possible errors that might invalidate our logic. So, we check if the coin already exists in our store, and if that's the case, we send an error. 

Otherwise, we can continue our logic and store a new default coin in StorageMap.

Now we can continue to define `do_flip` and `do_toss`:

```rust=
#[pallet::call_index(1)]
#[pallet::weight(T::WeightInfo::do_something())]
pub fn do_flip(origin: OriginFor<T>) -> DispatchResult {
    let who = ensure_signed(origin)?;
    Self::do_flip_coin(&who)?;
    Self::deposit_event(Event::CoinFlipped { who });
    Ok(())
}

#[pallet::call_index(2)]
#[pallet::weight(T::WeightInfo::cause_error())]
pub fn do_toss(origin: OriginFor<T>) -> DispatchResult {
    let who : AccountIdOf<T> = ensure_signed(origin)?;
    Self::do_toss_coin(&who)?;
    Self::deposit_event(Event::CoinTossed { who });
    Ok(())
}
```

And also, implement the business logic in `do_flip_coin`:

```rust=
// This method flips the coin for the given account
pub fn do_flip_coin(account_id: &T::AccountId) -> DispatchResult {

    // If a coin does not exist, return an error
    let mut coin = CoinStorage::<T>::get(account_id)
        .ok_or(Error::<T>::CoinDoesNotExist)?;

    // Flip the coin
    coin.side = match coin.side {
        CoinSide::Head => CoinSide::Tail,
        CoinSide::Tail => CoinSide::Head,
    };

    // Update the coin
    CoinStorage::<T>::insert(account_id, coin);

    Ok(())
}
```
and `do_toss_coin`:
```rust=
// This method tosses the coin for the given account
pub fn do_toss_coin(account_id: &T::AccountId) -> DispatchResult {
    let mut coin = CoinStorage::<T>::get(account_id)
        .ok_or(Error::<T>::CoinDoesNotExist)?;

    let block_number = <frame_system::Pallet<T>>::block_number();
    let seed = block_number.try_into().unwrap_or_else(|_| 0u32);

    // This approach uses blocknumber as a seed source. 
    // Never use it in production. 
    let new_side = if Self::generate_insecure_random_boolean(seed) == true {
        CoinSide::Head
    } else {
        CoinSide::Tail
    };

    // Update the coin's side
    coin.side = new_side;
    CoinStorage::<T>::insert(account_id, coin);

    Ok(())
}
```

Because we are working with *generic traits*, we don't know if the conversion at Runtime will work for '`T::BlockNumber`', as this trait may not be directly a `u32`, depending on how we define block numbers in our `Runtime` configuration.

### Adding a pallet dependency and deriving the config

To toss the coin in an almost-random manner, we need to enhance the Pallet's configuration trait `Config` by using [the Insecure Randomness Collective Flip pallet](https://substrate-developer-hub.github.io/substrate-how-to-guides/docs/pallet-design/randomness). The terms "insecure" stands for the fact that randomness generated is not cryptographically secure, as it can be influenced in various ways to gain an advantage. Having some oracle is the only safe way of doing it, but this Pallet is more than enough for our coin example.

We will declare a generic trait implementation of `MyRandomness` in the Config of our Pallet, and to execute the logic, we also need to define it in our Runtime.

1. First, we must declare our pallet dependency inside the Cargo.toml file

```toml=
# [dependencies]
pallet-timestamp = { version = "4.0.0-dev", default-features = false, git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v1.0.0" }
+pallet-insecure-randomness-collective-flip = { git = "https://github.com/paritytech/substrate", package = "pallet-insecure-randomness-collective-flip", default-features = false, branch = "polkadot-v1.0.0" }
```

2. Second, we derive the `Config` trait implementation of the `Insecure Randomness Collective Flip` pallet in our `Runtime`:

```Rust
impl pallet_insecure_randomness_collective_flip::Config for Runtime {}
```

3. Then we add a reference of the Pallet the `Runtime` in the `construct_runtime!` macro

```Rust
RandomnessCollectiveFlip: pallet_insecure_randomness_collective_flip,
```

4. and finally, we bind our derived type MyRandomness with the concrete implementation: 

```Rust
type MyRandomness = RandomnessCollectiveFlip;
```

It is worth mentioning that this process is precisely how we have described the process of deriving a pallet in our configuration:

```plantuml
@startuml
!theme cerulean-outline

package "Pallets" {
[pallet_insecure_randomness_collective_flip] as FF
[substrate-coin-flipper] as SCF
}

package "Runtimes" {
frame polkadot_runtime {
[Polkadot STF] as PSCF
}
}

FF --> SCF : "derived by"
SCF ..> PSCF : "defined in"

@enduml
```

### Invoking the Extrinsic externally

To invoke the extrinsic we have just created, we need first to start the node with

```bash=
$ cargo run -- --dev
```

After starting the node template locally, we can interact with it using the hosted version of the [Polkadot/Substrate Portal](https://polkadot.js.org/apps/#/explorer?rpc=ws://localhost:9944) front-end by connecting to the local node endpoint.

```rust
https://polkadot.js.org/apps/#/explorer?rpc=ws://localhost:9944
```
1. First we Select from the menu Extrinsic, selecting one for the existing test accounts (`ALICE` in this case)
![Developer-Extrinsics](https://github.com/davassi/substrate-tutorials/assets/1568018/b6838faf-0d83-40f0-b6b4-cbed087c0f07)
    
2. Second, we select our Pallet and the extrinsic we want to invoke
    
    ![image](https://hackmd.io/_uploads/SyaBowLap.png)

3. Then we sign the transaction
    
    ![image](https://hackmd.io/_uploads/S1fyovL6p.png)
    
4. We check the result. It will appear in the form of one of these notifications:

    ![image](https://hackmd.io/_uploads/H13siDLTT.png)

### Tests, tests, plenty of tests.

As a general rule, we don't want to release anything in production that does not have a good percentage of tests, for multiple reasons.

* First, test code needs to always be considered first-class code. Testing is a design activity for our traits.
* Second, we can easily catch regressions when our codebase grows and we introduce fixes, patches, new functionalities, or refactorings.
* Third, with blockchain, we often deal with monetary resources such as native tokens, fungible assets, and NFTs. We really want to minimise the risk of losing liquidity and reputation due to bugs.

We will concentrate our efforts on testing the extrinsic, located in the test.rs file.

```rust=6

const ALICE: SignedOrigin = 1u64;

#[test]
fn create_coin_test() {
    new_test_ext().execute_with(|| {

        System::set_block_number(1);

        let origin = RuntimeOrigin::signed(ALICE);

        let result = TemplateModule::create_coin(origin);
        assert_ok!(result);

        System::assert_has_event(Event::CoinCreated { who: ALICE }.into());
    });
}
```

Let's break down the test:

* With `new_test_ext()`, we create a local test environment including new Storage according to the definition present in the mock Runtime.
* At line 14, we set up block number 1 to go past Genesis block (number 0) so events can be deposited. 
* We simulate a signed origin with the mocked account of `ALICE` (defined as an `u64` type).
* At line 17, we finally call the `create_coin` extrinsic
* And, we check that the `DispatchedResult` is actually an `Ok(())`
* As final operations, we check that the event `Event::CoinCreated` is created and deposited by the Extrinsic.

Following precisely the same structure, we can write down the tests for the other 2 Extrinsics:

```rust=
#[test]
fn flip_coin_test() {
    new_test_ext().execute_with(|| {
		
        System::set_block_number(1);

        let origin = RuntimeOrigin::signed(ALICE);

        assert_ok!(TemplateModule::create_coin(origin));
        assert_ok!(TemplateModule::do_flip(origin));

        System::assert_has_event(Event::CoinFlipped { who: ALICE }.into());
    });
}

#[test]
fn toss_coin_test() {
    new_test_ext().execute_with(|| {

        System::set_block_number(1);

        let origin = RuntimeOrigin::signed(ALICE);

        assert_ok!(TemplateModule::create_coin(origin));
        assert_ok!(TemplateModule::do_toss(origin));

        System::assert_has_event(Event::CoinTossed { who: ALICE }.into());
    });
}
```
### Final words and further resources

In this tutorial, we have just scratched the substrate development and testing surface. Writing a proper pallet is more complex and involves aspects of performance and security to consider. 

The following tutorial will cover Storage and Weight management. Meanwhile, here's a list of some references that were very helpful in understanding Substrate pallet development:

[Polkadot Deep Dives - Youtube](https://www.youtube.com/playlist?list=PLOyWqupZ-WGsfnlpkk0KWX3uS4yg6ZztG)
    
[Substrate Randomness](https://docs.substrate.io/build/randomness/)

[Substrate Storage Structures](https://docs.substrate.io/build/runtime-storage/)

[Substrate Accounts](https://docs.substrate.io/learn/accounts-addresses-keys/)

That's it for the moment. If you have suggestions or improvements, or if you find any issues in the code, typos, errors, etc, please feel free to share them on [GitHub](https://github.com/davassi/substrate-tutorials/issues).

I appreciate very much your feedback!
    
    

---

Feel free to reach me on:
    
- Discord: exquisitus9414
- Telegram: @Exquisitus9414  
- [Linkedin](https://www.linkedin.com/in/gianluigidavassi/)
- [GitHub](https://github.com/davassi/)
    