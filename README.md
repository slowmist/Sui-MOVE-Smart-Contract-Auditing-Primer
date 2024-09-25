# SlowMist Security Team SUI Move Contract Audit Method

![https://img.shields.io/twitter/url/https/twitter.com/slowmist_team.svg?style=social&label=Follow%20%40SlowMist_Team](https://img.shields.io/twitter/url/https/twitter.com/slowmist_team.svg?style=social&label=Follow%20%40SlowMist_Team)

[中文版本](./README_CN.md)
- [SlowMist Security Team SUI Move Contract Audit Method](#slowmist-security-team-sui-move-contract-audit-method)
  - [Key Knowledge Points](#key-knowledge-points)
    - [1. Module Declarations and Visibility](#1-module-declarations-and-visibility)
      - [1.1 `public(friend)` Function (Replaced by `public(package)` in the Latest Version of Sui)](#11-publicfriend-function-replaced-by-publicpackage-in-the-latest-version-of-sui)
      - [1.2 `entry` Function](#12-entry-function)
      - [1.3 `public` Function](#13-public-function)
    - [2. Object Management](#2-object-management)
      - [2.1 Object Uniqueness](#21-object-uniqueness)
      - [2.2 Wrapping and Unwrapping](#22-wrapping-and-unwrapping)
      - [2.3 Custom Transfer Strategy](#23-custom-transfer-strategy)
      - [2.4 Object Properties](#24-object-properties)
      - [2.5 Object Permission Checks](#25-object-permission-checks)
        - [Address-Owned Objects](#address-owned-objects)
        - [Immutable Objects](#immutable-objects)
        - [Shared Objects](#shared-objects)
        - [Wrapped Objects](#wrapped-objects)
  - [3. Security Checks](#3-security-checks)
    - [3.1 Overflow Checks](#31-overflow-checks)
    - [3.2 Reentrancy Checks](#32-reentrancy-checks)
- [Audit Basics](#audit-basics)
  - [Overflow Audit](#overflow-audit)
  - [Arithmetic Accuracy Deviation Audit](#arithmetic-accuracy-deviation-audit)
  - [Race Conditions Audit](#race-conditions-audit)
  - [Access Control Audit](#access-control-audit)
  - [Object Management Audit](#object-management-audit)
  - [Token Consumption Audit](#token-consumption-audit)
  - [Flashloan Attack Audit](#flashloan-attack-audit)
  - [Permission Vulnerability Audit](#permission-vulnerability-audit)
  - [Smart Contract Upgrade Security Audit](#smart-contract-upgrade-security-audit)
  - [External Call Function Security Audit](#external-call-function-security-audit)
  - [Unchecked Return Values](#unchecked-return-values)
  - [Denial of Service Audit](#denial-of-service-audit)
  - [Gas Optimization Audit](#gas-optimization-audit)
  - [Design Logic Audit](#design-logic-audit)
  - [Others](#others)
- [References](#references)


## Key Knowledge Points

### 1. Module Declarations and Visibility

#### 1.1 `public(friend)` Function (Replaced by `public(package)` in the Latest Version of Sui)

**Definition**:

`public(friend)` (replaced by `public(package)` in the latest version) is used to declare functions that can only be accessed by specified friend modules. This provides finer-grained access control, positioned between `public` and `private`.

**Example**:

```
public(friend) fun example_function() {
    // Can only be called by friend modules
}
```

#### 1.2 `entry` Function

**Definition**:

The `entry` function is an entry point of the module, allowing it to be called directly from a transaction block. Parameters must come from the input of the transaction block and cannot be the result of or data modified by previous transactions in the block. Additionally, `entry` functions can only return types with the `drop` ability.

**Example**:

```
public entry fun transfer_coin(coin: Coin, recipient: address) {
    // Entry point of the transaction
}
```

#### 1.3 `public` Function

**Definition**:

`public` functions can be called from transaction blocks and other modules, suitable for external interaction. Unlike `entry` functions, there are no restrictions on parameters and return values, making them commonly used to expose functionality to the outside.

**Example**:

```
public fun get_balance(account: &Account): u64 {
    // Allows external modules and transaction blocks to call
    account.balance
}
```

### 2. Object Management

#### 2.1 Object Uniqueness

**Definition**:

Every SUI object has a unique `objID`, ensuring the uniqueness of objects on the chain.

#### 2.2 Wrapping and Unwrapping

**Definition**:

- **Direct Wrapping**: Embedding a SUI object as a field of another object. To unwrap, the wrapped object must be destroyed.
- **Object Wrapping**: A wrapped object becomes part of another object and no longer exists independently. Unwrapping it does not change the object's ID.

#### 2.3 Custom Transfer Strategy

**Definition**:

 Use `sui::transfer::transfer` to define a custom transfer strategy. For objects with the `store` ability, `sui::transfer::public_transfer` can be created.

#### 2.4 Object Properties

**Definition**:

 SUI object properties include `copy`, `drop`, `store`, and `key`. These properties determine how objects behave.

#### 2.5 Object Permission Checks

##### Address-Owned Objects

**Definition**:

 Objects owned by a specific address (account address or object ID) can only be accessed and manipulated by the owner.

##### Immutable Objects

**Definition**:

 Immutable objects cannot be modified or transferred, but anyone can access them. They are suitable for data that requires global access without modification.

##### Shared Objects

**Definition**:

 Shared objects can be accessed and operated on by multiple users, which is useful for decentralized applications. However, due to the need for consensus, operations are more costly.

##### Wrapped Objects

**Definition**:

 Wrapped objects are objects embedded within another object, and once wrapped, they no longer exist independently. They can only be accessed through the wrapping object.

## 3. Security Checks

### 3.1 Overflow Checks

Sui Move performs overflow checks by default.

### 3.2 Reentrancy Checks

A reentrancy attack is when an unexpected (external) call is inserted into a normal contract transaction, altering the overall flow of the transaction and enabling illegal profit. Any place involving external contract calls may have potential reentrancy risks. Current reentrancy issues can be categorized into three types: single-function reentrancy, cross-function reentrancy, and cross-contract reentrancy.

- Move does not have dynamic calls; all external calls must be imported via `use`, meaning external calls are pre-determined and expected.
- There is no native token transfer triggering fallback functions.
- In Move, the resource model ensures that a resource can only be accessed by a single execution context at a time. This means if a function is not finished executing, other functions cannot access the same resource.

# Audit Basics

## Overflow Audit

**Explanation**:

Move performs overflow checks during mathematical operations, and transactions with overflow will fail. However, bitwise operations do not undergo such checks.

**Positioning**:

Identify locations in the code where bitwise operations are performed and check for potential overflow risks.

## Arithmetic Accuracy Deviation Audit

**Explanation**:

Move does not have floating-point types. Therefore, arithmetic operations that would result in floating-point results may cause precision errors. While precision errors are difficult to entirely avoid in certain cases, they can be mitigated through optimization and careful design.

**Positioning**:

Review all parts of the code involving arithmetic operations, especially those where precision errors may occur. Ensure these operations do not negatively impact contract logic or numerical accuracy and propose optimizations to reduce precision errors.

## Race Conditions Audit

**Explanation**:

In Sui, validators can also sort the transactions submitted by users. Thus, we still need to be aware of issues where transaction ordering in the same block could lead to profit.

**Positioning**:

- Whether there is any expectation management for contract data states before function calls.
- Whether there is expectation management for contract data states during function execution.
- Whether there is expectation management for contract data states after function calls.

## Access Control Audit

**Explanation**:

Some key functions in a contract should be limited to internal calls, such as those that directly update user deposit amounts. If these functions are accidentally exposed to the outside, they might bypass permission control, leading to security vulnerabilities and even asset loss. Therefore, access control must be strictly set to ensure only authorized roles or modules can call these functions, or that they are restricted to internal use only.

**Positioning**:

Check the access control settings for all functions, especially those that should not be exposed externally, to ensure they are restricted to internal use. If any interfaces that should not be exposed are found to be open, mark them as high-risk and propose corrective measures.

## Object Management Audit

**Explanation**:

In SUI, objects can be converted into shared objects, meaning their access rights may change from private to public. It is necessary to carefully review all objects in use to clarify whether each object is static or shared. Special attention should be paid to whether any object has been incorrectly converted from private to shared, which could result in unauthorized access to those objects, posing potential security risks.

**Positioning**:

Sort and analyze all objects involved in the module, check their types and permission settings, and ensure that the object's permissions match business requirements. If any private objects have been incorrectly converted into shared objects, mark them as potential risks and propose corrective actions.

## Token Consumption Audit

**Explanation**:

SUI's token model differs from other chains. SUI allows objects to hold tokens, and token objects can be nested within other objects and split. Therefore, in scenarios involving token consumption, special attention must be paid to token management and circulation to avoid security issues or unexpected losses.

**Positioning**:

When auditing token consumption, check the following key points:

1. Is the consumed amount accurate?
2. Has the token object been correctly transferred?
3. Are the token splits and merges reasonable?
4. Check the binding of tokens to objects.

## Flashloan Attack Audit

**Explanation**:

Sui's Move also supports flashloan functionality (Hot Potato). Users can borrow large amounts of funds in a single transaction for arbitrary use, as long as the funds are returned within the same transaction. Malicious users often use flashloans to amplify their capital to execute large-scale attacks such as price manipulation.

**Positioning**:

Analyze the protocol's algorithms (rewards, interest rates, etc.) and check whether the oracle dependencies are reasonable.

## Permission Vulnerability Audit

**Explanation**:

In Sui's Move contracts, permission vulnerabilities are closely related to business requirements and function design. Therefore, when encountering more complex modules, it is necessary to confirm the permissions of each method with the project team. Permissions generally refer to the visibility and invocation permissions of functions.

**Positioning**:

- Check and confirm the visibility and invocation permissions of all functions. During the project evaluation phase, the project team should provide design documentation, and permissions should be confirmed during the audit based on the documentation.
- Analyze the permissions of the project team's roles. If the roles have permissions that affect user assets, there is a risk of over-permission.
- Analyze the types of objects passed into external functions. If they are privileged functions, privileged objects must be involved.

## Smart Contract Upgrade Security Audit

**Explanation**:

In Move, external modules are imported using the `use` keyword. It is important to note that Sui contracts are upgradable, but the published contract package (package) is an immutable object that cannot be withdrawn or modified once deployed. The essence of contract upgrades is to redeploy an updated contract at a new address and migrate data from the old version of the contract to the new one. Therefore, the following points must be carefully considered during the contract upgrade process:

- **Init Function**: The `init` function is only executed when the contract is first deployed and will not be triggered during subsequent upgrades.
- **Upgrades Do Not Automatically Update Dependencies**: If your contract package depends on an external package, upgrading the external package does not automatically update your contract package. You must manually upgrade your package to point to the new dependency.

**Positioning**:

Carefully review the data migration logic during contract upgrades to ensure the migration is safe and accurate, and avoid issues with missing critical data or dependency updates.

## External Call Function Security Audit

**Explanation**:

This is similar to the external module usage audit. Since external calls in Move require importing external modules, the external call results are theoretically expected by the developer. The main focus is the stability of the external module.

**Positioning**:

Review the external libraries that are imported.

## Unchecked Return Values

**Explanation**:

In Move contracts, similar to other smart contract languages, the return values of certain functions need to be checked. If these return values are ignored, critical logic might not execute properly, leading to security issues.

**Positioning**:

Check the return values of all function calls in the code, especially those involving external calls or important state updates. If return values are not handled or verified, it could result in unexpected behavior and should be flagged as a potential risk.

## Denial of Service Audit

**Explanation**:

Denial of Service (DoS) attacks may be caused by code logic errors, compatibility issues, or other security vulnerabilities, rendering the smart contract unusable. Such issues may affect the contract's availability or even cause it to crash entirely.

**Positioning**:

- Focus on the robustness of business logic to ensure that it can execute normally under various circumstances and is not interrupted by errors or vulnerabilities.
- Pay attention to parts that interact with external modules, ensuring compatibility to avoid service interruptions due to external dependency issues.

## Gas Optimization Audit

**Explanation**:

Like Ethereum, Sui has a gas mechanism, and any module script calls will consume gas. Therefore, optimizing lengthy and complex code is necessary.

**Positioning**:

- For complex calls, check whether they can be decoupled.
- For high-frequency calls, check whether the efficiency of internal function execution can be optimized.

## Design Logic Audit

**Explanation**:

Design logic audits focus on checking the business processes and implementations in the code to ensure there are no design flaws or inconsistencies with expectations. If the code implementation does not align with the intended logic, it may lead to unexpected behaviors or security risks.

**Positioning**:

- Based on the permissions and scope of different roles, sort out the potential invocation paths in the business process.
- Determine the scope of data involved in each business process and ensure that the operations on the data are consistent with the business design.
- Compare the actual invocation paths with the expected business process and identify any cases where unexpected results could occur.

## Others

Anything not covered in the above sections.

# References

[Sui book](https://intro.sui-book.com/)

[Sui Doc](https://docs.sui.io/)

[Move Book CN](https://move-dao.github.io/move-book-zh/introduction.html)
