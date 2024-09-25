# 					慢雾安全团队 SUI Move 合约审计方法	

![https://img.shields.io/twitter/url/https/twitter.com/slowmist_team.svg?style=social&label=Follow%20%40SlowMist_Team](https://img.shields.io/twitter/url/https/twitter.com/slowmist_team.svg?style=social&label=Follow%20%40SlowMist_Team)

[English Version](./README.md)

- [慢雾安全团队 SUI Move 合约审计方法](#慢雾安全团队-sui-move-合约审计方法)
  - [关键知识点](#关键知识点)
    - [1. 模块声明和可见性](#1-模块声明和可见性)
      - [1.1 `public(friend)` 函数（在最新的Sui的版本中 `public(friend)`替换成 `public(package)`）](#11-publicfriend-函数在最新的sui的版本中-publicfriend替换成-publicpackage)
      - [1.2 `entry` 函数](#12-entry-函数)
      - [1.3 `public` 函数](#13-public-函数)
    - [2. 对象管理](#2-对象管理)
      - [2.1 对象的唯一性](#21-对象的唯一性)
      - [2.2 包装和解包](#22-包装和解包)
      - [2.3 自定义转移策略](#23-自定义转移策略)
      - [2.4 对象的属性](#24-对象的属性)
      - [2.5 对象的权限检查](#25-对象的权限检查)
        - [Address-Owned Objects](#address-owned-objects)
        - [Immutable Objects](#immutable-objects)
        - [Shared Objects](#shared-objects)
        - [Wrapped Objects](#wrapped-objects)
  - [3. 安全性检查](#3-安全性检查)
    - [3.1 数值溢出检查](#31-数值溢出检查)
    - [3.2 重入检查](#32-重入检查)
- [审计入门](#审计入门)
  - [溢出审计 Overflow Audit](#溢出审计-overflow-audit)
  - [算术精度误差审计 Arithmetic Accuracy Deviation Audit](#算术精度误差审计-arithmetic-accuracy-deviation-audit)
  - [条件竞争审计 Race Conditions Audit](#条件竞争审计-race-conditions-audit)
  - [访问控制审计 Access Control Audit](#访问控制审计-access-control-audit)
  - [对象管理审计 Object Management Audit](#对象管理审计-object-management-audit)
  - [Token 代币消耗审计 Token Consumption Audit](#token-代币消耗审计-token-consumption-audit)
  - [闪电贷攻击审计 Flashloan Attack Audit](#闪电贷攻击审计-flashloan-attack-audit)
  - [权限漏洞审计 Permission Vulnerability Audit](#权限漏洞审计-permission-vulnerability-audit)
  - [合约升级的安全审计 Smart Contract Upgrade Security Audit](#合约升级的安全审计-smart-contract-upgrade-security-audit)
  - [外部调用安全审计 External Call Function Security Audit](#外部调用安全审计-external-call-function-security-audit)
  - [返回值的检查 Unchecked Return Values](#返回值的检查-unchecked-return-values)
  - [拒绝服务审计 Denial of Service Audit](#拒绝服务审计-denial-of-service-audit)
  - [Gas 优化审计 Gas Optimization Audit](#gas-优化审计-gas-optimization-audit)
  - [设计逻辑审计 Design Logic Audit](#设计逻辑审计-design-logic-audit)
  - [其他 Others](#其他-others)
- [参考](#参考)


## 关键知识点

### 1. 模块声明和可见性

#### 1.1 `public(friend)` 函数（在最新的Sui的版本中 `public(friend)`替换成 `public(package)`）

**定义**：

`public(friend)`（最新版本中为 `public(package)`）用于声明函数，使其只能被指定的友元模块访问。这提供了更细粒度的访问控制，介于 `public` 和 `private` 之间。

**示例**：

```

public(friend) fun example_function() {
    // 仅友元模块可调用
}
```

#### 1.2 `entry` 函数

**定义**：

`entry` 函数是模块的入口点，允许从事务块中直接调用。参数必须来自事务块的输入，不能是块中先前事务的结果或被修改的数据。此外，`entry` 函数只能返回具有 `drop` 能力的类型。

**示例**：

```

public entry fun transfer_coin(coin: Coin, recipient: address) {
    // 事务入口函数
}
```

#### 1.3 `public` 函数

**定义**：

`public` 函数可以从事务块和其他模块调用，适合外部交互。在参数和返回值上没有和 `entry` 函数一样的限制，通常用于将功能暴露给外部。

**示例**：

```

public fun get_balance(account: &Account): u64 {
    // 允许外部模块和事务块调用
    account.balance
}
```

### 2. 对象管理

#### 2.1 对象的唯一性

**定义**：

每个 SUI 对象都有唯一的 `objID`，确保了对象在链上的唯一性。

#### 2.2 包装和解包

**定义**：

- **直接包装**：将一个 SUI 对象作为另一个对象的字段，解包时必须销毁包装对象。
- **对象包装**：包装后的对象成为另一对象的一部分，不再独立存在。解包后对象的 ID 不变。

#### 2.3 自定义转移策略

**定义**：

​	使用 `sui::transfer::transfer` 定义自定义转移策略，针对具有 `store` 能力的对象，可以通过 `sui::transfer::public_transfer` 创建。

#### 2.4 对象的属性

**定义**：

​	SUI 对象的属性包括 `copy`, `drop`, `store`, 和 `key`，这些属性决定了对象的行为方式。

#### 2.5 对象的权限检查

##### Address-Owned Objects

**定义**：

​	由特定地址（账户地址或对象 ID）拥有的对象，只有对象的所有者可以访问和操作这些对象。

##### Immutable Objects

**定义**：

​	不可变对象无法被修改或转移，任何人都可以访问。适用于需要全局访问但不需要修改的数据。

##### Shared Objects

**定义**：

​	共享对象可以被多个用户访问和操作，适用于去中心化应用等场景，但由于需要共识，操作成本较高。

##### Wrapped Objects

**定义**：

​	包装对象是将一个对象嵌入另一个对象，包装后的对象不再独立存在，必须通过包装对象访问。

## 3. 安全性检查

### 3.1 数值溢出检查

Sui Move 默认进行数值溢出检查。

### 3.2 重入检查

所谓的重入攻击是在一笔正常的合约调用交易中，被插入一笔非预期的(外部)调用，从而改变整体的业务调用流程然后进行非法获利的问题。涉及到外部合约调用的位置，都可能存在潜在的重入风险。目前的重入问题我们可以分为三类：单函数重入、跨函数重入以及跨合约重入。

- Move 中无动态调用，其外部调用都需先通过 use 进行导入，即外部调用都是预期、确定的
- 无 Native 代币转账触发 Fallback 功能
- 在 Move 中，资源模型确保资源一次只能由单个执行上下文访问。这意味着如果函数执行未完成，则在执行完成之前其他函数无法访问同一资源。



# 审计入门



## 溢出审计 Overflow Audit

**说明**

Move 进行数学运算时会进行溢出检查，运算溢出交易将失败。但需要注意的是位运算并不会进行检查。

**定位**

寻找代码中进行位运算的位置，检查是否有溢出风险。



## 算术精度误差审计 Arithmetic Accuracy Deviation Audit

**说明**：

Move 中没有浮点类型。因此，在进行算术运算时，如果运算结果需要以浮点数表示，可能会产生精度误差。虽然精度误差在某些情况下难以完全避免，但可以通过优化和合理设计来减轻其影响。

**定位**：

审查代码中所有涉及算术运算的部分，特别是可能产生精度误差的计算，确保这些运算不会对合约逻辑或数值准确性产生负面影响，并提出优化建议以减轻精度误差。



## 条件竞争审计 Race Conditions Audit

**说明**

Sui 中的验证者也可以对用户提交的交易进行排序，因此在审计中我们仍需要注意在同一个区块中对交易进行排序而进行获利的问题。

**定位**

- 是否有对`函数调用前`合约的数据状态进行预期管理。
- 是否有对`函数执行中`的合约数据状态进行预期管理。
- 是否有对`函数调用后`的合约数据状态进行预期的管理。



## 访问控制审计 Access Control Audit

**说明**：

合约中的某些关键函数应仅限于内部调用，例如能够直接更新用户存款数额的函数。如果这些函数不慎对外部开放，可能绕过权限控制，导致安全漏洞，甚至引发资产损失。因此，必须严格设置访问权限，确保只有授权角色或模块可以调用这些函数，或者确保函数只能在内部使用。

**定位**：

需要检查所有函数的访问控制设置，特别是那些不应该对外公开的函数，确保它们只能在内部调用。如果发现不应暴露的函数接口已经公开，必须标记为高风险并提出修正建议。



## 对象管理审计 Object Management Audit

**说明**：

在 SUI 中，对象可以被转换为共享对象（Shared Object），这意味着对象的访问权限可能从私有变为公共。因此，需要对所有使用的对象进行详细审查，明确每个对象是静态的还是共享的。特别要注意是否有对象被错误地从私有对象转换为共享对象，这样可能导致未经授权的用户访问这些对象，带来潜在的安全风险。

**定位**：

整理并分析模块中所有涉及的对象，检查对象的类型和权限设置，确保该对象的权限与业务需求匹配。如果发现私有对象被错误地转换为共享对象，必须标记为潜在风险并提出修正建议。



## Token 代币消耗审计 Token Consumption Audit

**说明**：

SUI 的代币模型与其他链的代币模型有所不同。SUI 允许对象持有代币，并且代币对象可以嵌套在其他对象中，还可以进行拆分。因此，在涉及代币消耗的场景中，需要特别关注代币的管理和流转，以避免安全问题或意外损失。

**定位**：

在审计代币消耗时，需要检查以下关键点：

1. 消耗的金额是否准确。
2. 代币对象是否已经正确转移。
3. 代币的拆分和合并是否合理。
4. 检查代币与对象的绑定。

## 闪电贷攻击审计 Flashloan Attack Audit

**说明**

Sui 的 Move 也有闪电贷的用法(Hot Potato)。用户可以在一笔交易中借取大量的资金任意使用，只需在这笔交易内归还资金即可。恶意用户通常使用闪电贷放大自身的资金规模进行价格操控等大资金攻击。

**定位**

分析协议本身的算法(奖励、利率等)、预言机依赖是否合理。

## 权限漏洞审计 Permission Vulnerability Audit

**说明**

在 Sui 的 Move 合约中权限漏洞这部分和业务的需求以及功能设计关系较大，所以遇见较为复杂的 Module 的时候，就需要和项目方一一确认各个方法的调用权限。这里的权限一般是指函数的可见性和函数的调用权限。

**定位**

- 检查和确认所有函数方法的可见性及调用权限，在项目评估阶段就需要项目方提供设计文档，审计时候根据设计文档中的描述一一确认权限。

- 梳理项目方角色的权限范围，如果项目方角色的权限会影响用户的资产，则存在权限过大的风险。

- 分析对外函数里面传递进去的对象是什么类型的对象，如果是一些特权函数那必须要有一些特权对象参与。

  

## 合约升级的安全审计 Smart Contract Upgrade Security Audit

**说明**：

在 Move 中，外部模块通过 `use` 关键字导入。需要注意的是，Sui 的合约是可升级的，但发布的合约包（package）是不可变的对象，一旦发布就无法撤回或修改。合约升级的本质是通过在新的地址上重新发布更新的合约，并将旧版本合约的数据迁移至新的合约中。因此，合约升级过程中需要特别注意以下几点：

- **init 函数**：`init` 函数仅在合约第一次发布时执行，后续合约升级时不会再次触发。
- **升级合约不会自动更新依赖**：如果您的合约包依赖于外部包，当外部包升级时，您的合约包不会自动指向升级后的合约地址。您需要手动升级自己的合约包以指定新的依赖项。

**定位**：

需要对合约升级过程中数据迁移的逻辑进行详细检查，确保迁移操作安全、准确，并避免遗漏重要数据或依赖更新的问题。



## 外部调用安全审计 External Call Function Security Audit

**说明**：

与外部模块使用审计项相同，由于 Move 中进行外部调用需要先将外部模块导入，因此理论上外部调用的结果都是开发人员所预期，所需主要的是外部模块的稳定性。

定位

需要对外部引入的库进行检查。



## 返回值的检查 Unchecked Return Values

**说明**：

在 Move 合约中，和其他智能合约语言类似，某些函数的返回值需要进行检查。如果忽略了对这些返回值的处理，可能会导致关键逻辑没有正确执行，进而引发安全问题。

**定位**：

需要检查代码中每个函数调用的返回值，特别是那些涉及外部调用或重要状态更新的函数。如果返回值未被处理或验证，可能会导致不可预期的行为，应该标注为潜在风险点。



## 拒绝服务审计 Denial of Service Audit

**说明**

拒绝服务（DoS）攻击可能由代码逻辑错误、兼容性问题或其他安全漏洞引发，导致智能合约无法正常运行。此类问题可能影响合约的可用性，甚至使其完全瘫痪。

**定位**

- 重点检查业务逻辑的健壮性，确保在各种情况下都能正常执行，不会因错误或漏洞导致合约中断。

- 关注与外部模块交互的部分，确保其兼容性，以防止由于外部依赖问题导致的服务中断。

  

## Gas 优化审计 Gas Optimization Audit

**说明**

与以太坊一样，Sui 也有 Gas 机制，任何模块脚本调用都会消耗 Gas。因此对一些冗长且高复杂度的代码进行优化是有必要的。

**定位**

- 涉及到复杂的调用看是否可以解耦。
- 涉及到高频率的调用看是否可以优化函数内部执行的效率。



## 设计逻辑审计 Design Logic Audit

**说明**：

设计逻辑审计的重点是检查代码中的业务流程和实现，确认是否存在设计缺陷或与预期不符的情况。代码实现如果与预期逻辑不一致，可能会导致意外的行为或安全风险。

**定位**：

- 根据不同角色的权限和作用范围，梳理业务流程中的可能调用路径。
- 确定每个业务流程所涉及的数据范围，确保数据的操作与业务设计一致。
- 将实际的调用路径与预期的业务流程进行比较，识别并分析任何可能导致非预期结果的调用情况。



## 其他 Others

未在上述表述中体现的内容。

# 参考


[Sui book](https://intro.sui-book.com/)

[Sui Doc](https://docs.sui.io/)

[Move Book CN](https://move-dao.github.io/move-book-zh/introduction.html)
