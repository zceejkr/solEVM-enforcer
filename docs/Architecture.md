# EVM Fraud Proof Architecture

The work builds on [solEVM](https://github.com/Ohalo-Ltd/solevm), an EVM runtime written in Solidity. This document will describe how the Solidy EVM Runtime can be extended to provide the 3 function that are necessary to do on-chain computation verification:

1. Run a program and export state at specific step (number of instructions executed).
2. Initiate the VM at any specific step with state exported in 1.
3. Once initiated as in 2., run the next OP-code and export state again as in 1.


These 3 functions together will allow to run any EVM code step by step, while producing the same results as a continuous run of an EVM. We will compress the exported state of the EVM through a deterministic hashing function. This hash will give us the follow two abilities:

- Ability to compare state of EVM at different states of the binary search (Truebit protocol)
- Ability to verify that EVM has been set up with correct parameters in state 2

## Execution Modes

We distinguish two modes of operation for the implementation we create:

**Off-chain Mode** - This mode is used to initiate the VM with a computation task and instruct it to run until a specific instruction count. The VM should then stop and export the state (function 1. above). For this mode the VM can be invoked with a static call. 

**On-chain Mode** - When the VM is initiated with an intermediate state generated by a run of the off-chain VM and instructed to run only the next OP-code. This invocation needs to fit into an on-chain transaction.

The OP-codes are organized in groups based on which elements of the VM they access:

**Stack-only type OP-codes:**
ADD, MUL, SUB, DIV, SDIV, MOD, SMOD, ADDMOD, MULMOD, EXP, SIGNEXTEND, SHL, SHR, SAR, LT, GT, SLT, SGT, EQ, ISZERO, AND, OR, XOR, NOT, BYTE, POP, DUP, SWAP, 


**Context and stack type OP-codes:**  
ADDRESS, BALANCE, ORIGIN, CALLER, CALLVALUE, GASPRICE, BLOCKHASH, COINBASE, TIMESTAMP, BLOCK_NUMBER, DIFFICULTY, GASLIMIT, JUMP, JUMPI, PC, GAS, RETURNDATASIZE

**Code and stack type OP-codes:**  
CODELOAD, CODESIZE, PUSH1 - PUSH32,

**Data and stack type OP-codes:**  
CALLDATALOAD, CALLDATASIZE  
<--- (!!! at this point we should be able to prove basic contracts !!!) --->

**Memory and Stack type:**  
MLOAD, MSTORE, MSTORE8, MSIZE

**Storage and Stack type OP-codes:**  
SSTORE, SLOAD  
<--- (!!! prove basic contracts with memory and storage !!!) --->

**Context, stack and memory type OP-codes:**  
LOG

**Data, stack and memory type OP-codes:**  
CALLDATACOPY

**Code, stack and memory type OP-codes:**  
CODECOPY

**Return, Stack and Memory type OP-codes:**  
RETURN, REVENT, RETURNDATACOPY  
<--- (!!! most self-contained contract provable !!!) --->

**Extra difficult type:**  
EXTCODECOPY, EXTCODESIZE, CREATE, CALL, DELEGATECALL, STATICCALL, SELFDESTRUCT, PRECOMPILES

the complexity of opcodes differs based on the VM elements they need to access and the complexity of the access (read/write full word vs. read/write dynamic array or struct):

a) OP-codes that access (stack and (context or code or data))
b) OP-codes that access memory and storage
c) OP-codes that access (stack and memory and (context or code or data))
d) OP-codes that modify global state in a complex way like call, create, selfdestroy

## Milestones

The work is split into two milestones in terms of proof size:

1. Simple full state initialization for on-chain VM - in this implementation the different VM elements are initiated with full arrays of data for the different elements like stack, memory, and storage.
2. Compressed proof-based initialization (hydrated state) for on-chain VM - in this implementation only the data which is needed by the next opcode is provided to the VM. Other data is represented by intermediate hashes of the merkle proof.