# High
### [H-*] CREATE opcode does not work on zksync era

**Description:** zkSync Era chain has differences in the usage of the create opcode compared to the EVM. According to the protocol documentation is going to be deployed on zksync Era. 
The [zkSync Era docs](https://era.zksync.io/docs/reference/architecture/differences-with-ethereum.html#create-create2) explain how it differs from Ethereum. The description of CREATE and CREATE2 (zkSynce Era Docs) states that Create cannot be used for arbitrary code unknown to the compiler.

**Impact:**
No editions can be created in the zkSync Era chain.

**Recommended Mitigation:**
This can be solved by implementing CREATE2 directly and using type(Edition).creationCode as mentioned in the zkSync Era docs.




