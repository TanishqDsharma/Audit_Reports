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
    * A DOLA Fed mints DOLA to a market, which is then available to borrow for users holding DBR, using the Borrow function.
    * If the borrowers credit limts falls off  below the value of their outstanding debt, a percentage of their collateral may be liquidated on behalf of the protocol. This liquidation carries an additional fee which will be paid out to the liquidator, and may benefit protocol governance as well.
    * Markets do not hold any collateral, but instead deposits and withdraws collateral from Escrows unique to each user and market

# DBR.sol
  * The DBR token differs from standard ERC20 tokens in a few ways. It will actively be burnt from user's wallets, as they hold debt within the broader FiRM system, which can not just go to 0, but can drop into a deficit.
  * DBR tokens act as a dynamic asset tied directly to the user’s debt, with a burn mechanism ensuring continuous payment for debt.
    * The burn rate is directly linked to the debt, and updates to the user’s DBR balance only occur when their debt changes.
    * Forced replenishments ensure users maintain sufficient DBR tokens, even when their balance runs low, by minting new DBR tokens and increasing their DOLA debt.
    * The forced replenishment process is decentralized, allowing anyone to initiate it and earn a reward, making the system self-regulating.

# Oracle.sol
* The Oracle is a pessimistic oracle, logging the lowest price of the day, whenever it's getPrice() function sees a new low price. The Pessimistic Oracle introduces collateral factor into the pricing formula. It ensures that any given oracle price is dampened to prevent borrowers from borrowing more than the lowest recorded value of their collateral over the past 2 days. This has the advantage of making price manipulation attacks more difficult, as an attacker needs to log artificially high lows. It has the disadvantage of reducing borrow power of borrowers to a 2-day minimum value of their collateral, where the value must have been seen by the oracle.

# Fed.sol
* Feds are a class of contracts in the Inverse Finance ecosystem responsible for minting DOLA in a way that preserves the peg and can't be easily abused. In the FiRM protocol, the role of the Fed is to supply and remove DOLA to and from markets. The Fed contract is essentially the only lender in the protocol.

## Escrow Contracts (Deployed as minimal proxies)

1. SimpleERC20Escrow.sol (SLOCs: 22) Basic escrow implementation.
2. INVEscrow.sol (SLOCs: 56) Token escrow that deposits INV into xINV tokens, and allows the depositor to delegate voting power in the Inverse DAO. 
3. GovTokenEscrow.sol (SLOCs: 31) An example token escrow implementation that highlights the ability of escrow contracts to allow individual delegation of governance tokens used as collatera


