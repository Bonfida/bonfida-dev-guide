# Encoding state

In the previous section, we described our program's architecture in rough terms. 
To begin with, let's look at how we can implement our program's data structures.

## How to interact with data on Solana

With Solana programs, any persistent data needs to be encoded into accounts. 
Within the context of a program, an account is represented by an [`AccountInfo`](https://docs.rs/solana-program/latest/solana_program/account_info/struct.AccountInfo.html) object. 
The data itself is a mutable reference to a slice of bytes. There are several available options to make use of this slice of bytes.

Thus, an account can represent quite a few things. 
For instance, it can be a token account which acts as a certificate of ownership for a particular token.

### Borsh

Using the [`borsh`](https://docs.rs/borsh/latest/borsh/) library, an object can be serialized and deserialized from an array of bytes. 
This approach has several advantages:
- Packed representation: this representation is quite space-efficient.
- Easier to write bindings in languages other than Rust.
- No constraint on memory alignment and padding

The main disadvantage posed by Borsh is that the serialization and deserialization operations have to iterate through the entire data slice, and perform copy operations.
This means that this approach is impractical for larger data structures and can consume a substantial amount of the available compute budget.

While this approach is recommended by Solana for its relative simplicity and good compatibility with existing tooling, for better performance, convenience, and even readability, we represent the following hybrid approach.

### Bytemuck on-chain, Borsh off-chain

The [`bytemuck`](https://docs.rs/bytemuck/latest/bytemuck/) library allows for safe _type casting_ in Rust. 
While type casting is a very common concept in the context of C and C++, it has a bad reputation within the Rust community as being synonymous with memory unsafety, which goes against the language's key design principles. 
However, using `bytemuck` allows us to get the best of both worlds: the efficiency of type casting without its pitfalls.

Type casting is extremely efficient because it doesn't copy any data. 
This means that very large data structures can be handled with no compute overhead _and_ optimal readability. 
When using a serialization approach, it is necessary to _commit_ any changes made to an object to its storage account.
This is an extremely error-prone approach.

These advantages come at the cost of some constraints:
- Certain types have to be aligned to particular offsets.
- Object sizes must be a multiple of their own alignment constraint.

While working with theses kinds of abstract constraints can seem quite daunting, `bytemuck` provides us with convenient derive macros which are able to detect these issues before compilation. By looking at an example, we'll see how these constraints come into play, and how we can deal with them.

Finally, once our data structure is well-defined to play along with `bytemuck`, we can write a compatible `borsh` schema to leverage its implementations across various programming languages.