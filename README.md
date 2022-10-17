# Debt DAO contest details
- $95,000 USDC main award pot
- $5,000 USDC gas optimization award pot
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2022-10-debtdao-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts October 20, 2022 20:00 UTC
- Ends October 27, 2022 20:00 UTC

---
# Background info
highly recommend reading our entire docs website https://docs.debtdao.finance/
These are the most relevant sections for Cod4rena wardens.
1. [Glossary](https://docs.debtdao.finance/glossary)
2. [Contract architecture Diagram](https://docs.debtdao.finance/developers/architecture)
3. [Known Exploits and Attack Vectors](https://docs.debtdao.finance/developers/edge-cases-and-risk-situations)
4. [Previous Audit by Halborn](https://docs.debtdao.finance/developers/security-audits/halborn)
5. [Code walkthrough by Debt DAO lead dev Kiba](https://www.loom.com/share/302e6981794f41429ad3f73db903033a) (slightly out of date but all concepts are the same just slightly different code)


# Novel Or Nonstandard Mechanisms 
1. We charge interest even if you dont have debt, we call this the `fRate` that a DAO pays to have access to immediate liquidity. It SHOULD be below the `dRate` charged when they borrower does have debt but does not explicitly have to be less.
2. Lender repayment queue. We use the `ids` array in Line of Credit contract to prioritize lenders that were drawn down on first, must be paid back first.
3. The Arbiter is a neutral third party that mediates conversations between all lenders and the borrower. They have priviliged access and are assumed to be honest at all times.
4. The Arbiter can declare a borrower INSOLVENT if there is no collateral left in Escrow or Spigot. This lets all lenders know that whatever balance they have deposited into the Line of Credit will never be repaid.


# Special Concerns For Audit
## Spigot 
Since our Spigot takes ownership of another protocol/DAO's smart contracts to secure their revneue streams (e.g. owning a yearn vault to ensure vault fees repays Yearn's debt) we want to make sure that the Spigot can't get bricked locking their contract forever, their contract cant be stolen from the Spigot, that revenue is securely escrowed inside the Spigot for the Owner to claim.


## SpigotedLine
Our integration between Line of Credit and Spigot contracts. It owns the Spigot so important that it doesnt lose ownership of Spigot unless `releaseSpigot()` successfully executes. That it can properly manage and call Spigot contract. It must be able to trade tokens on Spigot to a credited token using 0x protocol. 


# Smart Contracts

## LineOfCredit (478 sloc)
Core contract responsible for:

- Recording positions and accounting for the borrower and lenders
- Define line of credit terms (oracle, arbiter, borrower, term length, interest rate, escrow and spigot collateral)
- Coordinating Escrow, Spigot, and InterestRate modules 

- external calls to - Oracle, InterestRate
- Libraries - LineLib, CreditLib, CreditListLib


## SpigotedLine (247 sloc)
- Integration between Spigot and Line of Credit contract
- Manages Spigot config based on line health status
- Trades DAO revenue for tokens owed to lenders
- Stores excess revenue or trade slippage in `unused` tokens for later use in repayment
- allows borrower to clawback collateral if line is fully repaid
- allows liquidating `unused` or the Spigot itself if line is LIQUIDATABLE

- external calls to - 0x protocol, Spigot
- Libraries - LineLib, SpigotedLineLib, 


## EscrowedLine (60 sloc)
- Allows Arbiter to liquidate collateral if Line is LIQUIDATABLE
- Update Line status based on Escrow collateral ratio vs minimum collateral ratio

- external calls to - Escrow
- Libraries - LineLib 

## SecuredLine (98 sloc)
- Combines logic of Line of Credit, Escrowed Line, and Spigoted Line to create a fully secured lending solution.
- allows transferring all collateral to a new Line of Credit contract
- 

- external calls to - Oracle, Interest Rate,  Escrow, Spigot, 0x protocol
- Libraries - LineLib, SpigotedLineLib

---

# Escrow (114 sloc)
- Allows Borrower to deposit tokens as collateral for their Line of Credit
- Allows borrower to withdraw collateral if it does not make Line LIQUIDATABLE
- Allows Arbiter to whitelist specific collateral allowed for the Line of Credit
- Allows Arbiter to liquidate collateral if Line is LIQUIDATABLE

- external calls to - Oracle, Line Of Credit
- Libraries - LineLib, CreditLib, EscrowLib, 

# Spigot (190 sloc) 
- Takes full ownership of a DAO or protocols contracts to escrow revenue earned by them.
- Allows Owner to pull escrowed funds at anytime
- Allows arbitrary payment splits between Spigot Owner and DAO Treasury
- Allows updating stakeholder addresses


- external calls to - arbitrary contract with arbitrary calls
- Libraries - SpigotLib

# Oracle (24 sloc)
- A wrapper contract to simplify integration with Chainlink FeedRegistry
- Returns all token prices in USD 8-decimal denomination

- external calls to - Chainlink FeedRegistry
- Libraries - Chainlink Denominations

# Interest Rate (72 sloc)
- Stores interest rates and last time interest was accrued for individual lender positions on a Line of Credit
- Purely calculates interest owed, Line of Credit contract is responsible for updating balances
- only allows Line Of Credit contract to call it

- external calls to -
- Libraries -



# LineLib (79 sloc)
- Stores basic functinos for core Line of Credit e.g. health statuses, , and transfering tokens

# CreditLib (208 sloc) 
-  Stores basic functinos for interacting with lender positions e.g. computing position ids, accruing interest, and repaying debt

# CreditListLib (53 sloc) 
-  Stores functionality for interacting with collection of lender positions on Line of Credit e.g. adding and removing positions or resorting positions for repayment queue

# EscrowLib (183 sloc)
- calculates total value of assets escrowed using Oracle Price
- Calculates collateral ratio based on collateral value and outstanding debt value on Line of Credit
- ensures collateral enabled hasa price feed
- 

# MutualConsent (48 sloc)
- Forked from https://github.com/IndexCoop/index-coop-smart-contracts/blob/1acec44229b3aaf4a40dad2095b0cc6accb8fbfc/contracts/lib/MutualUpgrade.sol
- Essentially a 2/2 multisig baked into your contract
- Ensures that two predefined addresse both sign a tx with the same inputs and then executes the function with those parameters. 

# SpigotLib (212 sloc)
-  Stores functionality for claiming revenue, updating revenue splits, updateding stakeholder addresses, and all other Spigot functionality listed above
