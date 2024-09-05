# Bugs Found:

Informational: 
* Magic Numbers

LOW:
* Can enter the
* KillRavan can start early and end late
* Entrance Fee should be constatn
* if (block.timestamp > 1728691200) { //@audit Wrong time 

Medium:  
* mintRamNFT is vulnerable to reentrancy
* choosingRam selects any RAM candidate
* RAM can battle with himself
* killRavana should only be called by RAM instead on anyone

High:
* contract ChoosingRam has two instances of weak Randomness
* checking valid NFT on basis of token counter 
* withdraw is vulnerable to reentrancy
