# 1 Introduction

This document is a review of Pareto Network’s ERC-20 token contract supporting their decentralized ecosystem. The `ParetoERC20.sol` file contains 348 lines of Solidity code. All of the functions and state variables are well commented using the `natspec` documentation format,  which is a good standard for quickly understanding how everything is supposed to work. The contract implements all the ERC-20 standard functions, events and state variables, and explicitly defines the visibility of each function. AllCode reviewed the contract from a technical perspective looking for bugs in the codebase. Overall we recommend minor feature enhancements and a few improvements which will reduce risks. See more Below.


# 2 Files Audited

We evaluated the Pareto ERC-20 token contract file at: https://github.com/ParetoNetwork/ParetoTokenContract/blob/master/ParetoERC20.sol

# 3 Disclaimer

The audit makes no statements or warrants about utility of the code, safety of the code, suitability of the business model, regulatory regime for the business model, or any other statements about fitness of the contracts to purpose, or their bug free status. The audit documentation is for discussion purposes only.

# 4 Executive Summary

This token contract’s code is very clean, thoughtfully written and in general well architected. The contract only possess a minor vulnerability which has been described in detail in the discussion section of this audit. The code conforms closely to the documentation and specification.

The Pareto token contract inherits many contracts from the OpenZeppelin codebase. In general, OpenZeppelin’s codebase is good and secure, and this is a relatively safe start.




# 5 Vulnerabilities

## 5.1 Critical Vulnerabilities

There is only a single critical vulnerability in the constructor of the Pareto contract. When `_value` tokens are transferred from `owner` to `distributionAddress`, there is no firing of corresponding `Transfer` event. This absence of `Transfer` event results in unlogged transfers of tokens and poses issues of incorrect calculations when the number of token holders or number of transfers for the token contract are calculated. Both the data on `Etherscan` against `owner` and `distributionAddress` will be shown incorrect as Etherscan calculates and shows data collected from event logs. Also, as the `Transfer` event in discussion is not fired, no data is logged, and there is no existence of this record on the Ethereum blockchain.

## 5.2 Moderate Vulnerabilities

As written, the contracts are vulnerable to two common issues: a short address attack; and a double-spend attack on approvals.

### 5.2.1 Short Address Attack

Recently the Golem team discovered that an exchange wasn’t validating user-entered addresses on transfers. Due to the way `msg.data` is interpreted, it was possible to enter a shortened address, which would cause the server to construct a transfer transaction that would appear correct to server-side code, but would actually transfer a much larger amount than expected.

This attack can be entirely prevented by doing a length check on `msg.data`. In the case of `transfer()`, the length should be 68:

                                       `assert(msg.data.length == 68);`

Vulnerable functions include all those whose last two parameters are an address, followed by a value.

In ERC20 these functions include `transfer`, `transferFrom` and `approve`.

A general way to implement this is with a modifier (slightly modified from the one suggested by redditor izqui9):




                                   modifier onlyPayloadSize(uint numwords) {
                                        assert(msg.data.length == numwords * 32 + 4);
                                        _;
                                   }

                         function transfer(address _to, uint256 _value) onlyPayloadSize(2) { }

If an exploit of this nature were to succeed, it would arguably be the fault of the exchange, or whoever else improperly constructed the oﬀending transactions. However, we believe in defense in depth. It’s easy and desirable to make tokens which cannot be stolen this way, even from poorly-coded exchanges.

Further explanation of this attack is here: http://vessenes.com/the-erc20-short-address-attack-explained/


### Testing and testability

This audit will examine how easily tested the code is, and review how thoroughly
tested the code is.

## About airswap

airswap is a decentralised exchange that uses offchain signatures to authorise
swaps of tokens between independent parties. Users are divided into 'makers' and
'takers'; makers broadcast intent to trade and sign messages authorising
trades, while takers agree to a specified trade by submitting it to the airswap
contract.

The airswap token contract provides an ERC20 compatible token contract, with additional features to lock up tokens for fixed periods.

## Terminology

This audit uses the following terminology.

### Likelihood

How likely a bug is to be encountered or exploited in the wild, as specified by the
[OWASP risk rating methodology](https://www.owasp.org/index.php/OWASP_Risk_Rating_Methodology#Step_2:_Factors_for_Estimating_Likelihood).

### Impact

The impact a bug would have if exploited, as specified by the
[OWASP risk rating methodology](https://www.owasp.org/index.php/OWASP_Risk_Rating_Methodology#Step_3:_Factors_for_Estimating_Impact).

### Severity

How serious the issue is, derived from Likelihood and Impact as specified by the [OWASP risk rating methodology](https://www.owasp.org/index.php/OWASP_Risk_Rating_Methodology#Step_4:_Determining_the_Severity_of_the_Risk).

# Overview

## Source Code

The airswap smart contract source code was made available in the [airswap/protocol](https://github.com/airswap/protocol/) Github repository.

The code was audited as of commit `6bdee4e9ab0bfdb4e5af1cd2a0133294f037e269`.

The following files were audited:
```
SHA1(contracts/AirSwapToken.sol)= 72a6aeb815c4e7a31de5730edd926b1f5b74769c
SHA1(contracts/lib/StandardToken.sol)= 2d8310edcaaac100cc1a830f2506b0b1010af54e
SHA1(contracts/lib/Token.sol)= 757c7ca8ea7169b04305f1c02b4965a1237f01d9
```

## General Notes

The code is readable and was easy to audit, with well placed comments and small, clearly written functions. Several `require` conditions that were used in several places could have been refactored into modifiers for further clarity and less repetition of code.

We note that the token locking mechanism could be bypassed by creating a wrapping token that lacks the lock mechanism, and permits people to transfer tokens to and from the wrapping token contract. Whether this is a concern or not depends on the overall goals of the locking mechanism.

It is also worth noting that the locking mechanism utilises a fixed lock-up period; this could be generalised to a variable lock-up period with no increase in complexity by simply making the `BalanceLock` structure store an unlock date instead of a lock date, and accepting an unlock date in the `lockTokens` function.

## Contracts

`AirSwapToken` implements the token functionality.

## Testing

A fairly complete set of unit tests is provided. Unit tests are clearly written
and easy to follow. Happy cases are tested along with most simple failures.

Notably, initial token lockup time is hardcoded into the token contract, rather than supplied as a cconstructor parameter, and is set to a fixed timestamp, which the tests validate. As a result, these unit tests will begin failing after the lockup period concludes. Tests should not depend on wallclock time, and this should be remedied by either using fake time, or making the token contract accept an unlock time as a constructor argument.

Testing requires running a separate server process. No automated build is set up
for the repository.

We recommend setting up an automated build, so new commits can be vetted against
the existing test suite.

# Findings

We found one note issue, four low issues, and one medium issue.

## Note Issues

### Choice of 8 decimal places

 - Likelihood: low
 - Severity: low

The token contract makes a seemingly arbitrary choice of 8 decimal places. While this is unlikely to cause significant issues, we recommend the consensus value of 18 decimal places, matching Ether and a majority of other tokens, absent any compelling reason to choose another value.

## Low Issues

### Use of `if(...) revert()` instead of `require`

 - Likelihood: medium
 - Severity: low

The `onlyAfter` modifier uses the `if(...) revert()` construction. Generally, using `require` is recommended instead, as it aids both readability and analysis by static analysis tools.

### Extending the locking period requires increasing number of locked tokens

 - Likelihood: medium
 - Severity: low

Because the only way to extend the locking period is calling `lockBalance`, and because this requires a `_value` argument greater than zero, users cannot extend the period for which their tokens are locked without also increasing the number of locked tokens.

We recommend either removing the `>0` check, or making the `_value` argument a total rather than a delta; this would have the additional benefit of making multiple calls to `lockBalance` and `unlockBalance` idempotent.

### Expired token locks must be explicitly unlocked

 - Likelihood: medium
 - Severity: low

Once a user's locked tokens have expired, they must manually call `unlockTokens` to make those tokens available for use.

We recommend removing `unlockTokens` entirely, and instead creating a `spendableBalance` property that relevant functionality depends on to determine how many tokens can be spent.

An additional benefit of resolving this and the previous issue would be to allow users with expired locks to create new locks for a smaller number of tokens in a single operation.

### `approve` function breaks ERC20

 - Likelihood: low
 - Severity: medium

The approve function throws if the caller attempts to change an approval unless either the previous approval amount or the new approval amount is zero. This is intended to prevent a known edge case with ERC20 approvals, but results in violating ERC20.

The approvals race condition does not affect other contracts, which can read and adjust approvals in a single atomic operation, and calling contracts may be unaware of this check, which violates the interface specification laid out in ERC20. As a result, these contracts may be rendered unusable with this token.

We recommend leaving out this check, and making it the responsibility of user software that issues approvals to responsibly zero them out and check them before reauthorisation.

## Medium Issues

### Callers have no way to determine when tokens become transferrable

 - Likelihood: medium
 - Severity: medium

While the `balanceLocks` mapping is publicly readable, it stores the lock date of a user's tokens, rather than the unlock date. Further, the `lockingPeriod` variable is not publicly readable. As a result, external contracts have no way to know when tokens become transferrable, except by convention (Eg, out-of-band knowledge of the contract's locking period).

We recommend either making `lockingPeriod` public, or preferably, making the `balanceLocks` date refer to the unlock date rather than the lock date.

## High Issues

None found.

## Critical Issues

None found.
