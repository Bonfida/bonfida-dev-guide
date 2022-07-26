# Getting Started

## Installing the Bonfida tool suite

Installing the Bonfida tool suite is quite straightforward.

```bash
cargo install --git https://github.com/Bonfida/bonfida-utils.git cli
```

This command installs the `bonfida` CLI tool, which incorporates a few useful development tools.
You can run `bonfida help` to get a list of available commands.

In order to update the tool, just re-run the above command.

## Creating a new project

To initialize a new project _my-project_ in the current directory, we use the following command:

```bash
bonfida autoproject my-project
cd my-project
```

## Overview of project structure

Each new project is a sort of monorepo containing three folders

```bash
├── js
├── program
└── python
```

- The `js` folder is used to build JavaScript bindings for the current project.
- The `program` folder contains the on-chain Solana program.
- The `python` folder contains Python bindings.

### The program

The summary below describes the basic structure of the program's files, as well as their individual purpose.

```text
program                               
├── Cargo.toml                       
├── src                              
│   ├── cpi.rs                       
│   ├── entrypoint.rs                # Boilerplate for the Solana program's entrypoint
|   |
│   ├── error.rs                     # Custom errors for the program. Varied and 
|   |                                  descriptive errors should be preferred!
|   |
│   ├── instruction.rs               # The instruction enum which serves as a registry 
|   |                                  of supported instructions.
│   │                                  It also contains the Rust bindings for every 
|   |                                  instruction.
|   |
│   ├── lib.rs                       # Structural root of the crate. Contains a 
|   |                                  `declare_id` statement which defines the 
|   |                                  program's reference on-chain key.
|   |
│   ├── processor                    # Folder holding the available instructions' logic
|   |   |
│   │   └── example_instr.rs         # Example instruction. Each instruction file 
|   |                                  follows a strict template to optimize ease of 
|   |                                  audit and general readability.
|   |
│   ├── processor.rs                 # The processor itself is a dispatcher for all 
|   |                                  instructions. The program entrypoint directly 
|   |                                  calls the processor.
|   |
│   ├── state                        # Contains one file per type of program state 
|   |   |                              account. This can range from anything to user 
|   |   |                              accounts or central system state
|   |   |
│   │   ├── example_state_borsh.rs   # An example account type which uses Borsh for 
|   |   |                              serialization and deserialization
|   |   |
│   │   └── example_state_cast.rs    # An example account type which uses direct 
|   |                                  casting. Casting is more efficient in terms of 
|   |                                  compute budget compared to conventional 
|   |                                  serialization/deserialization.
|   | 
│   └── state.rs                     # Contains general utilities related to state 
|                                      accounts. Includes the main registry of account 
| types for this program: the Tag enum |
|--------------------------------------|
└── tests                            # Contains integration tests
    |
    ├── common                       # Contains common integration testing utilities
    │   ├── mod.rs                   
    │   └── utils.rs                 
    └── functional.rs                # The main integration test. Should be used as a 
                                      high-level and primitive test of every 
                                      instruction.


```
