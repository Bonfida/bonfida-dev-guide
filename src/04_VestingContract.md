# VestingContract

The core data structure on which we are building our program will be called `VestingContract`. 
Since we will be using type-casting through the `bytemuck` library, we need to iterate and think about the various types of constraints our definition will need to obey.

## First iteration : what data do we need?

Our program's logic will need something equivalent to the following data structure.

```rust
pub struct VestingContract {
    /// The eventual token receiver
    pub owner: Pubkey,
    /// The contract escrow vault
    pub vault: Pubkey,
    /// Index in the current schedule vector of the last completed schedule
    pub current_schedule_index: u64,
    /// Used to generate the signing PDA which owns the vault
    pub signer_nonce: u8,
    /// Describes the token release schedule
    pub schedule: Vec<VestingSchedule>
}

pub struct VestingSchedule {
    pub unlock_timestamp: u64,
    pub quantity: u64
}
```

In our freshly created project, let's start by renaming the `src/state/example_state_cast.rs` to `src/state/vesting_contract.rs`.
We'll also delete the `src/state/example_state_borsh.rs` file.
Hopefully our IDE will take care of the refactor.
Let's then refactor the `ExampleStateCast` struct to `VestingContract` and paste in the above definitions.

We should be left with something like this :

```rust,noplayground
#[derive(Clone, Copy, Zeroable, Pod)]
#[allow(missing_docs)]
#[repr(C)]
pub struct VestingContract {
    /// The eventual token receiver
    pub owner: Pubkey,
    /// The contract escrow vault
    pub vault: Pubkey,
    /// Index in the current schedule vector of the last completed schedule
    pub current_schedule_index: u64,
    /// Used to generate the signing PDA which owns the vault
    pub signer_nonce: u8,
    /// Describes the token release schedule
    pub schedule: Vec<VestingSchedule>,
}

pub struct VestingSchedule {
    pub unlock_timestamp: u64,
    pub quantity: u64,
}
```

Since `VestingContract` implements the `Clone`, `Copy`, `Zeroable` and `Pod` traits we need to derive these traits for `VestingSchedule` as well.
The last two traits are related to `bytemuck` and enable type casting.
Same goes for the `repr(C)` atrribute which is essential for type casting.


```rust,compile_fail
#[derive(Clone, Copy, Zeroable, Pod)]
#[allow(missing_docs)]
#[repr(C)]
pub struct VestingContract {
    /// The eventual token receiver
    pub owner: Pubkey,
    /// The contract escrow vault
    pub vault: Pubkey,
    /// Index in the current schedule vector of the last completed schedule
    pub current_schedule_index: u64,
    /// Used to generate the signing PDA which owns the vault
    pub signer_nonce: u8,
    /// Describes the token release schedule
    pub schedule: Vec<VestingSchedule>,
}

#[derive(Clone, Copy, Zeroable, Pod)]
#[repr(C)]
pub struct VestingSchedule {
    pub unlock_timestamp: u64,
    pub quantity: u64,
}
```

However, the above still does not compile. To sum up, the core bytemuck trait we are interested in is the `Pod` trait. This trait requires the `Zeroable` and `Copy` traits. However, a `Vec` does not implement the `Copy` trait. In general, this is due to the fact that `Vec` is a pointer which _owns_ a section of heap memory. Copying it would mean allocating a new section of heap memory, whereas the `Copy` trait is reserved for variables which exist on the stack.

In fact it is impossible to directly cast `Vec` objects. 
To work around this limitation, we split our object into a reference to a header and a reference to a slice of `VestingSchedule` objects.
This means that the `VestingContract` object will hold cast references to the underlying objects, instead of being a direct cast by itself.
This layer of indirection is of no consequence in terms of performance, and allows us a lot more flexibility in terms of the kind of data structures we can type cast.

## Second Iteration: fixing our definitions

We begin by renaming the `VestingContract` object into `VestingContractHeader`, removing the schedules from it :

```rust
#[derive(Clone, Copy, Zeroable, Pod)]
#[allow(missing_docs)]
#[repr(C)]
pub struct VestingContractHeader {
    /// The eventual token receiver
    pub owner: Pubkey,
    /// The contract escrow vault
    pub vault: Pubkey,
    /// Index in the current schedule vector of the last completed schedule
    pub current_schedule_index: u64,
    /// Used to generate the signing PDA which owns the vault
    pub signer_nonce: u8,
    pub _padding: [u8; 7],
}

#[derive(Clone, Copy, Zeroable, Pod)]
#[repr(C)]
pub struct VestingSchedule {
    pub unlock_timestamp: u64,
    pub quantity: u64,
}
```

Ah, and we also added an extra field called `_padding`. 
This is where using directly type-cast data structures can seem daunting at first.
Let's take some time to talk about memory alignment.

### An aside on padding and memory alignment

Modern CPUs are wonderful things which are able to manipulate all kinds of objects.
In practice, every operation which a CPU can execute (also called an assembly instruction), _is_ actual physical wiring on the chip.
Depending on the chip, designers can cut down on complexity and conversely increase performance by requiring data to be aligned in a certain way.
As a somewhat appropriate analogy, let's take a list of five 8-letter words :
```
although
boundary
calendar
chemical
diameter
```
Providing a CPU with non-aligned data is akin to presenting you the same list in this way :
```
although
       boundary
   calendar
                chemical
 diameter
```
It's just harder to parse, because we're our ability to parse lists of words is _hard-wired_ to a certain format.
So let's give our poor CPUs a break and look at alignment constraints on the `BPF` architecture.

For any memory address, we say that it is aligned to `n` if its address is a multiple of `n`.
On the Solana BPF (the on-chain program runtime) and `x86_64` architectures, alignment constraints are as follows :
- Primitive types must be aligned to their size, up to a maximum of 8.
- Structs must be aligned to the maximum of their fields' alignment constraints.

For primitive types, this yields the following table :

| Primitive Type | Size (in bytes) | Alignment constraint |
| -------------- | --------------- | -------------------- |
| `u8`, `i8`     | 1               | 1                    |
| `u16`, `i16`   | 2               | 2                    |
| `u32`, `i32`   | 4               | 4                    |
| `u64`, `i64`   | 8               | 8                    |
| `u128`, `i128` | 16              | 8                    |

When the runtime provides you with an account, the address of its first byte can be considered to be 0.
The first 8 bytes are going to be used by the account or instruction tag, which is encoded as a `u64`.
We would be tempted to use a `u8` here, but this would make the rest of the buffer start aligned to 1, which precludes all but the most basic types.

On the `BPF` architecture, being 8-aligned is essentially equivalent to being 0-aligned.
Fortunately, this is also the case on the common `x86_64` architecture.
Unfortunately, the apple ARM architecture (i.e. Apple M1, M2, etc.) can sometimes ask for an alignment of 16.
To get around this, we can use a compilation flag to replace every instance of a type-cast `u128` by a `[u64;2]`.
Even if you don't own an Apple ARM computer, it is important to think about members of the community that do.
We will discuss how to tackle this issue in an annex.

So looking at our `VestingContractHeader` object, we can see that it contains a `u64` field.
This means that its size has to be a multiple of 8, which is why we add 7 bytes of padding.
In practice, these will be implicitly 0 and won't be used by any of our program logic.
As a nice bonus, if you later want to add a new field to the object, you'll be able to do so while maintaing backwards compatibility!

### Final definition

We then define our `VestingContract` object. 
You should notice that since `VestingContract` holds references, it has a generic lifetime argument.
While this can seem daunting, it will not affect its use.
`bonfida-utils` can help us out here by automatically deriving the `WrappedPodMut` trait.
This trait is specifically designed for objects which hold references to `Pod` objects which are contiguous in memory.

```rust,noplayground
#[derive(WrappedPodMut)]
pub struct VestingContract<'a> {
    pub header: &'a mut VestingContractHeader,
    pub schedules: &'a mut [VestingSchedule],
}
```

The `impl` blocks are refactored in the following manner :
```rust
impl VestingContractHeader {
    pub const LEN: usize = std::mem::size_of::<Self>();
}

impl VestingSchedule {
    pub const LEN: usize = std::mem::size_of::<Self>();
}

/// An example PDA state, serialized using Borsh //TODO
#[allow(missing_docs)]
impl<'contract> VestingContract<'contract> {
...
#}
```

What remains is to update `VestingContract`'s `initialize` and `from_buffer` method.
The first step in doing so is to refactor the `super::Tag::ExampleStateCast` object to `super::Tag::VestingContract`.
Doing so finalizes the `initialize` method : its only role is to write the account's tag into the first 8 bytes of the data buffer.
Tags are incredibly important to ensure that an account is being interpreted correctly : we wouldn't want an attacker to substitute one type of account for another.
This is a way of implementing runtime type checks on all accounts to decrease our program's attack surface.

The `initialize` method also checks that the account has not been initialized before.
This gives us a double guarantee : that we're not attempting to corrupt/overwrite existing data, and that the entire account's data is zeroed out.

Finally, we need to fix the `from_buffer` method. The first 4 lines of the method do not need to be changed. The correct implementation is as follows :

```rust
    pub fn from_buffer(
        // We use the `contract lifetime here since our VestingContract
        // is a set of cast references to this buffer
        buffer: &'contract mut [u8],
        expected_tag: super::Tag,
        // Since we're using a wrapper of references, we return Self
        // and not &mut Self
    ) -> Result<Self, TokenVestingError> {
        let (tag, buffer) = buffer.split_at_mut(8);
        if *bytemuck::from_bytes_mut::<u64>(tag) != expected_tag as u64 {
            return Err(TokenVestingError::DataTypeMismatch.into());
        }
        // The WrappedPodMut trait does all the heavy lifting here
        Ok(Self::from_bytes(buffer))
    }
```

One important thing to know is that we're casting the _entire length_ of the buffer.
This means that the account's allocated length _must_ be of a valid size, otherwise this operation will fail.
<!-- An alternative approach would be to add a `number_of_schedules` field to our `VestingContractHeader`, and then `split_at_mut` the `schedules` buffer again at `number_of_schedules * VestingSchedule::LEN`. -->
To make this less error-prone, we need to write a helper static method called `compute_allocation_size` which determines the valid size for a `VestingContract` data account in terms of its desired number of schedules :

```rust
    pub fn compute_allocation_size(number_of_schedules: usize) -> usize {
        8 + VestingContractHeader::LEN + number_of_schedules * VestingSchedule::LEN
    }
```

You should note that we're always adding 8 bytes to account for the type tag.

`find_key` is the last method we need to take a look at. This method is useful when we want our account's address to be uniquely determined by a set of parameters. For instance, if we wanted to allow for only one vesting contract per recipient, we could write `find_key` as 

```rust
    pub fn find_key(program_id: &Pubkey, recipient: &Pubkey) -> (Pubkey, u8) {
        let seeds: &[&[u8]] = &[Self::SEED, &recipient.to_bytes()];
        Pubkey::find_program_address(seeds, program_id)
    }
```

This means that our vesting contract would be a unique program-derived address (PDA). 
However, in our case, we do not want to add this constraint since any user can hold several vesting contracts.
This is why we won't be using a PDA here, and we can just delete the `find_key` method and its associated `SEED` constant.

If you have been following along, your `src/state/vesting_contract.rs` file should look something like this :

```rust,noplayground
use bonfida_utils::WrappedPodMut;
use bytemuck::{Pod, Zeroable};
use solana_program::pubkey::Pubkey;

use crate::error::TokenVestingError;

#[derive(WrappedPodMut)]
pub struct VestingContract<'a> {
    pub header: &'a mut VestingContractHeader,
    pub schedules: &'a mut [VestingSchedule],
}

#[derive(Clone, Copy, Zeroable, Pod)]
#[repr(C)]
/// Holds vesting contract metadata
pub struct VestingContractHeader {
    /// The eventual token receiver
    pub owner: Pubkey,
    /// The contract escrow vault
    pub vault: Pubkey,
    /// Index in the current schedule vector of the last completed schedule
    pub current_schedule_index: u64,
    /// Used to generate the signing PDA which owns the vault
    pub signer_nonce: u8,
    pub _padding: [u8; 7],
}

#[derive(Clone, Copy, Zeroable, Pod)]
#[repr(C)]
/// An item of the vesting schedule
pub struct VestingSchedule {
    /// When the unlock happens as a UTC timestamp
    pub unlock_timestamp: u64,
    /// The quantity of tokens to unlock from the vault
    pub quantity: u64,
}

impl VestingContractHeader {
    pub const LEN: usize = std::mem::size_of::<Self>();
}

impl VestingSchedule {
    pub const LEN: usize = std::mem::size_of::<Self>();
}

impl<'contract> VestingContract<'contract> {
    /// Initialize a new VestingContract data account
    pub fn initialize(buffer: &mut [u8]) -> Result<(), TokenVestingError> {
        let (tag, _) = buffer.split_at_mut(8);
        let tag: &mut u64 = bytemuck::from_bytes_mut(tag);
        if *tag != super::Tag::Uninitialized as u64 {
            return Err(TokenVestingError::DataTypeMismatch);
        }
        *tag = super::Tag::VestingContract as u64;
        Ok(())
    }

    /// Cast the buffer asa a VestingContract reference wrapper
    pub fn from_buffer(
        buffer: &'contract mut [u8],
        expected_tag: super::Tag,
    ) -> Result<Self, TokenVestingError> {
        let (tag, buffer) = buffer.split_at_mut(8);
        if *bytemuck::from_bytes_mut::<u64>(tag) != expected_tag as u64 {
            return Err(TokenVestingError::DataTypeMismatch);
        }
        Ok(Self::from_bytes(buffer))
    }

    /// Compute a valid allocation size for a VestingContract
    pub fn compute_allocation_size(number_of_schedules: usize) -> usize {
        8 + VestingContractHeader::LEN + number_of_schedules * VestingSchedule::LEN
    }
}


```