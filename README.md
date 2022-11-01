
# Debt DAO contest details
- Total Prize Pool: $115,500 USDC
  - HM awards: $85,000 USDC
  - QA report awards: $10,000 USDC
  - Gas report awards: $5,000 USDC
  - Judge + presort awards: $15,000 USDC
  - Scout awards: $500 USDC
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2022-11-debt-dao-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts November 3, 2022 20:00 UTC
- Ends November 10, 2022 20:00 UTC


# Contest setup
Go to our [project repo](https://github.com/debtdao/Line-of-Credit/tree/2254b3e8a36110481a03e31e96874bd99538b4f6) and follow the instructions there  (commit hash: `e8aa08b44f6132a5ed901f8daa231700c5afeb3a`)

## Contact Info
Cod4rena discord channel for contest 
https://discord.gg/bD4nnRwYPN

---
# Background info
We highly recommend reading our entire docs website https://docs.debtdao.finance/

These are the most relevant sections for Code4rena wardens.
1. [Glossary](https://docs.debtdao.finance/glossary)
2. [FAQ](https://docs.debtdao.finance/faq)
3. [Contract Architecture Diagram](https://docs.debtdao.finance/developers/architecture)
4. [Functions and Methods](https://docs.debtdao.finance/developers/functions-and-methods)
5. [Known Exploits and Attack Vectors](https://docs.debtdao.finance/developers/edge-cases-and-risk-situations)
6. [Previous Audit by Halborn](https://docs.debtdao.finance/developers/security-audits/halborn)
7. [Code walkthrough by Debt DAO lead dev Kiba](https://www.loom.com/share/302e6981794f41429ad3f73db903033a) (slightly out of date but all concepts are the same just slightly different code)


# Novel Or Nonstandard Mechanisms 
1. We charge interest even if a Borrower hasn't drawn down any actual debt. This type of interest is the `fRate` that a Borrower DAO pays to have access to immediate liquidity. It SHOULD be below the `dRate` charged when the Borrower does have debt but does not explicitly have to be less.
2. Lender repayment queue. We use the `ids` array in LineOfCredit.sol to prioritize Lenders that were drawn down on first. They must be paid back first.
3. The Arbiter is a neutral third party that mediates conversations between all Lenders and the Borrower. They have privileged access and are assumed to be honest at all times.
4. The Arbiter can declare a borrower INSOLVENT if there is no collateral left in the collateral Escrow or in the Spigot. This lets all Lenders know that whatever balance they have still deposited into the Line of Credit will never be repaid.
5. The Spigot MUST take ownership of a revenue generating contract. There must be no other access points to the underlying contract. Scenarios are identified under Known Exploits in docs


# In Scope For Audit
| File                                                   |
|--------------------------------------------------------|
| contracts/modules/credit/LineOfCredit.sol              |
| contracts/modules/credit/SpigotedLine.sol              |
| contracts/modules/credit/SecuredLine.sol               |
| contracts/modules/credit/EscrowedLine.sol              |
| contracts/modules/spigot/Spigot.sol                    |
| contracts/modules/escrow/Escrow.sol                    |
| contracts/modules/oracle/Oracle.sol                    |
| contracts/modules/interest-rate/InterestRateCredit.sol |
| contracts/modules/factories/LineFactory.sol            |
| contracts/utils/CreditLib.sol                          |
| contracts/utils/LineLib.sol                            |
| contracts/utils/CreditListLib.sol                      |
| contracts/utils/EscrowLib.sol                          |
| contracts/utils/SpigotLib.sol                          |
| contracts/utils/SpigotedLineLib.sol                    |
| contracts/utils/MutualConsent.sol                      |
| contracts/utils/LineFactoryLib.sol                     |

# Special Concerns For Audit
## Spigot 
Since the Spigot takes ownership of another protocol/DAO's smart contracts to secure their revenue streams (e.g. owning a Yearn vault to ensure vault fees repays Yearn's debt) we want to make sure that the Spigot can't get bricked locking their contract forever, that their contract can't be stolen from the Spigot and that revenue tokens are securely escrowed inside the Spigot for the Owner to claim.

## SpigotedLine
Our integration between the Line of Credit and Spigot contracts. It owns the Spigot so it's important that it doesn't lose ownership of the Spigot (unless `releaseSpigot()` successfully executes) so that it can properly manage and call the Spigot contract. It must be able to trade revenue tokens captured by the Spigot to credit tokens owed to Lenders using 0x protocol. 

# Smart Contracts

## LineOfCredit (478 sloc)
LineOfCredit.sol is the core contract responsible for:
- Inherited by SecuredLine.sol
- Recording credit lines, positions and accounting for Borrowers and Lenders
- Defining Line of Credit terms (Oracle, Arbiter, Borrower, term length, interest rate, escrow and spigot collateral)
- Coordinating the Escrow, Spigot, and InterestRateCredit modules

- External calls to Oracle.sol and InterestRateCredit.sol
- Libraries - LineLib.sol, CreditLib.sol, CreditListLib.sol


## SpigotedLine (247 sloc)
 - An integration between Spigot.sol and LineOfCredit.sol. 
 - Inherited by SecuredLine.sol
 - It owns the Spigot so it's important that it can properly manage and call Spigot.sol and doesn't lose ownership of it unless `releaseSpigot()` successfully executes. 
 - Manages a Spigot's configuration based on the health status of a Line of Credit
 - Trades a DAO's Revenue Tokens for Credit Tokens owed to lenders using 0x protocol
 - Stores excess revenue or trade slippage in 'unused' tokens for later use in repayment
 - Allows Borrowers to clawback escrowed tokens if a Line of Credit is fully repaid
 - Allows liquidating 'unused' tokens or the Spigot itself if a Line of Credit's status is LIQUIDATABLE

 - External calls to - 0x protocol, Spigot.sol
 - Libraries - LineLib.sol, SpigotedLineLib.sol


## EscrowedLine (60 sloc)
EscrowedLine.sol is an *abstract* contract holding all the collateral of a Borrower.  
- Inherited by SecuredLine.sol
 - It doesn't contain any external functions.
 - Allows an Arbiter to liquidate collateral if the Line of Credit's status is LIQUIDATABLE
 - Updates a Line of Credit's status based on the latest collateral ratio vs minimum collateral ratio (in Escrow.sol)

 - External calls to - Escrow.sol
 - Libraries - LineLib.sol


## SecuredLine (98 sloc)
 - Combines the logic of LineofCredit.sol, EscrowedLine.sol, and SpigotedLine.sol to create a fully secured lending solution.
 - Allows transferring all collateral to a new Line of Credit contract
 
 - External calls to - Oracle.sol, InterestRateCredit.sol, EscrowedLine.sol, Spigot.sol, 0x protocol
 - Libraries - LineLib.sol, SpigotedLineLib.sol

## LineFactory (179 sloc)
- Deploys SecurdLine, Spigot, and Escrow contracts 
- Has functions for default deployments and with custom config for borrower/lenders
- After deploying SecuredLine's ensures that modules are configured properly and Line is ACTIVE

---

# Escrow (114 sloc)
 - Allows a Borrower to deposit tokens as collateral for a Line of Credit
 - Allows a Borrower to withdraw collateral so long as that actions does't cause the status of a Line of Credit to be LIQUIDATABLE
 - Allows Arbiter to whitelist (enable) specific collateral allowed for a Line of Credit
 - Allows Arbiter to liquidate collateral if the status is LIQUIDATABLE

 - External calls to - Oracle.sol, LineOfCredit.sol
 - Libraries - LineLib.sol, CreditLib.sol, EscrowLib.sol 


# Spigot (190 sloc) 
 - Takes full ownership of a DAO or a protocol's contracts to escrow revenue earned by them
 - Allows the Spigot Owner to pull escrowed funds at anytime
 - Allows arbitrary payment splits between Spigot Owner and the Spigot Treasury (a contract of the DAO/protocol
 - Allows updating stakeholder addresses

 - External calls to arbitrary contract with arbitrary calls
 - Libraries - SpigotLib.sol, LineLib.sol


# Oracle (24 sloc)
- A wrapper contract to simplify integration with Chainlink FeedRegistry
- Returns all token prices in USD 8-decimal denomination

- External calls to - Chainlink FeedRegistry
- Libraries - Chainlink Denominations


# InterestRateCredit(72 sloc)
- Stores interest rates and the last time interest was accrued for individual Lender positions on a Line of Credit
- Purely calculates interest owed. LineOfCredit.sol is responsible for updating balances
- Only allows LineOfCredit.sol to call it

--- 

# LineLib (79 sloc)
- Stores basic functions for LineOfCredit e.g. health statuses and transferring tokens


# CreditLib (208 sloc) 
-  Stores basic functions for interacting with Lender positions e.g. computing ids for individual credit lines, accruing interest and repaying debt


# CreditListLib (53 sloc) 
-  Stores functionality for interacting with collections of Lender positions on a Line of Credit e.g. adding and removing positions or re-sorting positions in the repayment queue


# EscrowLib (183 sloc)
- Calculates the total value of collateral assets escrowed
- Calculates collateral ratio based on collateral value 
- Ensures that whitelisted (enabled) collateral enabled has a price feed


# MutualConsent (48 sloc)
- Forked from https://github.com/IndexCoop/index-coop-smart-contracts/blob/1acec44229b3aaf4a40dad2095b0cc6accb8fbfc/contracts/lib/MutualUpgrade.sol
- Essentially a 2/2 multisig baked into your contract
- Ensures that two predefined addresses both sign a tx with the same inputs and then executes the function with those parameters. 


# SpigotLib (212 sloc)
- Stores functionality for claiming revenue, updating revenue splits, updating stakeholder addresses and all other Spigot functions


# SpigotedLine.Lib (187 sloc
- Stores functionality related to including Spigot in Line of Credit lifecycle


# Out of Scope For Audit
| File                                                   |
|--------------------------------------------------------|
| contracts/interfaces/*                                 |
| contracts/modules/factories/*                          |
| contracts/utils/SpigotedLineLib.sol                    |
| contracts/utils/LineFactoryLib.sol                     |

Includes:
1. Anything related to the Arbiter
2. Anything related to the Oracle
