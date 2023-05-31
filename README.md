![OWASP Smart Contract Logo](https://github.com/jinsonvarghese/test/blob/main/assets/images/OWASP%20Smart%20Contract.png)

# [OWASP Smart Contract Top 10](https://owasp.org/www-project-smart-contract-security-top-10/)

## About the Smart Contract Top 10

The OWASP Smart Contract Top 10 is a standard awareness document that intends to provide Web3 developers and security teams with insight into the top 10 vulnerabilities found in smart contracts. 

It will serve as a reference to ensure that smart contracts are secured against the top 10 weaknesses exploited/discovered over the last couple of years.

### Top 10

* SC01:2023 - [Reentrancy Attacks](2023/en/src/SC01-reentrancy-attacks.md)
* SC02:2023 - [Integer Overflow and Underflow](2023/en/src/SC02-integer-overflow-underflow.md)
* SC03:2023 - [Timestamp Dependence](2023/en/src/SC03-timestamp-dependence.md)
* SC04:2023 - [Access Control Vulnerabilities](2023/en/src/SC04-access-control-vulnerabilities.md)
* SC05:2023 - [Front-running Attacks](2023/en/src/SC05-front-running-attacks.md)
* SC06:2023 - [Denial of Service (DoS) Attacks](2023/en/src/SC06-denial-of-service-attacks.md)
* SC07:2023 - [Logic Errors](2023/en/src/SC07-logic-errors.md)
* SC08:2023 - [Insecure Randomness](2023/en/src/SC08-insecure-randomness.md)
* SC09:2023 - [Gas Limit Vulnerabilities](2023/en/src/SC09-gas-limit-vulnerabilities.md)
* SC10:2023 - [Unchecked External Calls](2023/en/src/SC10-unchecked-external-calls.md)

### Overview

| Title | Description |
| -- | -- |
| SC01 - Reentrancy Attacks | This is when an attacker is able to repeatedly call a function within a smart contract, exploiting the fact that the state of the contract hasn't been updated as expected. This could lead to funds or other resources being drained from the contract. |
| SC02 - Integer Overflow and Underflow | These vulnerabilities occur when a numerical operation results in a value that is outside the range of the variable's data type. In a smart contract, this could be exploited to manipulate balances or other critical values. |
| SC03 - Timestamp Dependence | If a smart contract's behavior relies on the timestamp of the block it's included in, it may be vulnerable to manipulation. This is because miners have a degree of control over the block timestamp. |
| SC04 - Access Control Vulnerabilities | If a smart contract doesn't properly implement access control, it can leave critical functions exposed. This could allow unauthorized users to perform actions that should be restricted, such as altering the contract's state or withdrawing funds. |
| SC05 - Front-running Attacks | Front-running is a vulnerability specific to blockchain systems. An attacker can observe a pending transaction and then issue their own transaction with a higher gas fee, incentivizing miners to include it in the blockchain first. |
| SC06 - Denial of Service (DoS) Attacks | DoS attacks aim to make a contract unresponsive or otherwise unavailable. In smart contracts, this could be achieved by consuming all available gas, or causing transactions to continually fail. |
| SC07 - Logic Errors | If a smart contract is poorly coded, it may contain logic errors that lead to unintended behavior. This could range from incorrect calculations to faulty conditional statements, or even exposed administrative functions. |
| SC08 - Insecure Randomness | Blockchain networks are deterministic by nature, making it difficult to generate true randomness in smart contracts. If an attacker can predict or influence a supposedly random number, they can manipulate the contract to their advantage. |
| SC09 - Gas Limit Vulnerabilities | Each Ethereum block has a gas limit, restricting the number of operations it can include. If a function within a contract requires more gas than this limit, it may become unexecutable, potentially freezing the contract or its funds. |
| SC10 - Unchecked External Calls | When a contract calls an external function, it may not properly check the result of the call. If the external call fails but the original contract doesn't check for this, it could assume the call was successful and continue its execution, leading to unintended consequences.

## Getting Involved
All discussions take place on the OWASP Smart Contract Security Top Ten [GitHub repository](https://github.com/OWASP/www-project-smart-contract-security-top-10). 

We welcome all community members to actively participate and help enhance this project. If you have any suggestions, feedback or want to help improve the list, we invite you to kickstart a dialogue by raising an issue or submitting a pull request.

You can read our contributing guidelines [here](CONTRIBUTING.md).

## Licensing
The OWASP Smart Contract Top 10 document is licensed under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/), the Creative Commons
Attribution-ShareAlike 4.0 license. Some rights reserved.

[![license](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by-nc-sa.svg)](https://github.com/OWASP/www-project-smart-contract-security-top-10/blob/master/License.md)

## Project Leaders
- [Jinson Varghese Behanan](mailto:jinson@owasp.org) (Twitter: [@JinsonCyberSec](https://twitter.com/JinsonCyberSec))
