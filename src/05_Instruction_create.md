# Writing an instruction : create

In the previous sections, we have defined our program's main data structure `VestingContract`.
The next step is to write our program's one of two main primitives : `create`.

We'll start by renaming the `src/processor/example_instr.rs` file to `create.rs`.
Similarly we'll rename the `ExampleInstr` variant of the `instruction::ProgramInstruction` enum to `Create`, and the `instruction::example` binding to `create`.
We should also alter the log message in `processor.rs` from `"Instruction: Example Instruction"` to `"Instruction: Create"`.

This primtive will perform several operations :
- Initialize a new `VestingContract` account/object
- Configure the `VestingContract` with user-provided parameters
- Transfer funds into the program vault

An instruction's specification is defined by its `Accounts` and `Params` objects. Let's start by thinking about what kinds of accounts we'll need, and then we'll take a look at the associated parameters.

## Required accounts

For our vesting contract, we need to know about 
- the eventual token receiver, so we'll need a `recipient` account.
- the account that will hold the `VestingContract` data, the `vesting_contract` account.
We'll need this account to be writable _and_ owned by our current program.
- the escrow vault, the `vault` account. 
We'll need this account to be writable since we're going to transfer funds into it.
It needs to be owned by the `spl_token` program as well.
- the `spl_token_program` account, since we're going to be performing token transfers.
- the `source_tokens` account, which will need to be writable as well.
- the `source_tokens_owner` account, which we'll need as a signer to enable us to successfully transfer the tokens into our vault.

The associated `Accounts` struct thus looks like this :
```rust
#[derive(InstructionsAccount)]
pub struct Accounts<'a, T> {
    /// SPL token program account
    pub spl_token_program: &'a T,

    /// The account which will store the [`VestingContract`] data structure
    #[cons(writable)]
    pub vesting_contract: &'a T,

    /// The contract's escrow vault
    #[cons(writable)]
    pub vault: &'a T,

    #[cons(writable)]
    /// The account currently holding the tokens to be vested
    pub source_tokens: &'a T,

    #[cons(signer)]
    /// The owner of the account currently holding the tokens to be vested
    pub source_tokens_owner: &'a T,

    /// The eventual recipient of the vested tokens
    pub recipient: &'a T,
}
```

The first thing to notice is that the `Accounts` struct is generic over what we actually mean by what an _account_ is. 
The reason why it is generic is to enable the same struct to be used with references to `Pubkey` objects when writing bindings, or `AccountInfo` objects when writing the instruction's logic.

The `InstructionsAccount` trait is useful to auto-generate Rust instruction bindings, which we'll make use of later.

We then need to implement a method to parse the `&[AccountInfo]` slice into our `Accounts` struct. 
We will also use this method to perform rudimentary but _essential_ security checks.
You should understand the reason behind each check in order to familiarize yourself with basic security base practices when developing for Solana.

```rust,noplayground
impl<'a, 'b: 'a> Accounts<'a, AccountInfo<'b>> {
    pub fn parse(
        accounts: &'a [AccountInfo<'b>],
        program_id: &Pubkey,
    ) -> Result<Self, ProgramError> {
        let accounts_iter = &mut accounts.iter();
        let accounts = Accounts {
            spl_token_program: next_account_info(accounts_iter)?,
            vesting_contract: next_account_info(accounts_iter)?,
            vault: next_account_info(accounts_iter)?,
            source_tokens: next_account_info(accounts_iter)?,
            source_tokens_owner: next_account_info(accounts_iter)?,
            recipient: next_account_info(accounts_iter)?,
        };

        // Check keys
        check_account_key(accounts.spl_token_program, &spl_token::ID)?;

        // Check owners
        check_account_owner(accounts.vesting_contract, program_id)?;
        check_account_owner(accounts.vault, &spl_token::ID)?;
        check_account_owner(accounts.source_tokens, &spl_token::ID)?;

        // Check signer
        check_signer(accounts.source_tokens_owner)?;

        Ok(accounts)
    }
}
```
The first part of this method will always be quite similar.
However, the second part is where we can already protect ourselves against quite a few basic attacks (you would be surprised how many programs have been attacked for failing to perform these basic kinds of checks).
These checks are also the first thing an auditor will look for, and having them all in the same place facilitates their work.
Sometimes even auditors can get confused when these checks are not displayed obviously enough!

### Checks

Let's go through these checks one by one and explain why they are all here. 
You won't need to necessarily think too deeply about these checks when writing your own programs, you should just ask yourself: how can I constrain these accounts as much as possible using the rudimentary `check_signer`, `check_account_owner` and `check_account_key` security primitives? 
As general rule of thumb, the tighter the constraint, the smaller the attack surface.

```rust,noplayground
check_account_key(accounts.spl_token_program, &spl_token::ID)?;
```
This check is essential. We'll be performing token transfers using this program and we need to be sure that we're actually calling the write program. 
An attacker could substitute their own program here and leverage the `source_tokens_owner` signature to take ownership of the token account!

```rust,noplayground
check_account_owner(accounts.vesting_contract, program_id)?;
```
We're going to be editing the `vesting_contract` account data, and we need to be sure that _only our program_ can alter this data. 
The Solana runtime constraints ensure that every byte of a program-owned account's data is either :
- determined by the program's logic
- freshly allocated and therefore 0
This means that, as long as we trust our own program (which isn't a given considering our own program might be buggy), we should be able to trust the data that's held by its owned accounts.
Failing to perform this check will expose our programs to all sorts of potential attacks.

In reality our program's logic should already make this check redundant: the `VestingContract::initialize` method will either attempt to modify the account's data (which is only possible when it is owned by the current program), or just fail. 
Does this mean we should remove this check? 
_Absolutely not._ 
Our program's logic might change in the future, and we don't want this metaphorical sword of Damocles to hang over our heads. 
These checks are computationally inexpensive, and extremely valuable.
_Overuse_ them.

```rust,noplayground
check_account_owner(accounts.vault, &spl_token::ID)?;
check_account_owner(accounts.source_tokens, &spl_token::ID)?;
```
and
```rust,noplayground
check_signer(accounts.source_tokens_owner)?;
```

Technically unnecessary since the call to `spl_token_program` will take care of these for us.
Don't even _think_ about removing those.

## Parameters

In addition to the above set of accounts, we need some extra information from the user in order to properly configure the vesting contract.
We only have access to a user-provided slice of bytes, called the _instruction data_.
This slice of bytes can be cast into a `Params` wrapper object.
The approach will be very similar to the way we handled the `VestingContract` object in the previous section.

Let's take a direct look at the `Params` struct :

```rust,noplayground
#[derive(WrappedPod)]
pub struct Params<'a> {
    // Needs to be a `u64` for the schedules slice to be well-aligned in memory
    signer_nonce: &'a u64,
    schedule: &'a [VestingSchedule]
}
```

In order to handle Params being a wrapped Pod in our Rust instruction bindings, we need to activate the `instruction_params_wrapped` feature.
In the program `Cargo.toml`, edit the bonfida-utils dependency to :
```toml
bonfida-utils = {version = 0.2, features = ["instruction_params_wrapped"]}
```

Then in `instruction.rs`, update the `create` binding to :

```rust,noplayground
#[allow(missing_docs)]
pub fn create(accounts: create::Accounts<Pubkey>, params: create::Params) -> Instruction {
    accounts.get_instruction_wrapped_pod(crate::ID, ProgramInstruction::Create as u8, params)
}
```

In combination with the accounts defined above, we have all the information we need to begin writing the instruction logic!

## Instruction Logic

Our entry point into the instruction is always a function `process` with the following signature
```rust,noplayground
pub fn process(program_id: &Pubkey, accounts: &[AccountInfo], params: Params) -> ProgramResult {
    ...
}
```

The first step is to parse our Accounts struct using the `Accounts::parse` method we defined above.
Remember that this method is also responsible for performing basic security checks.
We then unwrap our `params` object into local variables for convenience.

```rust,noplayground
pub fn process(program_id: &Pubkey, accounts: &[AccountInfo], params: Params) -> ProgramResult {
    let accounts = Accounts::parse(accounts, program_id)?;
    let Params { signer_nonce, schedule } = params;
}
```

The first item to take care of is the initialization of the `VestingContract` object.
We can start by checking that the given account is of the correct size :

```rust,noplayground
let expected_vesting_contract_account_size = VestingContract::compute_allocation_size(schedule.len());

    if accounts.vesting_contract.data_len() != expected_vesting_contract_account_size {
        msg!("The vesting contract account is incorrectly sized for the supplied schedule!");
        return Err(ProgramError::InvalidArgument)
    }
```

A refinement to this program would involve taking care of the allocation ourselves, thus making sure that the supplied account will be of the correct size.
However, as long as we supply our users with bindings which can take care of this allocation, this is not really an issue.
Allocating an account within a program is only required when allocating Program-Derived Addresses (PDAs).
As discused earlied, this is not a concern here.

Once we have verified that the account is properly sized, we can actually initialize our `VestingContract` account, and then retrieve it!

```rust,noplayground
// This guard variable is the owned pointer to our actual data
// Once it goes out of scope (or is dropped), all its dependent references are dropped
let mut vesting_contract_guard = accounts.vesting_contract.data.borrow_mut();

VestingContract::initialize(&mut vesting_contract_guard)?;
let vesting_contract = VestingContract::from_buffer(&mut vesting_contract_guard, state::Tag::VestingContract)?;
```

Now that our `VestingContract` object is properly initialized, we need to save the user-provided configuration.

```rust,noplayground
*vesting_contract.header = VestingContractHeader { 
    owner: *accounts.recipient.key, 
    vault: *accounts.vault.key,
    current_schedule_index: 0,
    signer_nonce: *signer_nonce as u8,
    _padding: [0;7] 
};

let mut total_amount = 0u64;
for (schedule, slot) in schedule.iter().zip(vesting_contract.schedules.iter_mut()) {
    *slot = *schedule;
    total_amount = total_amount.checked_add(schedule.quantity).unwrap();
}
```
We keep track of the total amount as it represents what we'll have to transfer into our vault.
Notice that we use `checked_add` to compute our sum.
Using checked math is _absolutely essential_.
Sometimes it might seem redundant.
Sometimes it might actually be redundant.
But I'll say the same thing here I said when we discussed account checks : _dont' even think about it!_

The only exception to this rule is `checked_div` when dividing by a constant.
If you know it to be non-zero because it says so on the same line of code, then you should favor readability.

Finally, we transfer the funds to our vault using the `spl_token` `transfer` instruction :
```rust,noplayground
let instruction = spl_token::instruction::transfer(
    &spl_token::ID, 
    accounts.source_tokens.key, 
    accounts.vault.key, 
    accounts.source_tokens_owner.key, 
    &[], 
    total_amount
)?;

invoke(&instruction, &[
    accounts.spl_token_program.clone(),
    accounts.source_tokens.clone(),
    accounts.vault.clone(),
    accounts.source_tokens_owner.clone()
])?;
```

Regarding the use of `invoke`, the best way to know what kind of accounts to provide is to :
- first add the account for the program we're invoking
- look at the binding code and add all the accounts in order

### We're done... right?

As it stands our program has a major vulnerability which enables a fraudulent vesting contract issuer to dupe their recipient.
They can run away with the entirety of their funds!
If you can't find this vulnerability on your own, that's completely normal and we'll discuss general practices to analyze your code.
Try it anyways, and then let's fix it.
