---
title: ERC-4337 Account Abstraction Using Alt Mempool
description: A standard for aacount abstraction without consesus layer changes.
lang: en
---

## Introduction {#introduction}
ERC-4337 is an Ethereum standard that enables account abstraction, transforming user accounts into smart accounts without any changes to the consensus layer. The ERC-433 standard allows a single account to have the combined functionalities of smart contracts and Externally Owned Accounts (EOAs), thereby opening up new possibilities for wallet design & more user-friendly and accessible blockchain experience.

**Understanding Account Abstraction Smart Contract.**

Unlike External Accounts that hold one set of cryptographic keys, Contract Account wallets are programmed by smart contracts, providing users with more control over their funds. The ERC-4337 standard merges the two account types functionalities, allowing users to get rid of the legacy handling of user accounts.

While EIP 2938 proposes changes to the bottom-layer transaction type, The ERC-4337 account abstraction introduces a higher-layer pseudo-transaction object called a UserOperation. This object sends data into a separate mempool, which is then managed by bundlers. These bundlers are responsible for taking user operations from the mempool and including them in blocks on the Ethereum network.

## Prerequisites {#prerequisites}
To better understand this page, we recommend you first read Accounts, ERC-20, and EIP 4337.

## Function & Features on the ERC-4337 Standard: {#body}

### UserOperation {#UserOperation}
The structure(not called transaction) that describes a user-initiated action to be executed on the blockchain network.
It contains fields such as "sender", "to", "calldata", "maxFeePerGas", "maxPriorityFee", "signature", "nonce."

**Note:** the "nonce" and "signature" fields usage is defined by each account implementation rather than by the protocol.

Other unique fields include entrypoint, bundlers, and aggregators.

### EntryPoint {#EntryPoint}
Is a singleton contract for executing bundles of UserOperations. Bundlers/Clients whitelist supported entrypoints.
Interphase of the entrypoint Contract:

```
function handleOps(UserOperation[] calldata ops, address payable beneficiary);

function handleAggregatedOps(
    UserOpsPerAggregator[] calldata opsPerAggregator,
    address payable beneficiary
);

    
struct UserOpsPerAggregator {
    UserOperation[] userOps;
    IAggregator aggregator;
    bytes signature;
}
function simulateValidation(UserOperation calldata userOp);

error ValidationResult(ReturnInfo returnInfo,
    StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo);

error ValidationResultWithAggregation(ReturnInfo returnInfo,
    StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo,
    AggregatorStakeInfo aggregatorInfo);

struct ReturnInfo {
  uint256 preOpGas;
  uint256 prefund;
  bool sigFailed;
  uint48 validAfter;
  uint48 validUntil;
  bytes paymasterContext;
}

struct StakeInfo {
  uint256 stake;
  uint256 unstakeDelaySec;
}

struct AggregatorStakeInfo {
    address actualAggregator;
    StakeInfo stakeInfo;
}
```

### Bundler {#Bundler}

Is a node that bundles UserOperations and creates a transaction for EntryPoint. Not all nodes need to be bundlers. A bundled transaction combines several UserOperation objects into a single call to a globally published entry point contract using the handleOps function.

### Aggregator {#Aggregator}

Is a trusted contract for validating aggregated signatures. Bundlers/Clients whitelist supported aggregators.
Interphase of the entrypoint Contract:

```
The core interface required by an aggregator is:

interface IAggregator {

  function validateUserOpSignature(UserOperation calldata userOp)
  external view returns (bytes memory sigForUserOp);

  function aggregateSignatures(UserOperation[] calldata userOps) external view returns (bytes memory aggregatesSignature);

  function validateSignatures(UserOperation[] calldata userOps, bytes calldata signature) view external;
}
```

### validateUserOp Function {#validateUserOp Function{}

The validateUserOp function takes in a UserOp object as input and returns a boolean value indicating whether the operation is valid or not. The function performs various checks, such as verifying the signature of the user who submitted the operation and checking for any potential errors or inconsistencies in the operation data.
The validateUserOp function also enables wallets to act as smart contracts. This allows for more complex and customizable interactions with the blockchain network, as well as additional security and features that are not available in traditional, non-smart contract wallets.

```
contract MyContract {
    
    struct UserOp {
        address user;
        bytes data;
        bytes signature;
    }
    
    function validateUserOp(UserOp memory op) public view returns (bool) {
        // Check that the user address matches the recovered signer from the signature
        address recovered = recover(op.signature, op.data);
        if (recovered != op.user) {
            return false;
        }
        
        // Check for any other conditions required for the operation to be considered valid
        // ...
        
        return true;
    }
    
    function recover(bytes memory signature, bytes memory data) internal pure returns (address) {
        bytes32 messageHash = keccak256(data);
        bytes32 r;
        bytes32 s;
        uint8 v;
        
        // Split the signature into its components
        assembly {
            r := mload(add(signature, 32))
            s := mload(add(signature, 64))
            v := byte(0, mload(add(signature, 96)))
        }
        
        // Calculate the expected Ethereum message prefix
        bytes memory prefix = "\x19Ethereum Signed Message:\n32";
        bytes32 prefixedHash = keccak256(abi.encodePacked(prefix, messageHash));
        
        // Recover the signer's address from the signature components
        address signer = ecrecover(prefixedHash, v, r, s);
        
        return signer;
    }
}
```

This implementation defines a UserOp struct to represent a user operation, which includes the user's address, the operation data, and the user's signature. The validateUserOp function takes in a UserOp object as input, and checks that the user address matches the recovered signer from the signature, as well as any other conditions required for the operation to be considered valid.
