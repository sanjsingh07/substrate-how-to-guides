---
sidebar_position: 7
keywords: pallet design, intermediate, runtime
---

# Generating on-chain randomness

_Useful in instances to generate unique values or select things without bias._

## Goal

Implement randomness for a pallet.

## Use cases

- NFT applications
- Casino gaming type applications

## Overview

[Randomness][randomness-kb] is useful in computer programs for everything from gaming applications to selecting block
authors. True randomness is hard to come by in deterministic computers. This is particularly true in the context
of a blockchain when all the nodes in the network must agree on the state of the chain. FRAME provides runtime engineers
with a source of randomness, using the [Randomness trait][randomness-rustdocs].

This guide would step you through making use of FRAME's Randomness trait by using it's `random` method and a nonce as a subject.
For additional entropy to the randomness value, the last step shows how to assign the `RandomnessCollectiveFlip` pallet
to the configuration trait of a pallet exposing some "random" type.

## Steps

### 1. Import `Randomness`

From `frame_support`, import the `Randomness` trait:

```rust
use frame_support::traits::Randomness;
```

Now you'll have to include it in your pallet's configuration trait:

```rust
#[pallet::config]
	pub trait frame_system::Config {
        type MyRandomness: Randomness<H256>;
    }

```

Notice that the `Randomness` trait specifies a generic return of type `Output`. Use [`sp_core::H256`][h256-rustdocs] in your pallet
to satisfy that trait requirement.

:::note A warning on using this trait:
From the [documentation][randomness-rustdocs]: _"This gives you something that approximates [real randomness]. At best, this will be
randomness which was hard to predict a long time ago, but that has become easy to predict recently._"
:::

### 2. Create a nonce

Use a nonce to use as a subject for the `frame_support::traits::Randomness::random(subject: &[u8])` method.

There's two steps to including a nonce in your pallet:

1. **Create a `Nonce` storage item.** This could be of type `u32` or `u64` (no need for it to be larger).
2. **Create a private nonce function.** This will be used to increment the nonce each time it's used.

The `increment_nonce()` private function could be implemented in such a way that it returns the nonce as well as
updates it. Using this approach it would look like this:

```rust
    fn get_and_increment_nonce() -> Vec<u8> {
        let nonce = Nonce::get();
        Nonce::put(nonce.wrapping_add(1));
        nonce.encode()
    }
```

:::note
Learn more about [`wrapping_add`][wrappingadd-rustdocs] and [`encode()`][encode-rustdocs] in the Rust documentation.
:::

### 3. Use Randomness in a dispatchable

Using the nonce, you can call the `random()` method that `Randomness` exposes. The code snippet below is a made up example
that assumes relevant events and storage items have been implemented:

```rust
        #[pallet::weight(100)]
        pub fn create_unique(
			origin: OriginFor<T>)
			-> DispatchResultWithPostInfo {

                // Account calling this dispatchable.
                let sender = ensure_signed(origin)?;

                // Random value.
				let nonce = Self::get_and_increment_nonce();
                let randomValue = T::MyRandomness::random(&nonce);

                // Write the random value to storage.
                <MyStorageItem<T>>::put(randomValue);

                Self::deposit_event(Event::UniqueCreated(randomValue));
            }
```

### 4. Updating your pallet's runtime implementation

Having added a type to your pallet's configuration trait `Config` opens up the opportunity to further enhance the
randomness derived by the `Randomness` trait, by using the [Randomness Collective Flip pallet][rcf-pallet-rustdocs].

Using this pallet alongside the `Randomness` trait will significantly improve the entropy being processed by `random()`.

In `runtime/lib.rs`, assuming `pallet_random_collective_flip` is instantiated at `RandomnessCollectiveFlip`, specify your exposed
type:

```rust
impl my_pallet::Config for Runtime{
    type Event;
    type MyRandomness = RandomnessCollectiveFlip;
}

```

## Examples

- [Randomness used in BABE](https://github.com/paritytech/substrate/blob/master/frame/babe/src/randomness.rs)
- [FRAME's Lottery pallet](https://github.com/paritytech/substrate/blob/master/frame/lottery/src/lib.rs#L471)

## Related material

- [Verifiable Random Functions](https://en.wikipedia.org/wiki/Verifiable_random_function)

[randomness-kb]: https://substrate.dev/docs/en/knowledgebase/runtime/randomness
[randomness-rustdocs]: https://substrate.dev/rustdocs/latest/frame_support/traits/trait.Randomness.html
[h256-rustdocs]: https://substrate.dev/rustdocs/latest/sp_core/struct.H256.html
[encode-rustdocs]: https://substrate.dev/rustdocs/latest/frame_support/dispatch/trait.Encode.html#method.encode
[wrappingadd-rustdocs]: https://substrate.dev/rustdocs/latest/funty/trait.IsInteger.html#tymethod.wrapping_add
[rcf-pallet-rustdocs]: https://substrate.dev/rustdocs/latest/pallet_randomness_collective_flip/index.html
