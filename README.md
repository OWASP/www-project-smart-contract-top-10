![OWASP Smart Contract Logo](https://github.com/OWASP/www-project-smart-contract-top-10/blob/main/assets/images/OWASP%20Smart%20Contract.png)

# [OWASP Smart Contract Top 10](https://owasp.org/www-project-smart-contract-top-10/)

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
| SC01 - Reentrancy Attacks | A reentrancy attack exploits the vulnerability in smart contracts when a function makes an external call to another contract before updating its own state. This allows the external contract, possibly malicious, to reenter the original function and repeat certain actions, like withdrawals, using the same state. Through such attacks, an attacker can possibly drain all the funds from a contract. |
| SC02 - Integer Overflow and Underflow | The Ethereum Virtual Machine (EVM) defines fixed-size data types for integers, which limits the range of values they can represent. Overflow occurs when an arithmetic operation exceeds the maximum value a data type can hold, while underflow happens when an operation goes below the minimum value. For unsigned integers, underflow results in the maximum value, and for signed integers, exceeding the minimum value wraps around to the maximum positive value. |
| SC03 - Timestamp Dependence | Smart contracts often use block.timestamp for time-sensitive functions. However, miners can slightly adjust this timestamp, creating a vulnerability where they can manipulate the timing to gain an unfair advantage. |
| SC04 - Access Control Vulnerabilities | An access control vulnerability is a security flaw that allows unauthorized users to access or modify a contract's data or functions. These vulnerabilities occur when the contract's code fails to properly restrict access based on user permissions. |
| SC05 - Front-running Attacks | Front-running is an attack where a malicious actor exploits knowledge of pending transactions to gain an unfair advantage. Attackers observe the mempool and place their own transactions with higher gas fees to be processed before the target transaction, leading to potential financial losses and disruption of smart contract functionality. |
| SC06 - Denial of Service (DoS) Attacks | A Denial of Service (DoS) attack in Solidity targets vulnerabilities within smart contracts to exhaust critical resources such as gas, CPU cycles, or storage. These attacks aim to render the contract non-functional, disrupting its intended operation and potentially causing financial harm. |
| SC07 - Logic Errors | Logic errors, or business logic vulnerabilities, are subtle flaws found in smart contracts where the code deviates from its intended behavior. These errors can be challenging to detect as they reside within the contract's logic, potentially leading to unintended outcomes or exploitable conditions. |
| SC08 - Insecure Randomness | Generating true randomness in smart contracts on blockchain networks is challenging due to their deterministic nature. Predictability or influence over a supposedly random number can allow attackers to exploit contracts for their benefit, undermining fairness and security measures. |
| SC09 - Gas Limit Vulnerabilities | Gas limits on blockchain platforms like Ethereum impose constraints on smart contract computations per transaction. Functions exceeding the block gas limit, particularly those involving loops over dynamic data structures such as arrays, risk transaction failure due to resource exhaustion, highlighting a common vulnerability in contract design. |
| SC10 - Unchecked External Calls | In Ethereum smart contracts, failing to properly verify the outcome of external function calls can lead to unintended consequences. If the called function fails and the calling contract does not check for this, it may incorrectly proceed under the assumption of success, potentially compromising contract integrity and functionality.

## Getting Involved
All discussions take place on the OWASP Smart Contract Top Ten [GitHub repository](https://github.com/OWASP/www-project-smart-contract-top-10). 

We welcome all community members to actively participate and help enhance this project. If you have any suggestions, feedback or want to help improve the list, we invite you to kickstart a dialogue by raising an issue or submitting a pull request.

You can read our contributing guidelines [here](CONTRIBUTING.md).

## Licensing
The OWASP Smart Contract Top 10 document is licensed under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/), the Creative Commons
Attribution-ShareAlike 4.0 license. Some rights reserved.

[![license](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by-nc-sa.svg)](https://github.com/OWASP/www-project-smart-contract-security-top-10/blob/master/License.md)

## Project Leaders
- [Jinson Varghese Behanan](mailto:jinson@owasp.org) (Twitter: [@JinsonCyberSec](https://twitter.com/JinsonCyberSec))
- [Shashank](mailto:shashank@credshields.com) (Twitter: [@cyberboyIndia](https://x.com/cyberboyIndia))

