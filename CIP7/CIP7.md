| CIP  | Title                                   | Authors                   | Comments-URI | Status | Type      | Created    | License    |
| ---- | --------------------------------------- | ------------------------- | ------------ | ------ | --------- | ---------- | ---------- |
| 7    | Unique ticker occupation and numeration | Markus Gufler <m@rkus.it> |              | Draft  | Standards | 2020-09-17 | Apache-2.0 |

# Abstract

It's about an automatic unauthorized mechanism to protect the Ticker of a Cardano Stakepool operator against imitation.

At the same time it is also an extension of the existing simple name to standardize the numbering of several pools of a single operator.

# Motivation

In a decentralized environment like the Cardano Blockchain there should be **no central authority** that decides on the allocation and **uniqueness of a pool ticker**, as is the case with domain names.

Without any protection mechanisms, this means that a ticker name of a pool operator, which may involve a lot of effort for brand, reputation and awareness, can be **imitated by others at any time**. 

A solution in which a ticker, once registered on-chain, is **reserved forever is unthinkable**, as this would make hoarding reservations and high redemption sums only a matter of time.

We would like to propose a solution that provides **a momentary guarantee** and security. Both for the operator and the users who need to be protected from fraud attempts.

At the same time, this redefinition of the ticker handling should also take into account the **phenomenon of multiple tickers of a single pool operator**. Especially if the K-parameter should be increased significantly with the desired number of pools from currently 150.

It should be possible for an operator to clearly identify and **register multiple pools with a single ticker**.

# Floating Uniqueness

New registrations of a pool ticker should be verified by the existing pools. If such a ticker already exists and has **recently produced blocks**, this transaction for the second same ticker will be rejected and not added to the MemPool and on chain.

A practical setup could be in the simplest case that the existing pool must have produced at least one block in the previous epoch. Or it could be that the existing pool must have created at least 2 blocks in the previous 3 epochs.

This means that **nobody can reserve many tickers permanently** without having collected a certain amount of stakes which guarantees at least one block (or more) per epoch. And with this proposal, ticker names will be **released automatically** and at short notice if an operator stops his activity. 

To avoid two new and identical ticker registrations before the first one could even produce new blocks with active stack, the existing nodes would also have to validate and reject further identical tickers for two epochs.

It is unclear, what happens, if an existing ticker does not create blocks for some epochs, another operator registers the ticker and the first one continues later.  

# Numbered Multiplicity

Already with the desired number of pools of k=150 at Mainnet start, a phenomenon of numbered pool tickers of one and the same pool operator became visible.

typically the 4th or 5th character of the ticker name was given a consecutive number: for example IOG1,2,3... ZZZ, ZZZ2, ... or OCEAN, OCEA2, ...

If the k parameter is increased in the future, and the pledge factor does not become significantly more important, this numbering phenomenon will most certainly increase substantially. 

For this reason we suggest that a pool operator can add a sequential number to his ticker name. 

for example DIGI1 and DIGI2 become DIGI.1 and DIGI.2 

In this case, DIGI is a floating reserved and thus protected ticker name.   

The delegation centers of the wallets can therefore easily recognize this connection and, in case of saturation, refer to a second pool of the same operator.



# Copyright

This CIP is licensed under Apache-2.0
