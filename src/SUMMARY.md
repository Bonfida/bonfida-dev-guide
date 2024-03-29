<!-- markdownlint-disable MD007 -->
<!-- markdownlint-disable MD042 -->
# Summary

[Introduction](README.md)

- [Getting Started](01_Getting_started.md)
- [Writing a program](02_Writing_a_program.md)
    - [Encoding state](02.01_Encoding_state.md)
    - [Writing state: `VestingContract`](02.02_VestingContract.md)
    - [Writing an instruction: `create`](02.03_Instruction_create.md)
        - [Required Accounts](02.03.01_Required_accounts.md)
        - [Parameters](02.03.02_Parameters.md)
        - [Instruction Logic](02.03.03_Instruction_logic.md)
        - [Patching a vulnerability](02.03.04_Instruction_create_patch.md)
            - [Instruction data](02.03.04.01_Instruction_data.md)
            - [Account data](02.03.04.02_Account_data.md)
            - [Checks on third-party state](02.03.04.03_Implementing_checks_on_third-party_state.md)
        - [Conclusion](02.03.05_Conclusion.md)
    - [Writing an instruction: `claim`](02.04_Instruction_claim.md)
    - [Setting up integration testing](02.05_Integration_testing.md)
- [Examples]()
- [FAQ]()

# Using Bonfida tools

- [utils]()
- [autobindings]()
- [autodoc]()
- [benchviz]()
