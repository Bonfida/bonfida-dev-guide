# Introduction

As Bonfida devs, we have been writing Solana programs for some time now. 
As a result, we have come to cultivate a particular _code style_. 
This systematic approach to writing Solana programs has enabled us to write programs faster, with no sacrifice to their security.
If anything, being systematic abour our programs has made them _safer_ and easier to audit.

Our approach is based on several key principles:
- We don't use frameworks, and this is not quite a framework: critical program logic should never be hidden away behind macros.
- Security checks should always be obvious.
- Redundant security and safety checks are better than implicit ones.

This book is intended as a guide to what we have come to define internally as the _Bonfida style_. 
As we learn new and better ways to write programs, it will necessarily evolve.