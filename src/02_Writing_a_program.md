# Writing a simple program

In this section, we will work through the writing of a simple token vesting program.
This will enable us to better understand how to write programs using the bonfida toolkit.

## What we are trying to create

Our token vesting program will be a simple solution to a common problem. 
Given a supply of tokens, how can we use on-chain logic to distribute those tokens to investors while making sure that those tokens cannot be used until a predetermined delay, or _schedule_?
The basic idea is that we use an on-chain program (or _smart contract_) to hold the funds and gradually _unlock_ those. 
This allows the whole transaction to be completely trustless: the claimers can trust in the fact that they _will_ receive those tokens on the agreed-upon schedule, and the providers can trust in the fact that the claimers will not bypass the vesting schedule.
In this context, the program acts as a trusted third-party.

Any token vesting transactions thus has two types of users: the claimers, and the providers.
The core logic requires only two types of operation: vesting contract creation, and claiming.
We call those operations the program's _instructions_.

- `create` will initialize a vesting contract for a given quantity of a particular token, with a set schedule.
- `claim` will transfer unlocked funds from a particular vesting contract to its receiver.

The last thing we need is a way for the program to hold state. 
This means that we need a program to _remember_ the active vesting contracts and to hold their associated funds.
In order to hold the vested assets, each token vesting contract has an associated vault.
A vault is a normal token account which is owned by an associated _PDA_, more on that later.

Each vesting contract will thus have an associated state account owned by our program.
These accounts will hold a serialized version of a `VestingContract` object:

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
    /// When to unlock the assets
    pub unlock_timestamp: u64,
    /// What quantity of assets to unlock
    pub quantity: u64
}
```

In reality, we will define the `VestingContract` object quite differently in order to greatly optimize its on-chain serialization and deserialization. This will allow us to handle arbitrarily complex vesting schedules while never running out of compute budget. Thus, the actual `VestingContract` object will be defined in the following way: 

```rust
pub struct VestingContract<'a> {
    pub header: &'a mut VestingContractHeader,
    pub schedule: &'a mut [VestingSchedule]
}

pub struct VestingContractHeader {
    pub owner: Pubkey,
    pub vault: Pubkey,
    pub current_schedule_index: u64,
    pub signer_nonce: u8,
    pub _padding: [u64; 3],
    pub schedule_len: u32
}

pub struct VestingSchedule {
    pub unlock_timestamp: u64,
    pub quantity: u64
}
```
    
We will explore where this added complexity comes from, and why it's actually worth it.