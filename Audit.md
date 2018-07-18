# 1. Introduction

This document is a security audit of Pareto Network’s ERC-20 token contract supporting their decentralized ecosystem.

The scope of the security audit was restricted to:

* Scan the contracts listed above for generic security issues using automated systems and manually inspecting the results.
* Manually auditing the contracts listed above for security issues.

# 2. Files Audited

We evaluated the Pareto ERC-20 token contract file that resides on [Etherscan](https://etherscan.io/address/0xea5f88e54d982cbb0c441cde4e79bc305e5b43bc#code)

The Githuhb version used for this report is
[fd5c927392350d6e7dcee4549dc4a790e31d0ea1#diff-1b12214caabec7083a5d2d9c8b004846](https://github.com/ParetoNetwork/ParetoTokenContract/blob/master/ParetoERC20.sol)

# 3. Disclaimer

The audit makes no statements or warrants about utility of the code, safety of the code, suitability of the business model, regulatory regime for the business model, or any other statements about fitness of the contracts to purpose, or their bug free status. The audit documentation is for discussion purposes only.

# 4. Executive Summary

The contract implements all of the ERC-20 standard functions, events and state variables, and explicitly defines the visibility of each function. [AllCode](https://www.allcode.com) reviewed the contract from a technical perspective looking for security issues in the source code. Overall, we recommend minor feature enhancements and a few improvements which will reduce risks.

This token contract’s code is clean, thoughtfully writtenm and in general, well architected. The contract only possess a minor vulnerability which has been described in detail in the discussion section of this audit. The code conforms closely to the documentation and specification.

The Pareto token contract inherits many contracts from the [OpenZeppelin](https://github.com/OpenZeppelin) codebase, which is an industry standard.

# 5. Vulnerabilities

## 5.1 Critical Vulnerabilities

There is a single critical vulnerability in the constructor of the Pareto contract. When `_value` tokens are transferred from `owner` to `distributionAddress`, the `Transfer` event is not fired. The absence of the `Transfer` event results in the transfer of tokens not being logged to the blockchain, and poses issues of incorrect calculations when the number of token holders or number of transfers for the token contract are calculated. Both the data on `Etherscan` against `owner` and `distributionAddress` will be shown incorrect as Etherscan calculates and shows data collected from event logs.

## 5.2 Moderate Vulnerabilities

As written, the contracts are vulnerable to two common issues: a short address attack; and a double-spend attack on approvals.

### 5.2.1 Short Address Attack

Recently the Golem team discovered that an exchange wasn’t validating user-entered addresses on transfers. Due to the way `msg.data` is interpreted, it was possible to enter a shortened address, which would cause the server to construct a transfer transaction that would appear correct to server-side code, but would actually transfer a much larger amount than expected.

This attack can be entirely prevented by doing a length check on `msg.data`. In the case of `transfer()`, the length should be 68:

>assert(msg.data.length == 68);

Vulnerable functions include all functions whose last two parameters are an address, followed by a value. In ERC20 these functions include `transfer`, `transferFrom` and `approve`.

A general way to implement this is with a modifier (slightly modified from the one suggested by redditor izqui9):

```
modifier onlyPayloadSize(uint numwords) {
  assert(msg.data.length == numwords * 32 + 4);
  _;
}

function transfer(address _to, uint256 _value) onlyPayloadSize(2) { }
```

If an exploit of this nature were to succeed, it would arguably be the fault of the exchange, or whomever else improperly constructed the oﬀending transactions. We believe in defense in depth. It’s easy and desirable to make tokens which cannot be stolen this way, even from poorly-coded exchanges.

Further explanation of this attack is [here]() http://vessenes.com/the-erc20-short-address-attack-explained/)


### 5.2.2 Double-spend on Approval

Imagine that Alice approves Mallory to spend 100 tokens. Later, Alice decides to approve Mallory to spend 150 tokens. If Mallory is monitoring pending transactions,  when Mallory sees Alice’s new approval, she can attempt to quickly spend 100 tokens, racing to get her transaction mined prior to Alice’s new approval being written to the blockchain. If Mallory's transaction beats Alice’s, then she can spend another 150 tokens after Alice’s transaction for approval of 150 tokens.

This issue is a consequence of the ERC20 standard, which specifies that `approve()`takes a replacement value, but no prior value. Preventing the attack while complying with ERC20 involves a compromise, The users should set the approval to zero, make sure Mallory hasn’t snuck in a spend, and then set the new value.

In general, this sort of attack is possible with functions which do not encode enough prior state. In this case, Alice’s baseline belief of Mallory’s outstanding spent token balance from the Mallory allowance.

It’s possible for `approve()` to enforce this behavior without API changes in the ERC20 specification:

```
if ((_value != 0) && (approved[msg.sender][_spender] != 0)) return false;
```

However, this is just an attempt to modify user behavior. If the user does attempt to change from a non-zero value to another, then the double spend can still happen, since the attacker will set the value to zero.

If desired, a nonstandard function can be added to minimize the hassle for users. The issue can be fixed with minimal inconvenience by taking a change value rather than a replacement value:

```
function increaseApproval (address _spender, uint256 _addedValue) onlyPayloadSize(2) returns (bool success) {
  uint oldValue = approved[msg.sender][_spender];
  approved[msg.sender][_spender] = safeAdd(oldValue, _addedValue);
  return true;
}
```

Even if this function is added, it’s important to keep the original for compatibility with the ERC20 specification.

The likely impact of this bug is low for most situations.

For more, see this discussion on [Github] (https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729)

### 5.2.3 Reentrancy attack

In this vector, the Ether receiving fallback function of a contract can be used to re-enter the sending contract’s function and perform malicious activity if the code of calling contract is not secure such as the calling contract does not update its state before making an external contract or raw call to another contract.

The Pareto contract does not use `call.value()` for external calls but uses `send()` in `reclaimEther()` function, which is safe as there is no risk of re-entrancy attacks since the `send()` function only forwards `2,300 gas` to the `fallback` function of called contract. This amount of gas can only be used to log an event’s data and throw a failure. This way you’re unable to recursively call the sender function again, thus avoiding the re-entrancy attack.

##  5.2 Low Vulnerabilities

### 5.3.1 Prevent token transfers to 0x0 address or the Pareto contract address

At the time of writing, the "zero" address (0x0000000000000000000000000000000000000000) holds tokens with a value of more than 80$ million. All of these tokens are burnt or lost.

The transfer of Pareto Network tokens to the Pareto token contract would also result in the tokens being stuck.

An example of the potential for loss by leaving this open is the EOS token smart contract where more than 90,000 tokens are stuck at the contract address.

An example of implementing both the above recommendations would be to create the following modifier; validating that the "to" address is neither 0x0 nor the smart contract's own address:

```
modifier validDestination( address to ) {
  require(to != address(0x0));
  require(to != address(this) );
  _;
}
```

The modifier should then be applied to the "transfer" and "transferFrom" methods:

```
function transfer(address _to, uint _value)
  validDestination(_to)
  returns (bool) {
  (... your logic ...)
  }

function transferFrom(address _from, address _to, uint _value)
  validDestination(_to)
  returns (bool) {
    (... your logic ...)
  }
```


#  6. General Comments

##  6.1 Use of ‘approve’ Function

As mentioned earlier, the double spending problem resulting from the use `approve()` function, the Pareto token contract deals with it by introducing the functions `increaseApproval()` and `decreaseApproval()` for increasing and decreasing the approvals, respectively. Both of these functions eliminate the need to reassign allowance but the `approve()` function still does not protect against reassignment to a non-zero approved value if mistakenly called by the approver. The solution would be to force the user to first set the `allowance` value to zero before setting a non-zero new value.

```
function approve (address _spender, uint256 _value)  returns (bool success) {
  if ((_value != 0) && (allowance_of[msg.sender][_spender] != 0)) return false;
    //...
}
```            

##  6.2 Reclaiming Stranded Tokens

We recommend implementing the contract with the ability to call transfer on arbitrary ERC20 token contracts in case tokens are stranded there.

As an example, the Pareto token contract has been sent Pareto (presumably accidentally). We generalize this to Peterson’s Law: “Tokens will be sent to the most inconvenient address possible.”

The Pareto token contract does not have any mechanism to deal with stranded tokens.

##  6.3 Reclaiming Stranded Ether

It is very rare but possible to see Ether sent to a contract that is not payable through use of the `selfdestruct` function. For this reason we recommend allowing all contracts that are not holding Ether for other reasons to allow the owner to withdraw the funds.

The Pareto token contract inherits a contract from OpenZepplin called `HasNoEther` which defines a constructor rejecting any Ethers sent with deployment of contract and has a function `reclaimEther` which sends the Ether balance of Pareto token contract to the owner address. Both reclaims, tokens and Ethers can be combined into a single function – An example of that is described below:

```
function claimTokens(address _token) onlyOwner {
  if (_token == 0x0) {
    owner.transfer(this.balance);
    return;
  }

  Token token = Token(_token);
  uint balance = token.balanceOf(this);
  token.transfer(owner, balance);
  logTokenTransfer(_token, owner, balance);
}
```

##  6.4 Absence of ‘emit’ with events

Firing events without the ‘emit’ keyword has been deprecated. The ‘emit’ keyword should be added preceding each event firing, so the contract complies to latest versions of Solidity.

Example:

```
`emit Transfer(0x00, owner, totalSupply_);`
```

# 7. Line By Line Discussion

##  7.1 Line 327

The value of variable `distributionAddress` should be casted to `address` data-type.

```
address distributionAddress = address( 0x005d6E4E4921904641Da66d4f05a023b70E89d58 )
```

##  7.2 Line 329 - 330

Although, the `SafeMath` library is part of the contract, the value in `balances` mapping is not being updated using the library. The SafeMath library should be used here to avoid overflow of values.

The final code would be as follows:

```
balances[owner] = balances[owner].sub(_value);
balances[distributionAddress] = balances[distributionAddress].add(_value);
```

##  7.3 Line 331

At line 331, there should be a `Transfer` event fired for the transfer of tokens from `owner` to `distributionAddress`.

```
Transfer(owner, distributionAddress, _value);
```

As discussed above, without this event many exchanges would be showing the wrong token values for either `owner` or `distributionAddress` or both altogether. Blockchain explorer applications will also show wrong token values and transfers. It is a critical bug and must be fixed.

##  7.4 Line 84

For extra safety the `transfer` function should add a `msg.data.length` check so the short address attack can be avoided. As discussed in low level vulnerabilities, this function should also add the modifier `validDestination` to prevent transfers to 0x0 address or the Pareto smart contract to save tokens from being stuck.

```
function transfer(address _to, uint256 _value) public  validDestination(_to)
  onlyPayloadSize(2) returns (bool success) {
  //…   
  }
```

##  7.5 Line 136

The `transferFrom` function should also add a `msg.data.length` check so the short address attack could be guarded against. As discussed in low level vulnerabilities, this function should also add the modifier `validDestination` just like `transfer` function to prevent transfers to 0x0 address or the Pareto smart contract to save tokens from being stuck.

```
function transferFrom(address _from, address _to, uint256 _value) public validDestination(_to)
  onlyPayloadSize(3) returns (bool success) {
  //…   
}
```

##  7.6 Line 158

The `approve` function should also add a `msg.data.length` check for the short address attack. The code to force user to set the approval to zero before setting a new nonzero value should be added as well.

```
function approve (address _spender, uint256 _value) onlyPayloadSize(2)
  returns (bool success) {
    if ((_value != 0) && (allowance_of[msg.sender][_spender] != 0)) return false;
      //…
    }
```

# 8. Final Recommendation

Our final recommendation would be to pay attention to the constructor of the contract where we are supposed to use `SafeMath` library when updating `balances` mapping and fire `Transfer` event when the token balance is transferred from `owner` to `distributionAddress`.

Apart from the minor vulnerabilities discussed in the audit, the contract is very secure and well written.
