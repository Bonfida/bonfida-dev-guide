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

Let's go through these checks one by one and explain why they are all here. You won't need to necessarily think too deeply about these checks when writing your own programs : you should just ask yourself, how can I constrain these accounts as much as possible using the rudimentary `check_signer`, `check_account_owner` and `check_account_key` security primitives? The tougher the constraint, the smaller the attack surface.

```rust,noplayground
check_account_key(accounts.spl_token_program, &spl_token::ID)?;
```
This check is essential. We'll be performing token transfers using this program and we need to be sure that we're actually calling the write program. An attacker could substitute their own program here and leverage the `source_tokens_owner` signature to take ownership of the token account!

```rust,noplayground
check_account_owner(accounts.vesting_contract, program_id)?;
```
We're going to be editing the `vesting_contract` account data, and we need to be sure that _only our program_ can alter this data. The Solana runtime constraints ensure that every byte of a program-owned account's data is either :
- determined by the program's logic
- freshly allocated and therefore 0
This means that, as long as we trust our own program (which isn't a given considering our own program might be buggy), we should be able to trust the data that's held by its owned accounts.
Failing to perform this check will expose our programs to all sorts of potential attacks.

In reality our program's logic should already make this check redundant since the `VestingContract::initialize` will either attempt to modify the account's data (which is only possible when it is owned by the current progrma), or just fail. 
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

Technically unncessary since the call to `spl_token_program` will take care of these for us.
Don't even _think_ about removing those.

## Parameters