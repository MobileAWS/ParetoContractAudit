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


### 5.2.2 Double-spend on Approval

Imagine that Alice approves Mallory to spend 100 tokens. Later, Alice decides to approve Mallory to spend 150 tokens instead. If Mallory is monitoring pending transactions, then when she sees Alice’s new approval she can attempt to quickly spend 100 tokens, racing to get her transaction mined in before Alice’s new approval arrives. If her transaction beats Alice’s, then she can spend another 150 tokens after Alice’s transaction goes through.

This issue is a consequence of the ERC20 standard, which specifies that `approve()`takes a replacement value but no prior value. Preventing the attack while complying with ERC20 involves some compromise: users should set the approval to zero, make sure Mallory hasn’t snuck in a spend, then set the new value.

In general, this sort of attack is possible with functions which do not encode enough prior state; in this case, Alice’s baseline belief of Mallory’s outstanding spent token balance from the Mallory allowance.

It’s possible for `approve()` to enforce this behavior without API changes in the ERC20 specification:

                `if ((_value != 0) && (approved[msg.sender][_spender] != 0)) return false;`

However, this is just an attempt to modify user behavior. If the user does attempt to change from one non-zero value to another, then the doublespend can still happen, since the attacker will set the value to zero.

If desired, a nonstandard function can be added to minimize hassle for users. The issue can be fixed with minimal inconvenience by taking a change value rather than a replacement value:

`function increaseApproval (address _spender, uint256 _addedValue) onlyPayloadSize(2) returns (bool success) {
uint oldValue = approved[msg.sender][_spender];
approved[msg.sender][_spender] = safeAdd(oldValue, _addedValue);
return true;
}`

Even if this function is added, it’s important to keep the original for compatibility with the ERC20 specification.

The likely impact of this bug is low for most situations.

For more, see this discussion on github: https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729

### 5.2.3 Re-entrancy attack

The Ether receiving fallback function of a contract can be used to re-enter the sending contract’s function and perform malicious activity if the code of calling contract is not secure such as the calling contract does not update its state before making an external contract or raw call to another contract.

The Pareto contract does not use `call.value()` for external calls but uses `send()` in `reclaimEther()` function, which is safe as there is no risk of re-entrancy attacks since the `send()` function only forwards `2,300 gas` to the `fallback` function of called contract. This amount of gas can only be used to log an event’s data and throw a failure. This way you’re unable to recursively call the sender function again, thus avoiding the re-entrancy attack.



##  5.2 Low Vulnerabilities

### 5.3.1 Prevent token transfers to 0x0 address or the Pareto contract address

At the time of writing, the "zero" address (0x0000000000000000000000000000000000000000) holds tokens with a value of more than 80$ million. All of these tokens are burnt or lost. 

The transfer of Pareto Network tokens to the Pareto token contract would also result in the tokens being stuck.

An example of the potential for loss by leaving this open is the EOS token smart contract where more than 90,000 tokens are stuck at the contract address.

An example of implementing both the above recommendations would be to create the following modifier; validating that the "to" address is neither 0x0 nor the smart contract's own address:

`modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }`

The modifier should then be applied to the "transfer" and "transferFrom" methods:

   `function transfer(address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }
`

#  6 General Comments

##  6.1 Use of ‘approve’ Function

As mentioned earlier, the double spending problem resulting from the use `approve()` function, the Pareto token contract deals with it by introducing the functions `increaseApproval()` and `decreaseApproval()` for increasing and decreasing the approvals, respectively. Both of these functions eliminate the need to reassign allowance but the `approve()` function still does not protect against reassignment to a non-zero approved value if mistakenly called by the approver. The solution would be to force the user to first set the `allowance` value to zero before setting a non-zero new value.


 
             ` function approve (address _spender, uint256 _value)  returns (bool success) {

              if ((_value != 0) && (allowance_of[msg.sender][_spender] != 0)) return false;

              //...

              }
              `

##  6.2 Reclaiming Stranded Tokens

We recommend implementing the contract with the ability to call transfer on arbitrary ERC20 token contracts in case tokens are stranded there. 

As an example, the Pareto token contract has been sent Pareto (presumably accidentally). We generalize this to Peterson’s Law: “Tokens will be sent to the most inconvenient address possible.” 

The Pareto token contract does not have any mechanism to deal with stranded tokens.

##  6.3 Reclaiming Stranded Ether

It is very rare but possible to see Ether sent to a contract that is not payable through use of the `selfdestruct` function. For this reason we recommend allowing all contracts that are not holding Ether for other reasons to allow the owner to withdraw the funds. 

The Pareto token contract inherits a contract from OpenZepplin called `HasNoEther` which defines a constructor rejecting any Ethers sent with deployment of contract and has a function `reclaimEther` which sends the Ether balance of Pareto token contract to the owner address. Both reclaims, tokens and Ethers can be combined into a single function – An example of that is described below:

                         `function claimTokens(address _token) onlyOwner {
                              if (_token == 0x0) { 
                              owner.transfer(this.balance); 
                              return; 
                              }


                              Token token = Token(_token); 
                              uint balance = token.balanceOf(this);
                              token.transfer(owner, balance); 
                              logTokenTransfer(_token, owner, balance); 
                          }`


##  6.4 Absence of ‘emit’ with events

Firing events without the ‘emit’ keyword has been deprecated. The ‘emit’ keyword should be added preceding each event firing, so the contract complies to latest versions of Solidity.

Example:
      `emit Transfer(0x00, owner, totalSupply_);`
      
# 7 Line By Line Discussion



