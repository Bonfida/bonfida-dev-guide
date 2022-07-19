# Patching a vulnerability

As mentioned in the previous section, our program in its current state has a major vulnerability.
It enables a malicious issuer to dupe a recipient into thinking they trustlessly own vested assets, when in reality they don't.
Let's apply a methodical approach to find this vulnerability.

## Thinking about attack vectors

Very broadly speaking, a vulnerability exists when there is some way to manipulate a program's inputs to make it behave in _unexpected_ ways.
This means that patching vulnerabilities means looking at a program's inputs and making sure that those are safe.
Solana programs have two different kinds of input with different security implications :
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

Broadly speaking, your program will interact with two kinds of accounts :
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
To deal with theses kinds of vulnerabilities, a clear coding style and proper reviews and even audits are your friend.

Finally, we have accounts owned by other programs.
More often than not, depending on an other program's accounts means depending on its own safety.
Always think twice before integrating new on-chain dependencies into your projects :
- Is the program well reviewed and audited?
- Is the program widely used?
- If the program is upgradable, can the team be trusted?
- Do you have a good understanding of the program and its safety guarantees?

In the case of our `token-vesting` contract, we depend on the `spl-token` program.
Since this program is part of the official Solana Program Library, and has been around for a while now, we can assume that it's quite safe.
This ticks off the first three requirements.
However, the final point stands, we need to gain an understanding of this program's security guarantees before we can use it in good conscience.
As it just so happens, our program is currently vulnerable because we don't look at what's inside the `vault` spl-token account.

### Understanding your on-chain dependencies: `spl-token`