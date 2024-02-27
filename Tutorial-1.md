## Substrate from scratch, seriously.
#### A series of opinionated tutorials to grasp and learn Substrate.

*Part 1* - Let's get our hands dirty and let's build from scratch a coin-flipper with Substrate.

![Coin Flipping Retro Image](https://cdn-images-1.medium.com/max/800/0*NrLhr3_jviORad-b.jpg)
*The pleasure of flipping a coin, before Substrate.*

My experience with the [Polkadot Blockchain Academy](https://polkadot.network/development/blockchain-academy/) in Hong Kong was terrific (and I *strongly, strongly* encourage everyone [to participate](https://polkadot.network/development/blockchain-academy/])).

There, I had the opportunity to get deep into [the internals of Polkadot and Substrate](https://polkadot-blockchain-academy.github.io/pba-content/hong-kong-2024/index.html) thanks to quite skilled instructors. I found out Polkadot to be an impressive technology that modularises the bloated, monolithic, and static concepts of Ethereum and Bitcoin, making it modular and *evolutionary*. Honestly it makes Ethereum and Bitcoin look like technologies from last century.

On a personal level, I had always been interested in Polkadot, and now I'm even enthusiast about it. Unlike other blockchains, Polkadot's have dynamic and customisable foundations that allows it to quickly adapt to the needs of the future.

The Substrate framework also has a innovative approach: is literally a barebone "scaffolding" blockchain node that is used as a base node framework "ready for hack". It also constains a wide collection of modules called 'pallets' that implement a lot of blockchain concept, like networking and consensus. This approach hides tons of complexity allowing anyone to write its own Layer-1 Blockchain, concentrating just on the runtime logic. **Anyway, it's not exactly all rainbows and unicorns.** Such innovative approach comes to a price: complexity is hidden, therefore you really need to know what you are doing and there's a stepped learning curve, exactly like for Rust.

Furthermore, even if writing a layer 1 blockchain or a dapp runtime logic is a piece of cake with to substrate, the *type system* and all the *pallet configuration* details, in my personal experience, are a bit tricky to understand. Besides it's normal to dig deep into pallets code and documentation, macro here are really some kind of black magic and you can get lost from the amount of *associated types* you need to undestand and deal with. There are some few concept you need to understand VERY well before coding anything.

From my personal experience I learn much faster and deeper making my hands dirty building a project. That's why this tutorial is a biased approach about learning, I'm describing exactly how I learned to build a substrate app. First we grasp the fundamentals with a dead-simple app from zero, and then we build on top of it.

**So, I decided to write the Substrate tutorials I wished to read myself.** I hope my efforts prove useful in understanding this impressive and complex technology.

Let's finally get started. I aim to create a straightforward Substrate coin flipper. **Why a coin flipper?** It's a common [smart contract example](https://github.com/paritytech/ink-playgroung-flipper/blob/main/lib.rs), and my aim is breaking down complex concepts into simpler ones to make it easier to understand how to start to develop a Substrate FRAME pallet. We'll also add some random stuff logic to make it fancier and plenty of tests.

**TLDR**: You can just dive in the final code [here](https://github.com/davassi/substrate-coin-flipper.git).

### Let's start.
1. If you are reading this, I'm taking the assumption you know already how to use git, so just clone somewhere **precisely** this repository:
```bash=
$ git clone https://github.com/substrate-developer-hub/substrate-node-template
```
Here you have to rename the directory from substrate-node-template to something more meaningful, like **substrate-coin-flipper**.

2. I also take the assumption that  you have rust installed, so just type cargo build it and let's run our node to check everything is working correctly:
```bash=
$ cargo build 
$ cargo run
```
The first time it will take literally ages to compile, time for a smoothie. But now let's dive into the meat. Go exactly to TO THIS FILE 
```bash=
$ {YOUR_DIRECTORY_PROJECT}/substrate-coin-flipper/pallets/template/src/lib.rs
```
Here's where all our custom pallets live (we'll refactor the *template* name soon).

3. Here is the place that things get interesting, it's the core of our application where we are going to define all the logic of our application.
Please take some time to study and get familiar with this file: there are 5 Areas defined by macros that are super important to understand. I'm gonna break them down:

```rust=
#[pallet::pallet]
```
* This is the pallet definition. It is literally just a placeholder you must alway specify.

```rust=
#[pallet::config]
```

* This is the configuration pallet, the part we are gonna have a lot of ~~struggle~~ fun with.

```rust=
#[pallet::storage]
```
* In my personal experience this is probably one of most pleasant part to develop. It's the storage model where you record your structured data model. Storage is pretty interesting to play with and call logic is very straightforward.

```rust=
#[pallet::error]
```
* This is simple a bill of error that your application logic can rise under certain conditions.
```rust=
#[pallet::call]
```
* And here is where all the logic of the app is defined. Also this part is very pleasant to write.

This is the bare minimum knowledge you need to have to build a substrate pallet. My personal experience is that everything was pretty straightforward except for the type system and the config pallet that gave me some hard time.

The business logic is crucial, as here it is where the macro magic happens to enable anyone to interact with our pallet from the external world. Extrinsics are simply a broader term for transactions. When writing these functions, it's important to remember one key rule: DO NOT PANIC (interpret this in any possible way you can).

There are a lot of other macros `(#[pallet::hooks], #[pallet::genesis_config], #[pallet::genesis_build])` that are not essential for our barebone case and they will be material for the next tutorials.

Ok so far so good.

### Yeah, but now the things start to be counterintuitive.

Logically, if I want to do a coin flipper, obiviously I want to define a **coin** to *flip* (yep, I can define it easily with a struct) but where do I define it? 

We must never forget that we are dealing with a blockchain, therefore we need always to think in terms of data ledger that needs to be stored (using Storage) and transactions that change the state of this data (the Extrinsics).

Therefore we need to create a Coin entity and associate it with my account in the storage.

For me the first difficulty to overcome was to understand was **the type system** and the **Configuration**. For instance now how do I get my account? I need to identiy my account ID but first I need to find out where is defined the Account type!

**In any FRAME pallet the Config trait is always generic over T** and that makes our pallet logic completely agnostic to any specific implementation details, that will be defined and made concrete eventually at a later point, in the Runtime or in the mocks for the tests.

Here is where things get extremely interesting and, in my opinion, in the official documentation this is not stressed enough, this is where the "magic" starts. The Config of our pallets contains all specifications and informations that are referenced from different Pallets. In this place we get ALL the types and constants go in here. If the pallet is dependent on specific other pallets, then their configuration traits must be added HERE to our implied traits list.

And how we will see, most of the most common associated generic types definitions are "centralised" in a common pallet called the frame_system pallet.

Ok, now it seems even more complicated that it is.

I try to explain again:
1. All pallets have a configuration made of definition generic types. 
2. Our pallets are declaring a new configuration that can extends types of the dependency pallets. 
3. The runtime is the right place where to finally define the configuration.

The Config trait of our pallet is where you define how the runtime or other pallets can provide concrete types or implementations for the abstract concepts defined in your traits.

From official documentation:

`"The account properties—such as the AccountId—can be defined generically in the frame_system module. The generic type is then resolved as a specific type in the runtime implementation, and eventually assigned a specific value. For example, the Account type in FRAME relies on an associated AccountId type. The AccountId type remains a generic type until it is assigned a type in the runtime implementation for a pallet that needs this information."`

What does it mean? It means that the pallet called **frame_system** has already inside all the associated generic type declaration (not definition!) that we need, and we just need to recall them inside our pallet.

So if I check the file frame/system/src/libs.rs I find out that is already defined an associated type called `AccountId` that extends plenty of traits. 

So inside my Config I will need to use over a generic T that extends the frame_system the account id in the form 

```rust=16
<T as frame_system::Config>::AccountId  
```

that is even more handy to redefine as 
```rust=16
type AccountIdOf<T> = <T as frame_system::Config>::AccountId;
```

```rust=
#[pallet::config]
pub trait Config: frame_system::Config {
    // Here we specify they type of a very useful u32 constant...
    type VeryUsefulConstant: Get<u32>;
}
```
and now it's time to jump in the runtime file to specify the concrete implementation of this associated type.

```rust=
/// Configure the pallet-flipcoin
impl pallet_dex::Config for Runtime {
    // 
    type VeryUsefulConstant = ConstU32<42>;
}
```
Noteworthy is that this comes very handy in the moment we need to execute some tests, therefore we just configure a mock runtime to test the pallet.
So far so good, but what do we need to specify for our flipcoin pallet?
Generally you'll want to split these "calls" into two groups: 
A. Public calls that are called and signed by an external account. 
B. Root calls that are allowed to be made only by the governance system.
C. There's also another type, the unsigned transactions, that we won't cover here now.
Your final runtime is composed of Pallets, which are brought together with the construct_runtime! macro.