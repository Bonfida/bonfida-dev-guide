# Patching a vulnerability

As mentioned in the previous section, our program in its current state has a major vulnerability.
It enables a malicious issuer to dupe a recipient into thinking they trustlessly own vested assets, when in reality they don't.
Let's apply a methodical approach to find this vulnerability.

## Thinking about attack vectors

Very broadly speaking, a vulnerability exists when there is some way to manipulate a program's inputs to make it behave in _unexpected_ ways.
This means that patching vulnerabilities means looking at a program's inputs and making sure that those are safe.
Solana programs have two different kinds of input with different security implications:
- Instruction data
- Account data
- System variables

## Instruction data

This attack vector is the first we have to secure, and it is also the simplest.
Securing the instruction data input means making sure that both the deserialization and business logic can handle all possible inputs.
When we use Borsh or bytemuck as deserializers, any bit pattern which does not conform to our schema will result in an error.
This is precisely what we want.
As a corollary, you should be careful when implementing custom deserialization logic.
Borsh and the tools provided by `bonfida-utils` _should_ be enough for your use-case.

The harder part is thinking about the business logic itself. 
You should think about what kind of values your program should allow as input.
Be _as restrictive as possible_.
If a user reports that their own use-case for your product is not handled due to theses restrictions, you can update your program.
If your program's state / assets are compromised, you can still patch the vulnerability, but the incident will probably have cost you.

To secure your program's business logic, we recommend a combination of unit and integration testing.
It can be very useful to use coverage testing tools to make sure that your business logic is sound for every edge case.
We will discuss testing in a later section.

## Account data

The hardest attack vector to secure is an instruction's accounts.
The first step in securing this attack vector is to follow our account checks recommendations detailed in the previous section.
The second and hardest step is to look at account data itself.

Broadly speaking, your program will interact with two kinds of accounts:
- Accounts which it owns (and are therefore part of its own state)
- Accounts owned by other programs

### Thinking about your program's state

To secure the first type of accounts, a simple but very effective strategy is to use account tags.
In the Bonfida style, the first 8 bytes of any account are reserved as a descriptor of _what kind_ of account we're dealing with.
If your program's logic allows for the closing of accounts, a closed account's tag should be set to a `Disabled` value.
This makes sure that those accounts can't be used as an attack vector.

Outside of this notion of account tagging, the safety of an account owned by your own program depends on the safety of your _entire business logic_. 
This is due to the fact that a program-owned account's data is either just zeros or the result of your program's previous interactions with it.
This means that whenever your program modifies an account's data (or its state in general), you should make sure that the state remains safe in all situations.
To deal with theses kinds of vulnerabilities, a clear coding style and proper reviews and even audits are your friends.

Finally, we have accounts owned by other programs.
More often than not, depending on an other program's accounts means depending on this program's own safety.
Always think twice before integrating new on-chain dependencies into your projects:
- Is the program well reviewed and audited?
- Is the program widely used?
- If the program is upgradable, can the team behind it be trusted?
- Do you have a good understanding of the program and its safety guarantees?

In the case of our `token-vesting` contract, we depend on the `spl-token` program.
Since this program is part of the official Solana Program Library, and has been around for a while now, we can assume that it's quite safe.
This ticks off the first three requirements.
However, the final point stands: we need to gain an understanding of this program's security guarantees before we can use it in good conscience.
As it just so happens, our program is currently vulnerable because we don't look at what's inside the `vault` spl-token account.

### Understanding your on-chain dependencies: `spl-token`

Since on-chain programs are security-critical applications, it is essential to take the time to read through the official documentation.
It often contains security recommentations and can provide you with a better idea of what you are dealing with.
In the case of `spl-token`, the documentation can be found [here](https://spl.solana.com/token). 
Since we are going to be using the smart contract's bindings, we should take a look at the library documentation as well.
It can be found [here](https://docs.rs/spl-token/latest/spl_token/index.html).

When using an on-chain dependency, we should gain a deep understanding of the interface we are going to be using.
A program's interface has two components :
- Its instruction specification (what is encoded in `instruction_data`)
- Its state, and how it is encoded into accounts

#### Instruction specification

We're using `spl-token`'s `transfer` instruction.
Let's take a look at the associated binding's documentation [here](https://docs.rs/spl-token/latest/spl_token/instruction/fn.transfer.html).
This is what it looks like as of writing this :

```rust,noplayground
/// Creates a Transfer instruction.
pub fn transfer(
    token_program_id: &Pubkey, 
    source_pubkey: &Pubkey, 
    destination_pubkey: &Pubkey, 
    authority_pubkey: &Pubkey, 
    signer_pubkeys: &[&Pubkey], 
    amount: u64
) -> Result<Instruction, ProgramError>
```

Ah, the documentation is quite lackluster.
Looking at the binding's signature already gives us a bit of information.
However, we want to know as much as possible about this instruction.
We have several options here :
- Find some documentation elsewhere
- Read the instruction code

As a general rule of thumb, there's a pretty straightforward way to find the documentation we need elsewhere.
For some reason, program developers are more often than not given less documentation than frontend ones.
Maybe it's because program development is _supposed_ to be harder?
Really I don't know, and I would encourage any program developer to always think about both web clients and other programs as interfaces to tailor for.
I sincerely hope you're reading this and finding this paragraph outdated.
In the meantime, we can get the information we _deserve_ by looking at the JavaScript/TypeScript documentation [here](https://solana-labs.github.io/solana-program-library/token/js/modules.html#transfer).
This is what it roughly looks like: 

<blockquote>
Transfer tokens from one account to another
<h4 style="margin-top: 4px; margin-bottom: 3px">Parameters</h4>
<ul>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            connection: 
            <span style="font-weight:normal">
                <i>Connection</i>
            </span>
        </h4>
        Connection to use
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            payer: 
            <span style="font-weight:normal">
                <i>Signer</i>
            </span>
        </h4>
        Payer of the transaction fees
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            source: 
            <span style="font-weight:normal">
                <i>PublicKey</i>
            </span>
        </h4>
        Source account
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            destination: 
            <span style="font-weight:normal">
                <i>PublicKey</i>
            </span>
        </h4>
        Destination account
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            owner: 
            <span style="font-weight:normal">
                <i>PublicKey | Signer</i>
            </span>
        </h4>
        Owner of the source account
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            amount: 
            <span style="font-weight:normal">
                <i>number | bigint</i>
            </span>
        </h4>
        Number of tokens to transfer
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            multiSigners: 
            <span style="font-weight:normal">
                <i>Signer</i>[] = []
            </span>
        </h4>
        Signing accounts if owner is a multisig
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            <span style="display: inline-block;padding: 1px 5px;border-radius: 4px;color: rgb(47, 49, 54);background-color: rgb(220, 221, 222);text-indent: 0;font-weight: normal;"> Optional</span> confirmOptions: 
            <span style="font-weight:normal">
                <i>Signer</i>[] = []
            </span>
        </h4>
        Options for confirming the transaction
    </li>
    <li>
        <h4 style="margin-bottom: 0px; margin-top: 3px">
            programId: 
            <span style="font-weight:normal">
                <i>PublicKey</i> = TOKEN_PROGRAM_ID
            </span>
        </h4>
        SPL Token program account
    </li>
</ul>
<h4 style="margin-bottom: 4px">
    Returns
    <span style="font-weight:normal">
        <i>Promise&ltTransactionSignature&gt</i>
    </span>
</h4>
Signature of the confirmed transaction
</blockquote>

We get more information here.
Let's not care about any parameter mentioned here which isn't part of our Rust binding's signature.
This eliminates `connection` and `confirmOptions` from consideration. 
In general, you should make sure that you understand every option, because it's the only way to determine what's important.

Using this knowledge, we can annotate the Rust binding for our particular use-case: transfering funds to a vault:
```rust,noplayground
/// Creates a Transfer instruction.
pub fn transfer(
    /// This will be spl_token::ID since we're using the common spl_token instance
    token_program_id: &Pubkey, 
    /// This will be the key of the provided account currently holding the tokens to vest
    source_pubkey: &Pubkey, 
    /// Our vesting contract's vault key
    destination_pubkey: &Pubkey, 
    /// The owner of the source token account's key
    authority_pubkey: &Pubkey, 
    /// We won't support vesting tokens from a multisig source account.
    /// This will be an empty array.
    signer_pubkeys: &[&Pubkey], 
    /// The total quantity of tokens to vest
    amount: u64
) -> Result<Instruction, ProgramError>
```

We're quite lucky here: the existing call to `transfer` is correct in our `create` instruction!

#### Account specification

Our `create` instruction takes several `spl_token`-owned program accounts as input.
We should therefore understand what kind of accounts we should expect here.
Using the library documentation, we find that `spl_token`'s accounts are all defined in a `state` module [here](https://docs.rs/spl-token/latest/spl_token/state/index.html).
Out of all the different types of state defined by `spl_token`, we're only interested in the `Account` type.
This is because we're only using program accounts which _hold_ tokens.

Let's take a look at the definition of `spl_token::state::Account`:

```rust,noplayground
pub struct Account {
    /// The mint associated with this account
    pub mint: Pubkey,
    /// The owner of this account.
    pub owner: Pubkey,
    /// The amount of tokens this account holds.
    pub amount: u64,
    /// If `delegate` is `Some` then `delegated_amount` represents
    /// the amount authorized by the delegate
    pub delegate: COption<Pubkey>,
    /// The account's state
    pub state: AccountState,
    /// If is_native.is_some, this is a native token, and the value logs the rent-exempt reserve. An
    /// Account is required to be rent-exempt, so the value is used by the Processor to ensure that
    /// wrapped SOL accounts do not drop below this threshold.
    pub is_native: COption<u64>,
    /// The amount delegated
    pub delegated_amount: u64,
    /// Optional authority to close the account.
    pub close_authority: COption<Pubkey>,
}
```

Here we find that every field is properly documented, which will definitely make our job easier!
The next step is to go through each field and think about _constraints_.
How can we use each field to constrain our input accounts as much as possible?
I have annotated the same struct definition to see what kind of constraints we could enforce.

```rust,noplayground
pub struct Account {
    /// The mint associated with this account
    // We could check that our vault and destination mints match
    // However, we know that the spl_token program itself will perform this check
    pub mint: Pubkey, 
    /// The owner of this account.
    // Our vault should be owned by an account controlled by our program!
    // Otherwise the transfer we perform to our vault creates a fake contract
    // since our program cannot control the vault's funds!
    pub owner: Pubkey,
    /// The amount of tokens this account holds.
    // We expect our vault to be empty, so let's make sure.
    pub amount: u64,
    /// If `delegate` is `Some` then `delegated_amount` represents
    /// the amount authorized by the delegate
    // Our vault should not have a delegate authority.
    pub delegate: COption<Pubkey>,
    /// The account's state
    // The account should be Initialized
    pub state: AccountState,
    /// If is_native.is_some, this is a native token, and the value logs the rent-exempt reserve. An
    /// Account is required to be rent-exempt, so the value is used by the Processor to ensure that
    /// wrapped SOL accounts do not drop below this threshold.
    // Not caring about this makes our token vesting contract support wrapped SOL.
    pub is_native: COption<u64>,
    /// The amount delegated
    // This should be 0, but we're already enforcing the delegate field to be None
    pub delegated_amount: u64,
    /// Optional authority to close the account.
    // This should absolutely be None for our vault. 
    // It should be impossible for a third-party to close our vault account.
    pub close_authority: COption<Pubkey>,
}
```

Looking at the `Account` struct, we were able to gain a clearer picture of what we should expect from our `vault` account.
Note that we're mostly focusing on enforcing constraints on the vault account here.
The other token account at play in our `create` instruction only has to emit the tokens.
We can trust this simple constraint to be enforced by `spl_token` itself when we call the `transfer` instruction.

However, our vault account will remain tied to our vesting contract instance for the entirety of its lifetime.
We must therefore make sure that we're getting _precisely_ we need here.
Our vault account can be thought of as a piece of _third-party state_ for our program.
This is why we need to understand it as well as we understand our program's own state.

## Implementing checks on third-party state

In the above discussion, we have determined that our vault should be owned by an account our program controls and can sign for.
This is where program-derived addresses (PDAs) come into play.

### Aside: what are Program-Derived Addresses (PDAs)?

PDAs are Solana's solution for a recurring generic problem in program design : how can our program _own_ anything of value?
Conventionally, owning an asset on a blockchain means controlling a private key the public key of which is set as its _owner_.
Then, users can sign transactions to authorize operations on assets they own.
They key to all of this is that the private key is only known to its owner.

Sometimes, we need a program to own something in the same way that a user does.
However, any information a program has access to is available to anyone, so we can't really give a private key to our program.
What we need is a public key which isn't tied to a private key, but instead to a particular program.
This is precisely what PDAs are.

A PDA is uniquely derived from an array of _seeds_.
To generate a PDA, we can use one of two methods `Pubkey::find_program_address`, or `Pubkey::create_program_address`.
The difference between the two is that `create_program_address` needs _valid_ seeds which generate a valid PDA.
A valid PDA is a Pubkey which we're sure doesn't have a private key.
It lies outside the `ed25519` curve used by Solana.
`find_program_address` will take a seed array and find a `u8` to add to the seed array such that the resulting array is a valid PDA seed.
We call this `u8` the `signer_nonce`.

We can relate the two methods:

```rust,noplayground
let seeds: &[u8] = &[...];
let (k, nonce) = Pubkey::find_program_address(&[seeds], program_id).unwrap();
let k2 = Pubkey::create_program_ddress(&[seeds, &[signer_nonce]]).unwrap();
// k and k2 will be the same
assert_eq!(k, k2);
```

In general, `find_program_address` can be relatively inefficient: it loops over potential nonces until it finds a valid one.
The real issue is that the length of this loop is quite unpredictable.
In most cases, it will be very short.
This means that the entirety of your testing scenarios can only test for short loops, and edge cases can show up in production later down the line.
Therefore, as a general rule of thumb, `find_program_address` should be avoided on-chain because it consumes an unreliable amount of compute budget.
There are exceptions to this rule, especially when PDAs are used as a mapping function.
This is why we ask the user to provide a `signer_nonce`: `find_program_address` is executed off-chain.

A PDA's seeds serve a similar purpose to a private key.
When a program needs to sign for a PDA, it uses the PDA's seeds in a call to `invoke_signed`.
The runtime can then make sure that the PDA to sign for corresponds to those seeds and the calling program.

To add a layer of security, we want our vesting contract instances to be as compartmentalized as possible.
Therefore, we will use the `VestingContract` key as seeds.
Each instance already has its own vault, and this allows each contract to have its own separate signing authority.
The altenative to this is related to the idea of _central state_, which can be required by some use cases and is supported by `bonfida-utils`.

### Writing the `check_vault_account` method

Using the constraints we came up with by analyzing the `spl_token::Account` struct, we can add to `create.rs`:

```rust,noplayground
fn check_vault_account(
    vault: &AccountInfo,
    program_id: &Pubkey,
    contract_key: Pubkey,
    signer_nonce: u8,
) -> Result<(), ProgramError> {
    // We parse the Account struct using the Solana-provided Pack trait
    let vault_account = spl_token::state::Account::unpack(&vault.data.borrow())?;

    let vault_signer =
        Pubkey::create_program_address(&[&contract_key.to_bytes(), &[signer_nonce]], program_id)?;
    let is_valid = vault_account.owner == vault_signer
        && vault_account.amount == 0
        && vault_account.delegate.is_none()
        && vault_account.state == AccountState::Initialized
        && vault_account.close_authority.is_none();
    if !is_valid {
        return Err(TokenVestingError::InvalidVaultAccount.into());
    }
    Ok(())
}
```

If the provided vault account is invalid, we return the custom `InvalidVaultAccount` error.
Let's define it in `error.rs` by modifyin the `TokenVestingError` struct:

```rust,noplayground
#[derive(Clone, Debug, Error, FromPrimitive)]
pub enum TokenVestingError {
    #[error("This account is already initialized")]
    AlreadyInitialized,
    #[error("Data type mismatch")]
    DataTypeMismatch,
    #[error("Wrong account owner")]
    WrongOwner,
    #[error("Account is uninitialized")]
    Uninitialized,
    #[error("The provided vault account is invalid")]
    InvalidVaultAccount,
}
```

Our project does not compile at this point, and we need to edit `TokenVestingError`'s impl of the `PrintProgramError` trait in `entrypoint.rs`:

```rust,noplayground
impl PrintProgramError for TokenVestingError {
    fn print<E>(&self)
    where
        E: 'static + std::error::Error + DecodeError<E> + PrintProgramError + FromPrimitive,
    {
        match self {
            TokenVestingError::AlreadyInitialized => {
                msg!("Error: This account is already initialized")
            }
            TokenVestingError::DataTypeMismatch => msg!("Error: Data type mismatch"),
            TokenVestingError::WrongOwner => msg!("Error: Wrong account owner"),
            TokenVestingError::Uninitialized => msg!("Error: Account is uninitialized"),
            TokenVestingError::InvalidVaultAccount => {
                msg!("Error: The provided vault account is invalid")
            }
        }
    }
}
```

It might seem a bit redundant to have the same error message appear twice in our project.
However, doing it this way prevents our `PrintProgramError` implementation from having to use string formatting which is inefficient on-chain.
The last thing we want is our error messages to be buried under a `ComputationalBudgetExceeded` error.

Finally we just have to insert a call to `check_vault_account` in our `create` instruction's logic:

```rust,noplayground
check_vault_account(
    accounts.vault,
    program_id,
    *accounts.vesting_contract.key,
    signer_nonce,
)?;
```

If you've been following along, your `create.rs` file is in its final form and should look like this:

```rust,noplayground
//! Create a new token vesting contract

use bonfida_utils::{checks::check_account_owner, WrappedPod};
use solana_program::{msg, program::invoke, program_pack::Pack};
use spl_token::state::AccountState;

use crate::{
    error::TokenVestingError,
    state::{
        self,
        vesting_contract::{VestingContract, VestingContractHeader, VestingSchedule},
    },
};

use {
    bonfida_utils::{
        checks::{check_account_key, check_signer},
        InstructionsAccount,
    },
    solana_program::{
        account_info::{next_account_info, AccountInfo},
        entrypoint::ProgramResult,
        program_error::ProgramError,
        pubkey::Pubkey,
    },
};

#[derive(WrappedPod)]
pub struct Params<'a> {
    signer_nonce: &'a u64,
    schedule: &'a [VestingSchedule],
}

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

        // Check signer
        check_signer(accounts.source_tokens_owner)?;

        Ok(accounts)
    }
}

pub fn process(program_id: &Pubkey, accounts: &[AccountInfo], params: Params) -> ProgramResult {
    let accounts = Accounts::parse(accounts, program_id)?;

    let Params {
        signer_nonce,
        schedule,
    } = params;

    // We only want a one-byte signer nonce
    let signer_nonce = *signer_nonce as u8;

    let expected_vesting_contract_account_size =
        VestingContract::compute_allocation_size(schedule.len());

    if accounts.vesting_contract.data_len() != expected_vesting_contract_account_size {
        msg!("The vesting contract account is incorrectly sized for the supplied schedule!");
        return Err(ProgramError::InvalidArgument);
    }

    check_vault_account(
        accounts.vault,
        program_id,
        *accounts.vesting_contract.key,
        signer_nonce,
    )?;

    let mut vesting_contract_guard = accounts.vesting_contract.data.borrow_mut();

    VestingContract::initialize(&mut vesting_contract_guard)?;
    let vesting_contract =
        VestingContract::from_buffer(&mut vesting_contract_guard, state::Tag::VestingContract)?;

    *vesting_contract.header = VestingContractHeader {
        owner: *accounts.recipient.key,
        vault: *accounts.vault.key,
        current_schedule_index: 0,
        signer_nonce,
        _padding: [0; 7],
    };

    let mut total_amount = 0u64;
    for (schedule, slot) in schedule.iter().zip(vesting_contract.schedules.iter_mut()) {
        *slot = *schedule;
        total_amount = total_amount.checked_add(schedule.quantity).unwrap();
    }

    let instruction = spl_token::instruction::transfer(
        &spl_token::ID,
        accounts.source_tokens.key,
        accounts.vault.key,
        accounts.source_tokens_owner.key,
        &[],
        total_amount,
    )?;

    invoke(
        &instruction,
        &[
            accounts.spl_token_program.clone(),
            accounts.source_tokens.clone(),
            accounts.vault.clone(),
            accounts.source_tokens_owner.clone(),
        ],
    )?;

    Ok(())
}

fn check_vault_account(
    vault: &AccountInfo,
    program_id: &Pubkey,
    contract_key: Pubkey,
    signer_nonce: u8,
) -> Result<(), ProgramError> {
    let vault_account = spl_token::state::Account::unpack(&vault.data.borrow())?;

    let vault_signer =
        Pubkey::create_program_address(&[&contract_key.to_bytes(), &[signer_nonce]], program_id)?;
    let is_valid = vault_account.owner == vault_signer
        && vault_account.amount == 0
        && vault_account.delegate.is_none()
        && vault_account.state == AccountState::Initialized
        && vault_account.close_authority.is_none();
    if !is_valid {
        return Err(TokenVestingError::InvalidVaultAccount.into());
    }
    Ok(())
}
```