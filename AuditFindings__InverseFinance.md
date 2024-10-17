# Overview of Contracts

### Points to note:

* FRIM is an overcollaterlized money market protocol for borrowing DOLA stablecoin at a fixed price, over an arbitrary period of time.
* ONE `DBR` token gives rights to borrow ONE `DOLA` token for one year,
* DBR are minted and offered to the market by Inverse Finance
* NOTE: he DOLA stablecoin is Inverse Finance's own stablecoin, which has it's peg actively managed by a system of Fed contracts, that enact expansionary or contractionary monetary policy to keep the peg of the stable coin near 1$.
* DOLA is the only borrowable asset in the FiRM protocol.

## Contracts:

# Market.sol
  * Central Contract of the FIRM Protocol
    * Most logic for borrowing and liquidations

# DBR.sol
  * The DBR token differs from standard ERC20 tokens in a few ways. It will actively be burnt from user's wallets, as they hold debt within the broader FiRM system, which can not just go to 0, but can drop into a deficit.
  * 
