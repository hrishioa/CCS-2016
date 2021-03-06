### CCS 2016 - Oyente 

----

Cryptocurrencies were first introduced to the world in 2009, through a digital currency that we know as Bitcoin. They operate on a decentralized network model, where transactions are performed and verified by nodes on a network, with no centralized/trusted parties. Since then, they have grown in number and volume, with the two most popular currencies - Bitcoin and Ethereum - currently operating at a market capitalization of 10 billion and 1 billion USD respectively. Originally introduced for peer-to-peer payments, they were soon explored as an avenue for automated payments - and most recently, for use in Smart Contracts.

----

A Smart Contract is a computer program that encodes the rules and conditions of a contract in such a way that the contract can be executed and verified in an automated fashion. Originally introduced by Nick Szabo in a 1994 paper, they have seen adoption in both private and public settings.

The major introduction for smart contracts to the world came from Ethereum - a cryptocoin focusing primarily on executing turing-complete smart contracts on the blockchain. After one of the top 10 most most-valued crowdfuding campaigns of all time in 2014, Ethereum currently has over 17000 smart contracts.

Smart Contracts have been proposed for a multitude of applications, ranging from simple micropayment systems and automated loans to entire decentralized organisations. Here is a snippet of a popular smart contract on Ethereum:

The contract is written in Solidity, a javascript-like language, and this snippet here implements a withdrawal function for investors in this particular contract. As we can see, there is a global balance being updated as well as an array of local balances, and a payout being sent in Ether - the currency in Ethereum - to the person making the call. Deceptively simple to understand and implement, this piece of code is then compiled into a stack-based bytecode that is executed and verified by the nodes on the Ethereum network.



Unlike traditional systems, smart contracts are open to attack when operating on a public - permissionless - network, where anonymous adversaries may attempt to exploit their operation. Smart Contracts like this one can come to hold and transact large amounts of capital in their operating life, providing ample financial incentive. This is not accounting for the possibility of bugs in contract code that can lock away any amount of this capital forever irreversibly and irretrievably. In fact, this snippet of code belongs to a popular contract called TheDAO, a decentralized venture capital fund that was launched in May this year with a corwdfunding campaign that turned out to be the largest in history, amassing over a hundred and sixty million USD in capital from over 11000 investors, nearly 14% of all Ether tokens issued. Early June this year, a recursive splitting bug - which we will analyse in detail later -  was exploited that allowed attackers to drain over a third of this money - $50 million USD at the time - from the contract.

Moreover, smart contracts are immutable, which means that patches cannot be applied easily or sometimes at all. In fact, the vulnerability that led to the attack we've just seen was known and published some time before it was exploited.

*Talk about upcoming ethereum hard fork due to GAS manipulation?*

In this presentation, we will be introducing several new classes of security bugs in Smart Contracts, analysing them on the public blockchain, and how automated detection and improved design can help counter this threat. If we can understand the fundamental flaws in thinking that leads to these bugs, we can arrive at a better model for placing smart contracts on decentralized networks.

----



We introduce Oyente, a smart contract analyer that is capable of detecting the bugs being discussed today, with a false positive rate less than 7%. The code has been open sourced, and is available for easy download and evaluation at this link.

We also introduce Etherlite, an improved version of Ethereum that patches these issues without needing to modify existing contract code.

---

Why are these problems prevalent?

We believe that these flaws arise in practice due to a sematic gap - a mismatch in the rational expectations made by the architects of smart contracts and the reality of executing code on a decentralized network.

Moreover, the amount of abstraction provided by the high-level languages these contracts are written in - javascript and python equivalents, for Ethereum - lead to a false sense of security, where underlying operations and security flaws are obfuscated and often poorly understood.

----

Let us consider an example to illustrate the danger of abstracton.

Solidity provides users access to a timestamp from within the contract through the `now` variable. A sample contract that stores a particular timeout of 4 weeks is shown. The syntax is rather easy to read and implement. The `now` variable is an alias for `block.timestamp`, which represents the timestamp of the current block. This has led to contracts such as theRun, which implements a payout controlled by a psuedo random number generator, the code for which is shown.

The function `random` uses the block timestamp as a salt, which is not uncommon in a traditional program. However, there is no global timestamp in the decentralized network, and the rules governing the assignment of a block timestamp only serve to provide an approximation.

This leaves the timestamp - and by extension, any contracts using it - open to manipulation by the miners. In reality, the timestamp can be changed by as much as 900 seconds by a miner, which allows him/her to manipulate the output of the contract to an advantage.

This introduces what we've come to call timestamp dependence, which is one of the classes of bugs detected by Oyente.

---

Next, let us look at an example of how a semantic gap can be problematic.

The contract `MarketPlace` implements a decentralized marketplace. An owner can deploy the contract to sell stocks in an asset, which performs two primary functions - A user can buy any number of available units of stock, and the owner can update the unit price at any time.

In a centralized execution environment such as a web server, there is a a rational expectation that calls are atomic, which is commonly the case when they are completed and served on the order of milliseconds.

On Ethereum, each contract call is a transaction that updates the state of the contract, and many transactions are bundled together to form a block that enforces a state transition on the blockchain. Let us consider how two such transactions would occur - 

1. User A sends a call T1 to buy a 100 units of stock at $1, which is the current value as per the last block.
2. User B - the owner - send a call to UpdatePrice to update the current price to 0.01 cents, with or without the knowledge of A's transaction.
3. A block is compiled by a miner once a sufficient numer of transactions have been posted, which includes both A and B's transactions. Once the block is mined, their transactions can be considered processed, and the ownership of stock is transferred and the current price updated.

We can see how this is problematic. Even if B's transaction was posted after A's, there is a non-zero probability - even in a fair network - that it will be processed first. This means that a malicious owner can update the price to his advantage **after** observing a buy order.

It is understandable that a developer unfamiliar with the dynamics of contract execution misses this vulnerability, which we've come to call **Transaction Ordering Dependence (TOD)**.

---

Another class of bugs have been previously found, that exploits the way exceptions are handled in Smart Contracts on Ethereum.

On the Ethereum network, contracts are allowed to execute in up to a maximum callstack depth of 1024. Attempting to execute a function or contract call at this depth results in an exception. However, this exception is not propagated up the callstack as a default behavior. A contract can ignore the exception, which leads to unexpected and often unfortunate behavior.

Here's an example - this contract implements a government where a user can pay the old monarch to replace him as the current monarch. A monarch can only be displaced once he is compensated for his investment. However, if the user were to call the contract with a carefully constructed callstack of 1023, the payment for the current monarch would not be executed, and the contract would continue to dethrone him.

This is an instance of a vulnerability in the platform that is not apparent to a developer working on the contract, leading to unexpected results.

---

Finally, let us consider the vulnerability popularized by TheDAO contract we briefly covered at the beginning. This has been the single largest instance of a smart contract bug - in terms of monetary loss - with the amount lost from TheDAO being larger than all the disovered possible losses of all the other bugs *combined*. 

Let us see how it works. Consider the snippet of code we saw earlier. This particular piece of code is quite self-explanatory. A transfer event is registered to the blockchain, after which a reward is withdrawn andthe contracts internal balances are updated. However, the issue is in the `payOut` function, which is called by `withdrawReward`, the offending snippet of which is shown here:

The recipient contract is called with the Ether value (and no limit on the amount of gas), which is at the root of this vulnerability. When a call is made, the recipient contract is executed, and control does not return to the calling contract until is it complete.

Let's consider what happens here. When a call is made, the contract send the payment to a receiving contract, which then performs some function. After this, control returns to the calling contract, and the balances are updated. However, if the contract were malicious, it could use gas from the caller to re-execute the original call and double the payout - this will be valid since the balances have not been updated yet. In fact, this can repeated as many times as needed until the gas is exhausted.

This is the attack that crippled theDAO and enabled the attacker to siphon millions out of the contract. It tells us that large decentralised contracts such as theDAO are even more vulnerable to the semantic gaps and lack of understanding, further emphasizing the need for an automated pre-deployment testing.

---

As part of this work, we introduce Oyente, a smart contract analyzer

---

The Basic architecture of the tool is built around Symbolic Execution. We will explore this in detail on the next slide.

The Architecture is highly modular so as to be extensible. The tool is structured as a number of modules which are as follows:

1. Contract bytecode is fed to the CFG builder, which builds a skeletal control flow graph of all basic blocks and jumps. The CFG is constructed statically and updated dynamically later on.
2. The explorer uses the entrypoint from the CFG Builder and the current block state to symbolically evaluate the instructions sequentially from the entrypoint. The explorer queries Z3 as to the feasibility of branches and explores all feasible and reachable instructions.
3. The Core Analysis module checks for the existence of bugs.  
4. The Validator then attempts a check for false positives. This is far from complete - currently the module checks Z3 to enture that the TOD bugs found by the Core Analysis module are present only in *feasible* paths.
5. The Visualizer provides a graphical output for the Control Flow Grap that can aid in manual debugging.



The modular design of the tool allows for easy addition of extension modules to increase test coverage. For example, one proposed addition is a module that can estimate worst-case gas consumption for contracts.



---

Oyente relies on Symbolic Execution using Z3 to perform the path checks that enable bug detection.

Symbolic execution represent the values of program variables as symbolic expressions of input values, which are then used to statically reason about the feasibility of the paths in a program. This is often faster and more comprehensive to implement tests for than dynamic testing, where a program is tested input-for-input.

For example, consider a single path in a program. The path conditions for this particular path are calculated by evaluating the symbolic values of the variables in this path. Once the execution reaches the end of the path, the path conditions are evaluated to determine whether the path is feasible. Here, the path under analysis is feasible since there is a valid value for x that allows execution.

Symbolic execution also makes it easier to reason about validity of contracts, the analysis of which often involves comparing interleaving paths and the exploitation of intermediate states.

---

Oyente implements automated detection for all the bugs discussed.

For Transaction Ordering Dependence, the different paths are analysed for possibly concurrent Ether flows. If two separate paths have flows of Ether, Oyente reports it as a TOD Contract.

For Timestamp dependence, a special symbolic variable is associated with the timestamp. If there are any feasible paths affected by this variable, it is reprted as timestamp dependent.

Detecting a mishandled exception is done by bytecode analysis. A CALL instruction - which corresponds to a function or contract call, places the return value at the top of the stack. If there is no ISZERO instruction which checks for a valid value, the contract is reported to be have a mishandled exception.

Reentrancy Detection is more complicated. Upon encountering a CALL instruction - a possible point of vulnerability - we obtain the path condition for executing the instruction. We then check if the CALL can be executed again through another call to the contract. If so, this is flagged as a possible vulnerability.

---

Etherlite is a new set of semantics proposed for Ethereum that aims to remediate these bugs.

A few things were taken into account when developing Etherlite - 

1. Maximum Conformance to existing system - Etherlite is designed as a proposed upgrade to Ethereum. Therefore, it needs to suggest a change that can be practical to implement by updating the miners.
2. Backward Compatibility - Etherlite was designed so that existing contracts can still be executed, while removing the vulnerabilities that were found.

Here are the improvements proposed by Etherlite:

1. Guard Conditions - The rules for transaction execution have been modified to accept a guard condition, such that the transaction is not processed if the guard condition is not satisfied. The guard conditions can pertain to the state of the contract being executed, such that an expected result is either achieved or none at all. For transactions that do not specify a guard condition, this is set to `true`, so that the condition is always satisfied.
2. Deterministic Timestamp - The timestamp variable has been replaced with the block index to model the global time. Since the average block time in Ethereum adjusts to 12 seconds per block, this can provide a representation of time without obfuscating the true granularity of the timestamp thus provided. for contract logic, we can see how this would replace the timestamp. A timeout of 24 hours can be replaced by an equivalent measurement of the block index. 
3. Better Exception Handling - The exception handling characteristics of the ethereum virtual machine have been updated such that exceptions always halt execution, and are propagated up at the callstack until handled. If an exception is fatal, the transaction is rolled back. This would prevent attacks that take advantage ofmishandled exceptions.
4. Lower default GAS for CALL - The reentrancy attack can be remedied retroactively by restricting the amount of GAS supplied by default to a CALL or a SEND. Currently a CALL instruction is allotted the entirety of the remaining GAS amount for execution. When this is restricted, the possibility of recursively calling a contract in its intermediate state is limited.

----

the experimental Setup is as follows <Description as per slide>

----

Contract complexity was one of the key factors that led to the investigation regarding transaction ordering and exception mishandling. We found that most contracts nest deeper than 1 level in the stack, but alll contract invocations never exceed a depth of 50 in *benign* runs. This explains the lack of exception handling code in most contracts, as well as the need for one.

We also discovered that contracts have about 2 thousand instructions on average, with a median of 838. To handle all the contracts, Oyente would need to handle about 63 instructions, which is what was aimed for in development.

---

Here we can see the results of the benchmark. Oyente flags 8833 (or slightly more than 45%) of all the contracts as having at least one security issue, of which 1,682 are distinct by bytecode analysis. Exception mishandling is the most common, with 5,411 contracts being affected. Oyente operates on the bytecode of ethereum contracts. While source code analysis provides more insight into the purpose of a contract, only a negligible percentage of Ethereum contracts (175 out of the 8,833) have source code available for analysis. This is because source code tagging is largely a manual process. 

False positive evaluation was done manually using available source code, and Oyente has a false positive rate of 6.4%, or 10 cases out of 175. The small number of source code samples highlights a need for automated false positive confirmation, which is one of the goals of the open-source project moving forward.

---

Here are some of the interesting attack discovered by Oyente.

The Ponzigovernmental Contract (The capitalization is not a typo), held over 11000 USD at the time of writing. Oyente highlighted multiple bug in this contract. The contract implements a lottery where a jackpot is awarded after a timeout to the last investor, with a small percentage being sent to the owners. The first bug pertains to the timestamp. The jackpot is released when a certain time has elapsed, and it is possible for a colluding miner to control when the timeout is invalidated and hence the payout.

Perhaps the graver bug is that of the possibility of stealing Ether from the contract. The contract by design pays the investors, decrements their stakes, and credits the remaining Ether to the owner. While this look innocuous, it is possible for a malicious owner to first execute the contract with a preconstructed callstack such that all transfers of Ether fail, at which point the Ether stays in the contract and the balances update to 0, and make a second call so that all of Ether is sent to the owner.



The second example is that of EtherID, which works as a name registrar for the Ethereum network to allow users to buy and sell IDs. The contract has proven quite popular with over 57 thousand transactions so far. As we have seen from the callstack attack, the send instruction that pays the last owner of an ID for a sale can be made to fail, in which case the Ether is locked in the contract forever.

---

<This slide is hidden, not sure if necessary>

In addition to this, there are a number of miscellanous errors that we felt needed highlighting: 

These are errors in logic when writing a contract, and need to be kept in mind.

The first is the lack of refundable deposits for many contracts. A number of contracts require a deposit of 1 Ether to continue execution, to doscourage spurious calls. This deposit is usually refunded at the end. however, we discovered a number of contracts that do not refund any deposit that does not match 1 Ether exactly, locking the money into the contract.

---

From this analysis, we've learned that there needs to be an improvement or a paradigm shift in the way that smart contracts are current developed. The number of logic and semantic gap points to a general lack of understanding of blockchain mehanics among developers, and the high level languages that obfuscate some of the inner workings are partly to blame. In addition, the underlying platform needs to improved in response to discovered bugs with security in mind. Finally, contract developers need to employ cryptography when storing sensitive user information on the blockchain, as an alarming number of contracts currently store information that could be exploited by a malicious adversary. Finally, there is an absolute need for pre-deployment analysis, to reduce any and all errors after a contract is deployed. As mentioned before, the nature of the blockchain makes patching contracts arduous at best.

---

Finally, the tool and and accompaying benchmarks can be found here. There is an automated docker pipeline that makes set-up easier, which can be run from the following command. Etherlite is described in detail in the paper.



Thank You!

* ## Introduction

* ## Why You Should Care

  * ### Ethereum Numbers

  * ### TheDAO Attack

  * ### Code is Law

  * ### Malicious interference and accidental loss of funds

  * ### Need to ensure and enfore correctness *before* deployment

* ## Smart Contracts on the Blockchain

  * ### Stack-based language

  * ### Consensus

  * ### Gas

  * ### Implementations - Solidity and Serpent

* ## Ethereum and Turing Complete Smart Contracts

* ## Contributions

  * ### Oyente - A Smart contract Analyser

  * ### Improved Semantics for Turing complete Smart Contracts on Blockchain

    * #### Improving security without rewriting existing contracts

* ## Problems - Basic Overview

  * ### Why Problems occur in Smart Contracts

    * #### Imperfect understanding of Contract Execution

    * #### Obfuscation of underlying mechanisms in programs

    * #### Issues with platform implementation

* ## Transaction Ordering Dependence

* ## Timestamp Dependence

* ## Callstack Attack

* ## Reentrancy Bug

* ## Solution - Detection

  * ### Symbolic Analysis with Z3

  * ### Pattern Matching for Faster Analysis

* ## Transaction Ordering Dependence

  * ### Number of Contracts found

  * ### Examples and Z3 Results

  * ​