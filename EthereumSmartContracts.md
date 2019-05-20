## Ethereum smart contracts security recommendations and best practices:
1. In ERC20 approve first change to 0 and then to value
   1. tokens should verify in approve that previously was 0 if changing not to 0 like in minime and also add increaseAllowance/decreaseAllowance preferring it over approve
   2. https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit
   3. https://github.com/ethereum/EIPs/issues/738
   4. https://github.com/ethereum/EIPs/pull/610#issuecomment-304746812 was said to not be necessary: https://github.com/Giveth/minime/pull/18#issuecomment-337928059
   5. minime solution is shortest and best and here is an explanation of why it is enough: https://github.com/ethereum/EIPs/issues/20#issuecomment-277542427
   
2. Using things like block.coinbase, block.difficulty, block.gaslimit, block.number, block.timestamp, tx.gasprice, or tx.origin in constant functions is not a good idea, because it is unspecified what EVM will return, and different implementations, or even different versions of the same implementation, may behave differently
   1. https://github.com/ethereum/EIPs/pull/610#issuecomment-327514172
   
3. re-entrancy: transferring ether can trigger another contract which triggers back current contract causing money drain. this can be solved by Checks-Effects-Interactions pattern https://solidity.readthedocs.io/en/v0.5.8/security-considerations.html#re-entrancy
   1. this caused the DAO hack https://ethereum.stackexchange.com/questions/6210/how-was-the-recursion-created-that-lead-to-thedao-hack
   2. https://medium.com/spankchain/we-got-spanked-what-we-know-so-far-d5ed3a0f38fe
   3. the smart contracts execution is single-threaded (but the order between transactions is unknown until they settle with enough confirmations) which makes it possible to use state variables as mutexes, however this adds gas cost so other solutions might be preferable
      1. https://ethereum.stackexchange.com/questions/8261/how-to-solve-solidity-asynchronous-problem/8265#8265
      2. https://github.com/sigp/solidity-security-blog#preventative-techniques
	 4. https://github.com/ethereum/wiki/wiki/Safety#pitfalls-in-race-condition-solutions
   
4. always first do checks and then state changes and only then external contracts calls or transfers https://solidity.readthedocs.io/en/v0.5.8/security-considerations.html#use-the-checks-effects-interactions-pattern

5. using "transfer" can cause problems and it is better to allow withdrawals instead. ALWAYS use ERC20 approve+transferFrom instead of transfer, which will help protect from transfers to bad address and other errors, see: https://solidity.readthedocs.io/en/v0.5.8/common-patterns.html#withdrawal-pattern

6. never trust transaction origin as identification (origin is always the EOA who made initial transaction while msg.sender can be a contract which made internal transaction as a result of the original transaction) https://solidity.readthedocs.io/en/v0.5.8/security-considerations.html#tx-origin

7. use openZepplin safemath to protect from math overflows and underflows for any mathematic operations https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-101
   1. there is an EIP about evm opcodes protecting from overflows so use them when available https://eips.ethereum.org/EIPS/eip-1051
   
8. enable SMTChecker using appropriate pragma to allow formal verification https://solidity.readthedocs.io/en/v0.5.8/layout-of-source-files.html#smt-checker https://solidity.readthedocs.io/en/v0.5.8/security-considerations.html#formal-verification

9. dont add kill/self destruct or it can cause money loss: https://github.com/parity-contracts/0x863df6bfa4/pull/2 https://github.com/paritytech/parity-ethereum/issues/6995

10. https://medium.com/loom-network/how-to-secure-your-smart-contracts-6-solidity-vulnerabilities-and-how-to-avoid-them-part-2-730db0aa4834
    1. point 4 in the link: calling selfdestruct with a contract address, sends ether to that address without calling the contract fallback function so never base decisions on the balance. 
       1. another way is using coinbase transaction of mining reward
       2. another way is pre-fund an address before contract creation https://github.com/sigp/solidity-security-blog#pre-sent-ether
       3. another way is if the constructor is payable then to transfer funds on contract creation
    2. point 5 - always assume that transfers and external calls (of instance) can trigger revert
       1. calling method from contract instance propogates errors and allows to get return value while calling using .call(..) does not require knowing the abi, on failure only returns false without reverting and does not allow getting the true return value https://ethereum.stackexchange.com/questions/30383/difference-between-call-on-external-contract-address-function-and-creating-contr
       2. UPDATE: From solidity compiler version 0.5 the function .call DOES allow to get the return value, see: https://solidity.readthedocs.io/en/v0.5.8/050-breaking-changes.html#semantic-and-syntactic-changes
    3. ALWAYS avoid the use of now and block.blockhash for your contract’s business logic as their results are predictable or can be manipulated by miners https://www.reddit.com/r/ethereum/comments/483rr1/do_not_use_block_hash_as_source_of_randomness/
    4. more things covered already in other bullets in the recommendations list here https://medium.com/loom-network/how-to-secure-your-smart-contracts-6-solidity-vulnerabilities-and-how-to-avoid-them-part-1-c33048d4d17d
   
11. .call() and .call().value() should be avoided (.value(..) adds ether to send and .gas(..) adds gas limit). they both dont limit gas (without calling .gas(..)) and return true/false instead of reverting on failure and they may cause re-entrance vulnerability https://ethereum.stackexchange.com/questions/43782/importance-of-call-value/43821#43821

12. use cases of using send,transfer,.call().value(..).gas(..)()(). basically most cases use transfer and if using call then always set .gas(..) or otherwise the other contract can do a DOS by doing assert which takes away all gas preventing code from continueing execution after the call https://ethereum.stackexchange.com/questions/19341/address-send-vs-address-transfer-best-practice-usage/38642#38642 https://github.com/ethereum/solidity/issues/610

13. always prefer to throw and revert than to return true/false (more safe against reentrance attacks) as from discussion here https://github.com/ethereum/solidity/issues/610
    1. some good arguments about why better to return false instead of throw: https://github.com/ethereum/EIPs/pull/610#issuecomment-305770167
    2. some good arguments about why better to throw than return false: https://github.com/ethereum/EIPs/issues/20#issuecomment-300500880 https://github.com/ethereum/EIPs/issues/20#issuecomment-300746940

14. in general it is safer to trigger exception than to return false because exception reverts any changes to the state while return false leaves the responsability for the caller to revert if needed https://ethereum.stackexchange.com/questions/15140/would-it-be-better-to-use-throw-instead-of-return-false/15147#15147

15. new solidity compiler versions removed throw keyword and it now has assert,require,revert
    1. discussion on use cases of assert (in the end of 2018 open zepplelin changed safemath to use require instead of assert) https://github.com/OpenZeppelin/openzeppelin-solidity/issues/1120
    2. require and revert return the rest of the gas to the user while assert uses all of the gas recevied
    3. require is for input validations while assert is mainly for static code analysis to identify situations which should NEVER happen and create compile time warnings
       1. https://ethereum.stackexchange.com/questions/27812/why-using-assert-since-it-would-consume-all-gas/27824#27824
       2. https://ethereum.stackexchange.com/questions/15166/difference-between-require-and-assert-and-the-difference-between-revert-and-thro/50957#50957
    4. require vs revert seems to be different syntax for the same thing: https://github.com/ethereum/solidity/issues/6689

16. if contract does not need to receive ether (erc20 token without buying period for example) then dont implement payable functions and dont implement fallback functions. dont create unnecessary attack interface. more code = more attack interface. transfer to the contract will revert automatically: https://ethereum.stackexchange.com/questions/34160/why-do-we-use-revert-in-payable-function/34164#34164

17. still need a way to get ether out even if fallback function is not implemented in case where ether will still be received using selfdestruct or transfered when calling some methods

18. should make error messages as short as possible while still being clear because error messages increase contract deployment gas (while not increasing executing gas) https://ethereum.stackexchange.com/questions/66879/does-a-string-message-increase-the-gas-usage-of-a-require-statement

19. on division make sure the number is even or otherwise it will get rounded DOWN

20. be careful of replay attacks and of different assumptions https://media.consensys.net/discovering-signature-verification-bugs-in-ethereum-smart-contracts-424a494c6585
    1. for example previously a replay was possible between ethereum mainnet transaction and other chains, this was addressed in eip 155: https://eips.ethereum.org/EIPS/eip-155

21. Don’t write fancy code

22. Use audited and tested code https://security.stackexchange.com/questions/18197/why-shouldnt-we-roll-our-own

23. Write as many unit tests as possible https://consensys.github.io/smart-contract-best-practices/software_engineering/#contract-rollout

24. perform security audits and implement security layers architecture to protect from the unknown https://dasp.co/#item-10

25. Inline assembly is a way to access the Ethereum Virtual Machine at a low level. This bypasses several important safety features and checks of Solidity. You should only use it for tasks that need it, and only if you are confident with using it https://solidity.readthedocs.io/en/v0.5.8/assembly.html#inline-assembly

26. even if selfdestruct is not implemented, it is still possible using delegatecall so libraries should never call delegatecall/callcode

27. careful with delegatecall because it preserves context and allows the called contract to modify state of original contract. so the caller is at the merci of the called contract https://ethereum.stackexchange.com/questions/11578/how-to-protect-against-delegatecall-in-a-hub-spoke-contract-model/11598#11598

28. callcode also allows called contract to modify original contract storage just like delegatecall
    1. https://ethereum.stackexchange.com/questions/3667/difference-between-call-callcode-and-delegatecall
    2. to detect if your smart contract is called by callcode/delegatecall: https://ethereum.stackexchange.com/questions/69551/how-to-prevent-the-code-of-my-contract-being-used-in-a-callcode/69552

29. libraries called with delegatecall to modify original contract state, SHOULD NOT STORE STATE of themselves. because their state is actually pointing to storage slots of the caller contract which used delegatecall. use "library" keyword instead of "contract" keyword for libraries to detect problems at compile time (ensures the library contract is stateless and non-self-destructable) https://github.com/sigp/solidity-security-blog/blob/master/README.md#preventative-techniques-3

30. order of state variables matters! sort them from smallest to biggest type so that the storage allocated will be minimal which will require less gas to deploy
    1. mappings and dynamically-sized arrays don't follow normal storage rules
    2. when byte32 is converted to byte16 its taken from the left (0x922aba368bee9844aefc4b47b1d58d2857781b382dc1ad896d512e19131d108f -> 0x922aba368bee9844aefc4b47b1d58d28) https://gist.github.com/kenjirai/d1b9c135117eb39d7891d658cbd6154c
    3. when uint32 is converted to uint16 then the smallest bytes are taken 0x12345678 -> 0x5678

31. private members of contract are accessible (for example using web3.eth.getStorageAt) so don't store any private information in the contract. zksnarks and zero knowledge proofs and private key signature can allow proof of secret without revealing the secret
    1. getStorageAt returns 32 bytes content of desired storage slot which is a 32 bytes storage unit

32. constant variables of contract are not stored in storage

33. even if interface states that function is view/pure, the function implementation contract can define the function without view/pure and modify state. so can't assume state is not modified by interface declaration alone

34. when relevant to call something without letting it change state: STATICCALL opcode makes sure the called function does not modify state which new solidity compilers use for view/pure functions https://eips.ethereum.org/EIPS/eip-214

35. careful with EXTCODESIZE and CODESIZE because they behave differently during contract initialization: contract address is calculated from contract creation info and code initialization code is executed BEFORE the code is associated to the contract address so "During initialization code execution, EXTCODESIZE on the address should return zero, which is the length of the code of the account while CODESIZE should return the length of the initialization code" https://medium.com/coinmonks/ethernaut-lvl-14-gatekeeper-2-walkthrough-how-contracts-initialize-and-how-to-do-bitwise-ddac8ad4f0fd 
    1. "Put simply, if you try to check for a smart contract’s code size before or during contract construction, you will get an empty value. This is because the smart contract hasn’t been made yet, and thus cannot be self cognizant of its own code size"

36. contract which has problems at initialization will have an address but have no code

37. structs
    1. struct declarations default to storage so ALWAYS use "memory" keyword in initialization of temporary struct inside function to prevent it being saved to storage. also because of this better to avoid structs for temporary calculations
    2. if you declare a new storage struct in your function, it will overwrite other globally stored variables (starting from the first slot)
    3. when you directly save a memory struct into a state variable, the memory struct is automatically forced into storage
    4. function inputs are memory and not storage backed
    5. you cannot implicitly convert memory into storage in assignment of function input struct to storage variable struct
    6. although you cant return structs from functions, you can create mapping state variables with struct values and a default getter will be created for it which returns struct

38.when calling contract from remix better to always wrap big numbers with quotes to avoid problems and truncating

39. contract address is deterministically calculated from transaction so it is possible to recover lost funds if can send transaction of contract creation which creates address where funds are at (or to prefund a contract this way before its creation)
    1. https://github.com/sigp/solidity-security-blog#keyless-ether
    2. address is calculated from sender address and nonce. for contract the nonce starts from 1 (0 used for contract self creation): address(keccak256(0xd6, 0x94, YOUR_ADDR, 0x01)). next contracts address can be calculated by incrementing nonce in the formula
    3. https://medium.com/coinmonks/ethernaut-lvl-18-recovery-walkthrough-how-to-retrieve-lost-contract-addresses-in-2-ways-aba54ab167d3

40. if many contracts are deployed and want to optimize deployment gas by making contracts smaller then can deploy contract bytecode directly which can make contract opcode as small as for example 12 opcodes for initialization and 10 opcodes for runtime, see: https://medium.com/coinmonks/ethernaut-lvl-19-magicnumber-walkthrough-how-to-deploy-contracts-using-raw-assembly-opcodes-c50edb0f71a2

41. arrays
    1. arrays declarations default to storage so ALWAYS use "memory" keyword in initialization of temporary arrays inside function to prevent it being saved to storage
    2. dynamic arrays are ones defined with empty []
    3. Dynamic arrays must be initialized as "storage" variables
    4. You can resize dynamic storage arrays but You cannot resize memory arrays, nor fixed size arrays
    5. when you call solidity method you pass array size and it is not checked against actual payload. this allows to pass arrays of size bigger than possible to actually be generated and might allow other exploits so it needs to be taken into account when developing contracts (that array length is not bound by solidity)
    6. arbitrary array length change and index access in array should not be allowed because it can possibly allow to access any contract storage using the array indexer
    7. changing array length can be done by changing the array length property. need to be careful of underflows
    8. https://ylv.io/ethernaut-alien-codex-solution/
    9. https://f3real.github.io/Ethernaut_wargame2022.html

42. be aware that maps and arrays and any other variables, are not really allocated in an independent location, they are all together in the storage slots of the contract and are references to some positions in that storage (array and maps positions in the storage are calculated using a special formula), thus for example it is possible to access the same storage using a map and an array if the array is big enough

43. be aware that revert reverts everything including events except gas spent https://ethereum.stackexchange.com/questions/4085/is-it-a-good-practice-to-log-an-event-every-time-i-throw-in-solidity

44. https://blog.zeppelin.solutions/onward-with-ethereum-smart-contract-security-97a827e47702
    1. Avoid declaring variables using var if possible, otherwise the type will be the smallest possible (for i=1 the type will be uint8 maxing at 255) 
       1. UPDATE: from solidity compiler version 0.5 using var is disallowed https://solidity.readthedocs.io/en/v0.5.8/050-breaking-changes.html#variables
    2. the recommended naming for custom events is that they should start with "Log" to make them easily identified

45. add fail strings to require and revert commands for easier error finding and bugs detection

46. short address attack: if user address ends with one or more zeroes then by ommiting the zeroes if the sending system does not check the address length and create a transaction then the zero will be padded and will cause the transaction value to be added a zero causing undesired amount transfer
    1. https://dasp.co/#item-9
    2. mitigation using assert(msg.data.length == numOfParams * 32 + 4) creates bugs for multisig wallets working with the token
       1. https://github.com/Giveth/minime/issues/8#issuecomment-348762658
       2. https://vessenes.com/notice-about-fun-token-support-for-multisignature-wallets-2/
       3. funfair token paid for gas of transition to new contract https://funfair.io/new-fun-token-contract/
    3. mitigating by >= instead of == is also problematic https://blog.coinfabrik.com/smart-contract-short-address-attack-mitigation-failure/
    4. OpenZeppelin removed the short address attack validations from smart contracts because of the bugs they caused https://github.com/OpenZeppelin/openzeppelin-solidity/issues/261
    5. was fixed in solidity 0.5.0 so no need to perform anything in the smart contract however SHOULD perform address validations in the deposit/withdrawal interfaces in our system https://github.com/ethereum/solidity/pull/4224

47. must compile with latest solidity versions to get latest security fixes, for example the short address attack fix in solidity 0.5.0 https://github.com/ethereum/solidity/pull/4224
    1. https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-102

48. can't just do the hard work and security during contract development and then relax after deployment since solidity and evm upgrades many times enter new vulnerabilities to previously safe code, so need to always be up to date and ready to respond and to plan risk management for such cases. Example: https://medium.com/chainsecurity/constantinople-enables-new-reentrancy-attack-ace4088297d9
    1. another example where geth protocol change caused a bug and loss of funds https://steemit.com/cryptocurrency/@barrydutton/breaking-the-biggest-canadian-coin-exchange-quadrigacx-loses-67-000-usdeth-due-to-coding-error-funds-locked-in-an-executable
    2. example of place to periodically read is the solidity release notes to see bug fixes and behavior changes: https://github.com/ethereum/solidity/releases

49. don't base the smart contract code (passing .gas(2301) for example) on operations taking specific amount of gas such as transfer taking 2300 gas because those may change at any moment as discussed here: https://ethereum-magicians.org/t/remediations-for-eip-1283-reentrancy-bug/2434

50. CREATE2 opcode allows modifying contract code without modifying the address (redeploying to same address) which might be useful for some things but is considered a hack so the ability might be removed in future ethereum upgrades and can be used for attacks by redeploying malicious code in place of valid code
    1. resources
       1. https://medium.com/@jason.carver/defend-against-wild-magic-in-the-next-ethereum-upgrade-b008247839d2
       2. https://ethereum-magicians.org/t/potential-security-implications-of-create2-eip-1014/2614
       3. openzepplin zos is considering using it https://github.com/zeppelinos/zos/issues/152
       4. code creating redeployable metamorphic  contracts using CREATE2: https://github.com/0age/metamorphic
       5. https://blog.ricmoo.com/wisps-the-magical-world-of-create2-5c2177027604
       6. https://github.com/ricmoo/will-o-the-wisp
       7. https://github.com/Zoltu/deterministic-deployment-proxy
    2. ways to protect a smart contract from redeployable smart contracts (CREATE2)
       1. dont interact with destructible contracts and libraries which use selfdestruct or CALLCODE or DELEGATECALL
       2. validate that contract was created by EOA or by CREATE and the parent contract all the way up was created by EOA or by CREATE and not by CREATE2

51. SWC (Smart Contract Weakness Classification) collects known smart contracts weaknesses
    1. resources
       1. https://ethereum-magicians.org/t/eip-1470-smart-contract-weakness-classification-swc/1532
       2. https://github.com/SmartContractSecurity/SWC-registry
       3. https://github.com/ethereum/EIPs/issues/1469
       4. https://smartcontractsecurity.github.io/SWC-registry/
    2. several weaknesses which do not appear previously in this list
       1. functions and variables visibility should always be set appropriately to prevent unauthorized access https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-100
          1. All function names are in lower camelCase (eg. sendCoin) and all event names are in upper CamelCase (eg. CoinTransfer). Input variables are in underscore-prefixed lower camelCase (eg. _offerId), and output variables are _r for pure getter (ie. constant) functions, _success (always boolean) when denoting success or failure, and other values (eg. _maxValue) for methods that perform an action but need to return a value as an identifier. Addresses are referred to using _address when generic, and otherwise if a more specific description exists (eg. _from, _to) https://github.com/ethereum/wiki/wiki/Standardized_Contract_APIs
          2. UPDATE: From solidity compiler version 0.5 the compiler enforces explicit functions visibility declarations https://solidity.readthedocs.io/en/v0.5.8/050-breaking-changes.html#explicitness-requirements
       2. in non-library contracts, the solidity compiler pragma in the contract should be locked to a specific version in which the contract was tested to avoid vulnerabilities being introduced by compilation in a different compiler version (supporting multiple compiler versions is more appropriate for libraries used by other contracts) https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-103
          1. https://github.com/0xjac/ERC777/issues/61#issuecomment-479983564
       3. always validate .call return value https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-104 https://github.com/sigp/solidity-security-blog#9-unchecked-call-return-values-1
       4. never use assert unless asserting a bug which should NEVER happen. don't use it for logic reverts or for validations since it consumes all the gas that was sent to the transaction https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-110
       5. fix all warnings which can be fixed and don't use deprecated functions https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-111
       6. avoid race conditions and don't trust the miner https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-114 https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-116
          1. https://dasp.co/#item-7
          2. https://dasp.co/#item-8
          3. https://github.com/sigp/solidity-security-blog#10-race-conditions--front-running-1
          4. https://github.com/sigp/solidity-security-blog#12-block-timestamp-manipulation-1
       7. don't assume that signatures are unique and don't hash the signatures https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-117
       8. avoid shadowing variables https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-119
          1. All function names are in lower camelCase (eg. sendCoin) and all event names are in upper CamelCase (eg. CoinTransfer). Input variables are in underscore-prefixed lower camelCase (eg. _offerId), and output variables are _r for pure getter (ie. constant) functions, _success (always boolean) when denoting success or failure, and other values (eg. _maxValue) for methods that perform an action but need to return a value as an identifier. Addresses are referred to using _address when generic, and otherwise if a more specific description exists (eg. _from, _to) https://github.com/ethereum/wiki/wiki/Standardized_Contract_APIs
       9. don't trust blockchain randomness without oracles https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-1204
          1. https://dasp.co/#item-6
          2. https://www.reddit.com/r/ethereum/comments/74d3dc/smartbillions_lottery_contract_just_got_hacked/
       10. when dealing with signatures make sure to protect from replay attacks https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-121
       11. do not "bring your own crypto" https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-122 https://security.stackexchange.com/questions/18197/why-shouldnt-we-roll-our-own
       12. in multiple inheritance inherit from the more general to the more specific (the compiler will linearize the inheritance from right to left) https://consensys.github.io/smart-contract-best-practices/recommendations/#multiple-inheritance-caution https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-125
       13. be careful or even avoid using function pointers or letting user input set their value https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-127
       14. avoid dynamic length loops or actions because if they increase with time then the contract can become unusable gas-wise https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-128 https://github.com/sigp/solidity-security-blog#the-vulnerability-10
           1. dynamic length loop vulnerability and proposed solution in the following audit https://mainframe.com/Mainframe_Secondary_Audit.pdf
       15. check unicode characters of contract code to make sure they are all ascii before trusting what you see because right-to-left-override and similar unicode characters can mass up the code (can't trust etherscan verified code for example without this check) https://smartcontractsecurity.github.io/SWC-registry/docs/SWC-130
       16. Use interface type instead of the address for type safety (note that this will still not protect from malicious external implementation so never trust external calls and use validation-effects-calls pattern) https://consensys.github.io/smart-contract-best-practices/recommendations/#use-interface-type-instead-of-the-address-for-type-safety

52. when testing contract using metamask DONT FORGET to change metamask to test network or use metamask without real ether or otherwise can mistakenly use real ether and lose it

53. be aware that state variables are mutatable under enough confirmations (12+) happen for the modification transaction block https://ethereum.stackexchange.com/questions/8261/how-to-solve-solidity-asynchronous-problem/8265#8265
    1. so to make sure a previous function call was confirmed, need to save block height at previous call and compare it with block height at current call
    2. when transactions settle the uncle blocks do not modify state even though they give mining rewards to miners https://ethereum.stackexchange.com/questions/57727/are-uncle-blocks-unnecessary-overhead-on-blockchain

54. consider making tokens pausable to have mitigations in case of a hack such as happened for BEC token where the pause ability saved a lot of money: https://medium.com/secbit-media/a-disastrous-vulnerability-found-in-smart-contracts-of-beautychain-bec-dbf24ddbc30e https://nvd.nist.gov/vuln/detail/CVE-2018-10299
    1. MFT is another contract with vulnerability found which was saved by being pausable: https://blog.mainframe.com/important-mft-contract-redeployed-4f0b0bd8dc3b
    2. EOS, Tron, Icon, OmiseGo, Augur, Status, Aelf, Qash, and Maker tokens are pausable https://blog.cryptofin.io/what-we-learned-from-auditing-the-top-20-erc20-token-contracts-7526ef3b6fb1
    3. although discussing security tokens, the arguments in the following link about why to make the token pausable are partially valid for utility tokens too: https://github.com/ethereum/EIPs/issues/1400#issuecomment-420490082
    4. allows tokens to be freezed in order to move them in the future to a different blockchain if wanted/necessary which is an important ability
    5. considered an "emergency stop mechanism" https://github.com/OpenZeppelin/openzeppelin-solidity/issues/401#issuecomment-325387093
    6. regarding trust, one can say that users should NOT trust tokens which are NOT pausable because they can lose their money with such tokens more easily where pausable tokens can pause in emergency and the users balances are publicly known and it would be a criminal act for the token issuer to just "walk away" with the money for no reason after pausing the token
    7. the following reddit thread was opened at the time of the infamous DAO hack WHILE IT WAS HAPPENING and you can read the comments to see how the DAO users helplessly watched their money drained without any way to stop it. they would surely prefer a pause ability https://www.reddit.com/r/ethereum/comments/4oi2ta/i_think_thedao_is_getting_drained_right_now/
    8. unlike redeployable tokens which break the trust more than pausable token (because pausable tokens are part of something the user agreed to while redeployable tokens can introduce a whole new undesired contract code. and also pausable tokens dont modify balances while upgradable tokens can cause modifications), the pausing ability is even more important than upgrading ability for security reasons. that is because the pause serves as a response to attack while the upgrade can be looked at as a second step of restoration and it can also be done manually by deploying a fixed contract and updating the balances in it.
    9. it is written here to pause the contract in "prepare for failure": https://github.com/ethereum/wiki/wiki/Safety#general-philosophy

55. make sure that smart contracts you interact with don't use dynamic address for contracts they call because then those innocent contracts could be malicious https://github.com/sigp/solidity-security-blog#preventative-techniques-6
    1. honeypot used it:
       1. https://www.reddit.com/r/ethdev/comments/7x5rwr/tricked_by_a_honeypot_contract_or_beaten_by/
       2. https://www.reddit.com/r/ethdev/comments/7xu4vr/oh_dear_somebody_just_got_tricked_on_the_same/dubakau/

56. when testing a contract it is better to test it using a forked mainnet than testnet just in case of small critical differences. testing by forking mainnet can be done using genache: https://www.reddit.com/r/ethdev/comments/7x5rwr/tricked_by_a_honeypot_contract_or_beaten_by/du7i96z/

57. be careful of divisions and losing precision https://github.com/sigp/solidity-security-blog#15-floating-points-and-precision

58. if someone withdraws money from an EOA address it does not mean he has the private keys for that address (done by taking a random signature and calculating the address from it and using the signature to withdraw from that address)
    1. https://github.com/sigp/solidity-security-blog#one-time-addresses
    2. https://medium.com/@weka/how-to-send-ether-to-11-440-people-187e332566b7

59. total amount of ether is not an invariant depending on mining if selfdestruct is called with its own address to send ether to (this was possibly fixed by now). same if after suicide the contract kills another contract which sends fund to the original suicided contract https://swende.se/blog/Ethereum_quirks_and_vulns.html quick#2

60. checking if address is a contract by checking extcodesize doesn't work in execution of contract constructor. note that checking if original sender is a contract can be done by "msg.sender != tx.origin", see quirk4 here: https://swende.se/blog/Ethereum_quirks_and_vulns.html

61. make sure to use wrapper around ERC20 interface when interacting with ERC20 tokens from new solidity compiler to be able to interact with many tokens (including BNB and OMG) which implemented ERC20 without return value. can use SafeERC20 library of open zeppelin for this purpose
    1. https://medium.com/coinmonks/missing-return-value-bug-at-least-130-tokens-affected-d67bf08521ca
    2. https://docs.openzeppelin.org/docs/token_erc20_safeerc20
    3. some exchanges use other wrappers: https://github.com/ethereum/solidity/issues/4116#issuecomment-439824542

62. "running a sale that accepts multiple currencies at a fixed exchange rate is dangerous and bad" https://vitalik.ca/general/2017/06/09/sales.html

63. be careful about assumptions of user being in control of his account/keys because many accounts are in control of exchanges and not users https://github.com/ConsenSys/singulardtv-contracts/issues/59

64. should use checkInvariants modifier like here: https://github.com/ethereum/wiki/wiki/Safety#assert-guards

65. when deploying use solidity optimizations to lower gas

66. must not add a check in erc20 approve of amount being not bigger than user balance because that prevents infinite approval and approval in advance
    1. https://github.com/sec-bit/awesome-buggy-erc20-tokens/blob/master/ERC20_token_issue_list.md#a19-approve-with-balance-verify

67. when interacting with the contract sometimes it can be good to first do a call and only if success then to send transaction https://gist.github.com/ethers/2d8dfaaf7f7a2a9e4eaa

68. consider adding validation that transfer does not transfer to contract address, see validDestination modifier implementation here (or if not adding it then add a way for owner to withdraw contract tokens balance): https://consensys.github.io/smart-contract-best-practices/tokens/

69. consider adding a comment in code published to etherscan with information of who to contact for issues found in the contract https://consensys.github.io/smart-contract-best-practices/documentation_procedures/

70. Beware of negation of the most negative signed integer (Probably the solution should be to check for some invariant in the end of the calculation and not just -1) https://consensys.github.io/smart-contract-best-practices/recommendations/#beware-of-negation-of-the-most-negative-signed-integer

71. make sure that there are no shadows built ins https://consensys.github.io/smart-contract-best-practices/recommendations/#be-aware-that-built-ins-can-be-shadowed

72. pay attention that safemath only works with uint256, so be careful when working with other types without safemath

73. add speed bumps delays for sensitive actions which are rare enough or/and can be delayed (also make sure to add a way to pause/cancel in case of problem unlike DAO unstoppable speed bump which was useless), this will allow time to verify/detect/respond to attacks https://consensys.github.io/smart-contract-best-practices/software_engineering/#speed-bumps-delay-contract-actions

74. consider adding rate limiting where possible from business perspective https://consensys.github.io/smart-contract-best-practices/software_engineering/#rate-limiting

75. prefer more granural roles than just "owner". open zeppelin is doing it now:
    1. https://github.com/OpenZeppelin/openzeppelin-solidity/issues/1146
    2. https://github.com/OpenZeppelin/openzeppelin-solidity/pull/1274

76. make sure gas is limited in external calls to prevent attacks such as GasToken minting https://cryptopotato.com/ethereums-malicious-gastoken-minting-could-result-in-a-disaster-for-crypto-exchanges/

77. it is recommended to disable a contract instead of using self destruct if that contract is sent ether by users https://solidity.readthedocs.io/en/v0.5.8/introduction-to-smart-contracts.html#deactivate-and-self-destruct

78. to securely sign data offchain and pass it to a contract use eip 712 (by using web3.eth.personal.sign) and dont forget to add anti-replay nonce to the data signed and dont forget to add contract address to signature in case contract is redeployed to prevent replay to redeployed contract (cross contract replay attack)
    1. https://github.com/ethereum/EIPs/pull/712
    2. https://solidity.readthedocs.io/en/v0.5.8/solidity-by-example.html#what-to-sign

79. to make the contract clearer to users and avoid some misunderstandings, it is recommended to use natspec for any public/external functions/variables in the code and tools can use that to display it to end users
    1. https://solidity.readthedocs.io/en/v0.5.8/style-guide.html#natspec
    2. it needs to be published to swarm after compilation (otherwise one whitespace change can make the hash different)
    3. the hash of the json metadata is embedded inside the contract bytecode suffix https://solidity.readthedocs.io/en/v0.5.8/metadata.html#encoding-of-the-metadata-hash-in-the-bytecode
    4. https://solidity.readthedocs.io/en/v0.5.8/natspec-format.html
    5. truffle does not support natspec yet: https://github.com/trufflesuite/truffle/issues/1118
    6. metamask does not support natspec yet: https://github.com/MetaMask/metamask-extension/issues/2501
    7. radspec is an implementation of natspec referenced by solidity documentation https://github.com/aragon/radspec#aside-why-is-natspec-unsafe
    8. here it says it is hard to find the swarm where the json was deployed: https://blog.zeppelin.solutions/deconstructing-a-solidity-contract-part-vi-the-swarm-hash-70f069e22aef
    9. Supporting natspec for state variables is stale for 1.5 years: https://github.com/ethereum/solidity/issues/3418
    10. Here it says that radspec and natspec are not compatible: https://github.com/aragon/radspec/issues/67
    11. seems natspec is not well supported right now but may be better supported in the future

80. dont trust hash of msg.data because some unused bits can be used to change the hash of msg.data https://solidity.readthedocs.io/en/v0.5.8/security-considerations.html#minor-details

81. enable SMTChecker when it will go to production https://solidity.readthedocs.io/en/v0.5.8/layout-of-source-files.html#smt-checker

82. Take care if you perform calendar calculations using these units, because not every year equals 365 days and not even every day has 24 hours because of leap seconds. Due to the fact that leap seconds cannot be predicted, an exact calendar library has to be updated by an external oracle https://en.wikipedia.org/wiki/Leap_second

83. careful when using blockhash. The block hashes are not available for all blocks for scalability reasons. You can only access the hashes of the most recent 256 blocks, all other values will be zero

84. be aware that packed abi encoding can be ambigious and hash collisions are possible for it https://solidity.readthedocs.io/en/v0.5.8/units-and-global-variables.html#abi-encoding-and-decoding-functions

85. If you use ecrecover, be aware that a valid signature can be turned into a different valid signature without requiring knowledge of the corresponding private key https://solidity.readthedocs.io/en/v0.5.8/units-and-global-variables.html#mathematical-and-cryptographic-functions

86. ethereum blockchain implements some functions (sha256, ripemd160 or ecrecover) as "precompiled" contracts which are deployed after their first use and thus first usage takes more gas and might run out of gas. relevant only for private blockchains https://solidity.readthedocs.io/en/v0.5.8/units-and-global-variables.html#mathematical-and-cryptographic-functions

87. note that if C is a contract then type(C).creationCode and type(C).runtimeCode have a lot of gotchas so be careful using them https://solidity.readthedocs.io/en/v0.5.8/units-and-global-variables.html#type-information

88. You should avoid excessive recursion, as every function call uses up at least one stack slot and there are at most 1024 slots available. for external calls (external calls generate message call while internal non prefixed calls are translated into jump opcodes) only 63/64th of the gas can be forwarded in a message call, which causes a depth limit of a little less than 1000 in practice

89. The low-level functions call, delegatecall and staticcall return true as their first return value if the called account is non-existent, as part of the design of EVM. Existence must be checked prior to calling if desired https://solidity.readthedocs.io/en/v0.5.8/control-structures.html#error-handling-assert-require-revert-and-exceptions

90. When dealing with function arguments or memory values, there is no inherent benefit to use types less than 256 bits because the compiler does not pack these values https://solidity.readthedocs.io/en/v0.5.8/miscellaneous.html#layout-of-state-variables-in-storage

91. use "indexed" keyword for events parameters which you wish to be indexed as topics. up to 4 indexed parameters are allowed per event. to have both the values and fast search, can add same value both as indexed parameter and as a not indexed parameter https://solidity.readthedocs.io/en/v0.5.8/abi-spec.html#events

92. It is highly suspicious if a contract was compiled with a compiler that contains a known bug and the contract was created at a time where a newer compiler version containing a fix was already released (note that most bugs are not critical but should periodically check the bugs list anyway, especially for new versions in the bottom here https://github.com/ethereum/solidity/blob/develop/docs/bugs_by_version.json) https://solidity.readthedocs.io/en/v0.5.8/bugs.html

93. The operators || and && apply the common short-circuiting rules. This means that in the expression f(x) || g(y), if f(x) evaluates to true, g(y) will not be evaluated even if it may have side-effects

94. The type byte[] is an array of bytes, but due to padding rules, it wastes 31 bytes of space for each element (except in storage). It is better to use the bytes type instead. As a general rule, use bytes for arbitrary-length raw byte data and string for arbitrary-length string (UTF-8) data. If you can limit the length to a certain number of bytes, always use one of the value types bytes1 to bytes32 because they are much cheaper https://solidity.readthedocs.io/en/v0.5.8/types.html#fixed-size-byte-arrays https://solidity.readthedocs.io/en/v0.5.8/types.html#bytes-and-strings-as-arrays

95. make sure that hardcoded literal addresses pass the checksum test or otherwise they are treated as regular rational numbers https://solidity.readthedocs.io/en/v0.5.8/types.html#address-literals

96. Underscores should be used to separate the digits of a numeric literal to aid readability and prevent mistakes, they do not influence the number https://solidity.readthedocs.io/en/v0.5.8/types.html#rational-and-integer-literals

97. pay attention that "string" type is UTF8 encoded so by "bytes(s)[7] = 'x';" Keep in mind that you are accessing the low-level bytes of the UTF-8 representation, and not the individual characters https://solidity.readthedocs.io/en/v0.5.8/types.html#bytes-and-strings-as-arrays

98. note that delete has no effect on mappings https://solidity.readthedocs.io/en/v0.5.8/types.html#delete

99. explicit overflowing conversions of negative integers is possible so be careful https://solidity.readthedocs.io/en/v0.5.8/types.html#explicit-conversions

100. always re-check the event data and do not rely on the search result based on the indexed parameters alone because the encoding of a struct is ambiguous if it contains more than one dynamically-sized array https://solidity.readthedocs.io/en/v0.5.8/abi-spec.html#encoding-of-indexed-event-parameters

101. readable and consistent code will lead to easier bug detection during development and eventually lead to a more secure code so follow the conventions here: https://solidity.readthedocs.io/en/v0.5.8/style-guide.html

102. library view functions do not have run-time checks that prevent state modifications https://solidity.readthedocs.io/en/v0.5.8/contracts.html#view-functions

103. Pure state reading are not enforced by evm https://solidity.readthedocs.io/en/v0.5.8/contracts.html#pure-functions

104. anonymous keyword for events makes the event name not be stored which makes the contract gas cheaper to deploy and call the event cheaper too. should be used when contract has only one event where the event name is not enforced by some standard because then all logs are known to be from this event and no need to store the event name https://github.com/ethereum/solidity/pull/6791#issuecomment-493989944

105. for tokens contract (and some other sensitive contracts too), in addition to being pausable, there should be an upgrading/migration strategy as described here: https://github.com/ethereum/EIPs/issues/644#issuecomment-494106553
