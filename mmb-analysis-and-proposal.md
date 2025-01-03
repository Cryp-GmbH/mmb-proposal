# Merkle Mountain Belt: Cost saving analysis + funding proposal V2[^original-proposal]
![Groupe Logo noir@4x](https://hackmd.io/_uploads/Hkdh8Idw0.png =10%x)
#### Cryp GmbH
##### Alfonso Cevallos, Robert Hambrock

## Table of Contents

0. [Intro](#Intro)
1. [MMB overview](#MMB-overview)
    a. [Technical features](#Technical-features)
    b. [Faster update times](#Faster-update-times)
    c. [Shorter membership proofs](#Shorter-membership-proofs)
2. [Cost saving analysis against MMRs](#MMBs-in-BEEFY)
    a. [Cost savings per transaction](#Cost-Estimate)
    b. [Cost savings in overall bridge traffic](#Traffic-Estimate)
3. [What the MMB data structure looks like](#What-the-MMB-data-structure-looks-like)
    a. [U-MMB](#Unbagged-MMB-U-MMB)
    b. [F-MMB](#MMB-with-forward-bagging-F-MMB)
    c. [MMB](#MMB-with-double-bagging-MMB)
4. [Research & Implementation Timeframes](#Timeframes)
    a. [Research Timeframe](#Research-Timeframe)
    b. [Implementation Timeframe](#Implementation-Timeframe)
5. [Cost of Proposal upon Completion](#Cost-of-Proposal-upon-Completion)
6. [Success Reward](#Success-Reward)
7. [Team](#Team)

<!---
    d. [Comparison of MMB flavors](#Comparison-of-MMB-flavors)
--->

## Intro 

Imagine this: by the beginning of 2026, decentralized applications have gained a significant foothold in the world finances. With them, blockchain platforms such as Polkadot and Ethereum move millions of dollars worth of value every day, and a successful DOT🡘ETH bridge is one of the top five cross-chain bridges in use. To realize this future for Polkadot, the time is NOW to build the most efficient and secure bridge out there. Yet this might not happen if the related bridging costs remain as high as they are now[^discussions-on-current-cost].

We propose developing a brand new data structure called Merkle Mountain Belt (MMB), namely both its theoretical and implementation sides, that would replace the use of the Merkle Mountain Range (MMR) in the BEEFY protocol used by bridges like Snowbridge and Hyperbridge. This change will considerably reduce the size of cross-chain message proofs in DOT→ETH bridging[^hyperbridge-operational-cost], in turn reducing the transaction costs for users, with estimated savings of 1-1.4 million dollars [per year](#Traffic-Estimate), at current prices.

We also propose writing a research paper in collaboration with Alistair Stewart, Head of Research at Web3 Foundation, about the new data structure, and publishing it in a top rated computer science conference. Polkadot has always been a thought leader in blockchain technology, being one of the few communities that pioneers top quality research, and not just bells and whistles. This proposal honors this Polkadot tradition. MMRs are currently widely used across multiple blockchains, and improving upon them with MMBs would heighten Polkadot's impact on and reputation within the wider ecosystem.

The members of our [team](#Team) are highly qualified and have worked in Polkadot since pre-mainnet days. Our principal researcher Alfonso has a Ph.D. in mathematics and is a co-author of many Polkadot papers, including the [2020 overview paper](https://arxiv.org/abs/2005.13456). Our principal developer Robert has been one of the earliest stewards of the Polkadot-Ethereum bridge, going back to its initial W3F grant, design of protocols, and implementation.

## MMB overview

:::info
**TLDR of MMBs & cost analysis:**
The MMB data structure serves as a plug-in replacement for MMR, to produce BEEFY commitments, which are primarily used by the DOT🡘ETH bridges. Over the next 5 years, MMB would [reduce the average proof size by over 40%](https://docs.google.com/spreadsheets/d/1PRea58H84NpxHeJirpl8KUikfqFLYvVr8l70OFrTxX4) compared to the current MMR implementation. From our gas and cost estimates, this corresponds to $1.11 of savings per transaction.
<!---If Polkadot's BEEFY bridges have comparable volume--->
:::

*See also the following presentation giving an MMB overview:*
[Sub0 2022: BEEFY: How Make Proofs Smol?](https://www.youtube.com/watch?v=uCwXbD9v3ew)

Balanced Merkle trees, hash chains, and Merkle Mountain Ranges (MMRs) are three examples of data structures widely used in blockchain as *commitment schemes*: they allow full nodes to produce a compact *commitment* to some database $X$, and then produce a small *membership proof* (a.k.a. Merkle proof) to show to a *verifier* that an item $x\in X$ is part of the committed database. 

These data structures are used everywhere in blockchain and allow for any entity to query and authenticate specific data efficiently, without having to follow consensus, download the full state, nor trust any one full node. The aforementioned structures have different properties, and one or other will be chosen depending on the specific use case. The right choice of structure can drastically reduce the corresponding data authentication costs. 

The Merkle Mountain Belt (MMB) is a new member of this family of Merkle structures, optimized specifically for the use case of BEEFY, the protocol designed for efficient DOT🡘ETH bridging: Via BEEFY, a smart contract on Ethereum periodically receives a digest of the current state of Polkadot, with a refresh rate in the order of minutes to a couple hours, and bridge users authenticate Polkadot events on Ethereum with the help of this digest. 

To oversimplify, we can say that if bridge users mostly care about Polkadot events from the last block, then the most efficient commitment scheme would be a hash chain, while if they care about very old data, the most efficient scheme would be an MMR. The MMB is a "best of both worlds" structure whose performance is always comparable to the better of the two former schemes, for any possible distribution of queries. But crucially, it outperforms them both in the case that users' queries are concentrated on events that took place in the time range from the last minute to the last day.[^comparison] 

Let's get into details.

### Technical features

Consider an item list $X=(x_1, x_2, \cdots, x_n)$ onto which new items are constantly being appended. This could be one of many databases, such as XCMP messages, but in our primary use case items correspond to Polkadot relay chain blocks.[^mmr-leaf-content] MMB is a data structure built on top of $X$ that produces a compact commitment to it. Its features are: 

* faster (constant time) updates after each append, and 
* shorter membership proofs for recently appended items.


|  | Chain | MMR | MMB |
| -------- | -------- | -------- | --- |
| Update time per append | $O(1)$ | $O(\log n)$ | $O(1)$ | 
| Proof size of the $k$-th newest item | $O(k)$ | $O(\log n)$ | $O(\log k)$ |

In this table, we assume $1\leq k\leq n$, where $n:=|X|$ is the current number of items in list $X$, and $k$ is the recency of the item for which we need a membership proof. In our use case $k$ is typically low, say between $10$ and $500$, because users mostly care about recent events. Also keep in mind that Polkadot  currently has north of $20$ million blocks produced, so we are dealing with a use case where $n$ quickly reaches the order of tens of millions.
<!---For the sake of comparison, in the following analysis we take $n=2^{24}$ as a canonical value.--->
For the sake of comparison, in the following analysis, we average $n$ over the first 5 years after anticipated MMB deployment within BEEFY on Polkadot, that is $10512000\leq n \leq 36792000$[^n-sampling-range].

### Faster update times

The main advantage of a hash chain is that it takes a single hash computation to append a new item. On the other hand, MMR, and any other structure that attempts to maintain a balanced tree, will pay an update cost of $O(\log n)$ per append, as the new leaf is immediately buried deep in the tree. In our use case, MMR requires on average about $\frac{1}{2}\log_2 n \approx 12$ hash computations (and growing) per append. 

Surprisingly, MMB achieves a **constant** cost per append: between 3 and 5 hash computations regardless of the list size $n$. To do so, it maintains a slightly unbalanced structure, where recent leaves are more shallow, though not completely unbalanced as with a hash chain. In our use case, having a faster append time is important for block authors, who have to perfom the update on-chain and on the fly as they generate blocks. 

### Shorter membership proofs

However, the killer feature of MMB for our use case is its slightly unbalanced structure, which makes recently appended items have shorter membership proofs. This leads to lower operational costs because bridge users are more likely to care about recent events[^relayer-frequency]. 

In particular a message, such as a transfer request, will first be created on Polkadot, and then sent to Ethereum, and authenticated there with the help of the BEEFY commitment stored on the bridge. Users typically want to execute this second step as soon as possible, i.e., the next time the BEEFY commitment is updated on the bridge. 

Hence, in order to ensure user friendliness, we need two things:

1. Short intervals between commitment updates in the bridge. Popular cross-chain bridges update their commitment every 15 minutes to 30 minutes. This could be reduced to five minutes, or increased to a couple of hours, depending on the chosen trade-off between operational cost and user friendliness. 
2. A structure, like MMB, that minimizes the membership proof size of items added in the last five minutes, to the last couple of hours.

<a id="proof-size-comparison"></a>

| Recency | Chain | MMR | MMB |
| -------- | -------- | -------- | --- |
| 6 seconds ($k\leq1$) | $1$ | $11.10$ | $2.13$ |
| 1 minute ($k\leq10$) | $5$ | $14.52$ | $4.96$ |
| 5 minutes ($k\leq50$) | $25$ | $15.82$ | $7.84$ |
| 15 minutes ($k\leq150$) | $75$ | $16.65$ | $9.95$ |
| 30 minutes ($k\leq300$) | $150$ | $17.15$ | $11.31$ |
| 1 hour ($k\leq600$) | $300$ | $17.65$ | $12.67$ |
| 4 hours ($k\leq2400$) | $1200$ | $18.66$ | $15.41$ |
| 24 hours ($k\leq14400$) | $7200$ | $19.95$ | $18.97$ |

*Average proof size (in hashes) of an item appended at different recency values (time from the present), assuming that a new item is appended every 6 seconds (like Polkadot relay chain blocks in BEEFY). The results are averaged across ($10512000\leq n \leq 36792000$), i.e., the size of the list between 2-7 years of appending.*

<!---
| Recency | Chain | MMR | MMB |
| -------- | -------- | -------- | --- |
| 1 second ($k\leq1$) | $1$ | $12$ | $2.1$ |
| 1 minute ($k\leq10$) | $10$ | $14.5$ | $6.5$ |
| 5 minutes ($k\leq50$) | $50$ | $15.7$ | $9.9$ |
| 20 minutes ($k\leq200$) | $200$ | $16.7$ | $12.6$ |
| 1 hour ($k\leq600$) | $600$ | $17.5$ | $14.7$ |
| 6 hours ($k\leq3600$) | $3600$ | $18.8$ | $18.3$ |
| 24 hours ($k\leq7200$) | $7200$ | $19.3$ | $19.7$ |
--->

We see that MMB produces shorter proofs in expectation than both MMR and a hash chain, for any commitment refresh rate between once per minute and once per day. 

<a id="MMBs-in-BEEFY"></a>

## Cost savings analysis against MMRs

In an Ethereum bridge, a smart contract on Ethereum stores a commitment to Polkadot's state. Currently, this commitment is built with an MMR, where a new leaf is added for each Polkadot relay chain block, and in turn each leaf contains commitments to parachain headers. For Snowbridge, when a message  goes over the bridge, either a user or a relayer has to authenticate it in the smart contract, by attaching an MMR proof and a header proof to the message:
1. The MMR proof shows that the MMR (whose root is held in smart contract) commits to a leaf, which in turns commits to a particular parachain header.
2. The header proof shows that this parachain header indeed contains the claimed content, such as a message.

As the number of Polkadot blocks grows, so do the MMR membership proofs that have to be relayed to Ethereum. In contrast, in MMBs the size of a membership proof is *independent* of the size of the structure -- its asymptotic behavior is instead governed by the recency of the corresponding leaf. As bridge users primarily care about recent events, MMB will offer much smaller membership proofs than MMRs on average.

<a id="Cost-Estimate"></a>

### Cost savings per transaction

We calculate the expected transaction cost saving from switching from MMRs to MMBs in the first 5 years after we conservatively expect the MMB upgrade to go live, namely Q1 2026. Since this is approximately 2 years after BEEFY got activated on Polkadot (February 2024), we'll average the MMR size over ($10512000\leq n \leq 36792000$), as done before in the [table above](#proof-size-comparison).

To compare both schemes, we consider the membership proof of leaves up until the 150th most recent leaf. This value $k\leq150$ corresponds to querying a message that was created on Polkadot up to 15 minutes ago (+ 24 minutes; see note on RanDAO[^randao]). For $k\leq150$, the average MMR proof size, measured in hashes, is 16.65, against 9.95 for MMB proofs, hence MMB saves 6.7 hashes: a proof size reduction of over 40%.

Via gas profiling from Snowfork's implementation, every additional hash in the membership proof of an MMR leaf adds approximately 2'568 gas of computation cost on Ethereum: [`Lederstrumpf/snowbridge/rhmb/profile-proof-item-gas-cost`](https://github.com/Snowfork/snowbridge/compare/main...Lederstrumpf:snowbridge:rhmb/profile-proof-item-gas-cost?expand=1#diff-ad43af2a8fe32a0a552c025fc1d86308adf4ee1e4d45f83070cce458c461c818R85-R87)

Assuming a gas price of 23.87 GWEI (365 day SMA via [Glassnode](https://studio.glassnode.com/metrics?a=ETH&category=&chartStyle=column&ema=0&from_exchange=aggregated&m=fees.GasPriceMean&mAvg=365&mMedian=0&resolution=24h&s=1687097402&to_exchange=aggregated&u=1718633402&utm_campaign=woc_02_2023&utm_medium=insights_woc&utm_source=gn_insights&zoom=365)), and an ETH price of $2'696.79/ETH (365 day SMA via [Glassnode](https://studio.glassnode.com/metrics?a=ETH&category=Market&chartStyle=column&ema=0&from_exchange=aggregated&m=market.PriceUsdClose&mAvg=365&mMedian=0&resolution=24h&s=1687097402&to_exchange=aggregated&u=1718633402&utm_campaign=woc_02_2023&utm_medium=insights_woc&utm_source=gn_insights&zoom=365)), the cost per hash is 16.57 cents.

Consequently, since an average proof size of a leaf in an MMR of depth up to $k\leq150$ is 16.65, the expected proof verification cost for such a leaf is $2.75. In contradistinction, with MMBs the expected proof size is 9.95, hence the cost goes down to $1.65. Using an MMB rather than an MMR for the BEEFY commitment thus would effect a saving of 6.7 hashes, or $1.10 per message sent to Ethereum.
<!---committed list of size $n=2^{24}$ (current Kusama/Polkadot block counts),--->

For more calculation details and a 1000-day[^glassnode-maximum-range] SMA comparison, see the [Cost Analysis Spreadsheet](https://docs.google.com/spreadsheets/d/1PRea58H84NpxHeJirpl8KUikfqFLYvVr8l70OFrTxX4).

***Note:** These costs are saved by bridge users and -- in case of a subsidy -- the Polkadot treasury[^treasury]. In either case, these fees would otherwise just be burned on Ethereum, with no benefit toward the Polkadot ecosystem. This reduction in running costs will make the Polkadot🡘Ethereum bridge more competitive with other bridges, and ensure its short-term adoption and long-term upkeep.*

<a id="Traffic-Estimate"></a>

### Cost savings in overall bridge traffic

To estimate the transaction numbers we can expect for a successful DOT🡘ETH bridge once it is mature, we take as a proxy the mean monthly transaction volumes of the 5 most utilized bridges.
<!--- TODO: maybe make the following a footnote to improve flow --->
Our estimate is thus based on the assumption that at least one DOT🡘ETH bridge bridge is a success. This is the same reason we chose 15 minutes (+ 24 minutes ranDAO [^randao] for Snowbridge) as a reference latency, given that such latency is offered by competitors like Stargate and Optimism, and even lower for trusted bridging setups.

A comparison, using data from 31 August 2024, is available here: [Cost Analysis Spreadsheet](https://docs.google.com/spreadsheets/d/1PRea58H84NpxHeJirpl8KUikfqFLYvVr8l70OFrTxX4). From it, we expect an average of 2'646 transactions per day, or 965'790 transactions per year.

**Result:** *The estimated cost saving from switching from MMRs to MMBs is in the range of $1'069'000/year - $1'397'000/year.*

**Note:** *On top of these expected savings for a bridge to Ethereum, MMB will also provide gas cost savings to bridges from Polkadot/JAM to Layer-2s such as Arbitrum and Base. A recent back-of-the-envelope analysis of these L2 gas cost savings is available [here](https://hackmd.io/@CrypGmbH/Cost-Saving-L2-Back-of-the-Envelope). Note that since this calculation is still preliminary and will require much deeper investigation to become robust, we opted not to incorporate these additional savings into our estimate above.*

## What the MMB data structure looks like

### Unbagged MMB (U-MMB)

In what follows we assume familiarity with the MMR data structure, and we build from there. Consider a list $X=(x_1, x_2, \cdots, x_n)$ we want to commit to. As in MMR, we build $O(\log n)$ *mountains*, where each mountain has an associated size $s$ and is a complete binary tree holding $2^s$ leaves. Each leaf is labeled with the hash evaluation of an item in $X$, and each non-leaf is labeled with the hash evaluation of the concatenation of its children. 

![MMB9](https://hackmd.io/_uploads/rkDvrCIDA.png)


This is the MMB structure for $n=9$ (without bagging), with three mountains of sizes $2,2,0$, whose peaks are in gray. 


We say two mountains of equal size $s$ are *mergeable*, as they can be merged into a mountain of size $s+1$ by creating a new peak node as the parent of the two old peaks. We follow this rule: whenever we append a new item to list $X$, we add it as a mountain of size $0$ (i.e., a singleton), and if there are mergeable pairs of mountains at that point, we merge *exactly one* such pair: the rightmost one.


![image](https://hackmd.io/_uploads/rkyZbvQvA.png)

For instance, here we see two append updates, with the new leaf in green and the new merge peak in gray. When appending $x_{10}$ we merge two mountains of size $0$ into one of size $1$, and when appending $x_{11}$ we merge two mountains of size $2$ into one of size $3$. 

In contrast, MMR follows the rule that any mergeable pair is merged immediately, which can lead to a domino effect of up to $O(\log n)$ merges after a single append. 

With our lazy merging rule, we make sure each append has constant complexity. But more importantly, we make mountains grow at a slow and steady rate, so that the $i$-th rightmost mountain is always of size $\Theta(i)$. In turn, this gives us the asymptotic behavior we want: the $k$-th most recent leaf is at distance $O(\log k)$ to its peak, regardless of the list size $n$.

### MMB with forward bagging (F-MMB)

Next, we want to group or *bag* the mountains into a single structure -- a *mountain range* -- to create a single root that commits to the full list $X$. We add an upper layer of *bagging nodes*, where each node is the parent of one peak and the previous bagging node. 

![image](https://hackmd.io/_uploads/BkbFWaEwC.png)

In the figure we see two possible ways of bagging the same set of mountains: in a backward fashion on the left, and in a forward fashion on the right. In each case, the root is in brown, peaks in gray, and the symbol $\bullet$ represents a non-existent child node.

Backward bagging is usually used in MMR: it leads to a more balanced structure because larger mountains (on the left) end up closer to the root, so items with a large leaf-to-peak distance are compensated with a short peak-to-root distance. However, we will embrace unbalancedness and choose forward bagging, for two reasons. 

First, it preserves the property we want: that the $k$-th most recent leaf maintains an $O(\log k)$ depth, regardless of the list size $n$. This is because both the leaf-to-peak and peak-to-root distances will be $O(\log k)$. And second, with forward bagging most bagging nodes don't need to be updated after an append, as changes only happen near the rightmost end of the structure. This optimizes the append operation.

![image](https://hackmd.io/_uploads/SJO7LpEPC.png)

E.g., here we see a series of append updates, if we use a forward bagging. For simplicity, we hide all non-peak mountain nodes, representing each mountain only by its peak labeled with its mountain size, and we place bagging nodes on a single horizontal level. The new leaf is in green, the merge peak in gray, and the updated bagging nodes in brown.

It can be proved that, with forward bagging, only two bagging nodes need to be updated on average per append. However, in the worst case we still need to update $O(\log n)$ bagging nodes (i.e., most of them), namely when the mountains being merged are large, and far from the rightmost end of the structure. We need one final ingredient.

### MMB with double bagging (MMB)

In the final version of MMB, we introduce a more complicated form of forward bagging that we call double bagging. Instead of grouping all mountains into a single range, we partition mountains into subsets, use forward bagging to group each subset into a range, and finally use an extra layer of forward bagging to group all mountain ranges into a *mountain belt*.

![image](https://hackmd.io/_uploads/Bkz6tDBv0.png)

E.g., here is the final MMB structure for $n=1337$. There are $10$ mountains (represented only by their peaks) partitioned into $5$ mountain ranges, each range is forward bagged with rhombi nodes, and finally the $5$ ranges are forward bagged with circle nodes into one mountain belt, whose root is in brown.

The point of splitting the mountains into several ranges is to ensure that each mergeable pair of mountains sits at the right end of a range. Hence, after a merge, we only need to update one bagging node in the affected range. And importantly, we also only need to update one or two bagging nodes in the outer bagging. This is because we always merge the rightmost mergeable pair, which has to be in either the rightmost or second rightmost range. In turn, this is by our range splitting rules: if there are no mergeable pairs to the right of the affected range, then there cannot be any range splits either.

![MMBs](https://hackmd.io/_uploads/BJLNCPHw0.png)

Here we see a series of append updates in MMB. Again, the new leaf is in green, the merge peak in gray, and we color in brown all bagging nodes that need to be updated. 

Notice that at most $4$ bagging nodes are updated, even when we merge large mountains! Thus, we achieve a constant complexity for the append operation even in the worst case. 

The full structure will be presented in detail in the research paper, where we will show that we get all the properties we want: an append operation with a constant complexity, and a depth of $O(\log k)$ for the $k$-th most recent leaf, regardless of the list size $n$.

<!---
### Comparison of MMB flavors

We highlight that in some scenarios it might be a good idea to implement one of the variants of MMB presented above. 

As U-MMBs omit the bagging step, they are much simpler to implement, and more importantly, they offer shorter membership proofs than MMB (about half the size), which in our use case translates to even smaller transaction costs for bridge users. On the down side, they require changes to the Ethereum light client and increase the data storage cost of the BEEFY commitment (i.e., increase the operational cost of the bridge). This is because the commitment would correpond to the list of hashes of all $O(\log n)$ mountain peaks. Full MMBs on the other hand require no changes to Snowbridge's Ethereum client.

F-MMBs are also simpler to implement than MMBs, because they use a simpler bagging process. Just as MMBs, in our use case they serve as a drop-in replacement to MMRs on the Ethereum light client. However, they have a $O(\log n)$ worst-case update time per append, against costant time for MMBs.  Likewise, F-MMBs have larger increment proofs[^increment-proofs] than MMBs.
--->

<a id="Timeframes"></a>

## Research & Implementation Timeframes
This funding request forks into a Research Track and an Implementation Track. The key deliverables are
1. a research paper,
2. an analysis of use cases for MMB in addition to BEEFY bridging, and
3. a Rust implementation of Merkle Mountain Belts integrated into a frame pallet, together with any necessary changes to Snowfork's BEEFY client necessary to accommodate:
https://github.com/Snowfork/snowbridge/blob/main/contracts/src/BeefyClient.sol

An additional success milestone is described in the [success reward section](#Success-Reward). The key deliverables for this track are
1. publication of the research paper in a well-respected conference, and
2. deployment of MMBs in Polkadot's BEEFY protocol.

### Research Track 
---
#### M1. Paper: **18 weeks (162'000 CHF)**
The primary milestone of the research track is completing a paper on MMBs currently in the works. The timeframe for completion of this paper is so short since a lot of the work has already been completed by the team members in collaboration with Alistair Stewart, for which we already received funding from Web3 Foundation, and we are not asking for any of this prior work to be funded again. Alistair Stewart will be a co-author on the paper.

The paper will present the novel data structure MMB, as well as its variants U-MMB and F-MMB. We will make a thorough comparison of these structures against (several variants of) MMR and the hash chain, highlighting the pros and cons of each one. In particular, we will focus our analysis on blockchain applications, and how these structures fit in the design of light client protocols. 


##### Writing: 12 weeks
Completion of paper, including considerable work needed in:

1. An analysis of amortized proof sizes for variants of MMB and MMR, i.e., find the exact constants hidden in big-O notations for long-term performance. 
3. An analysis of asynchrony: how we can make an outdated MMB proof remain valid against an updated commitment, and vice-versa. 
4. Increment proofs: how to efficiently prove that an old commitment actually corresponds an earlier version (a prefix) of the canonical blockchain, and not to a fork with invalid blocks. For instance, slashing of cross-chain equivocations requires increment proofs.
5. Pseudo-algorithms, and
6. Missing proofs. 


##### UTXO analysis: 6 weeks

Notice that MMB gives shorter proofs to recent items in the committed database. We conjecture that this is universally appealing for blockchain applications, because it is often the case that recently generated data needs to be queried and authenticated much more frequently than older data. 

To substantiate our conjecture, and add important theoretical motivation to the paper, we will study the lifespan of transaction outputs (TXOs), measured in blocks, from their creation until the time they are spent. In particular, Bitcoin and unshielded ZCash will be analysed, as their data is vast and easily accessible. Is this distribution heavily skewed towards recency? Would MMB make for a shorter proof size in expectation, relative to MMR? Following our conjecture, this should be the case, but it will have to be justified with data.

This milestone's time budget accounts for anticipated research timeframes such as data collection, analysis, and bibliographic research. The analysis we envision will employ a more differential methodology than exhibited here:
https://www.nature.com/articles/s41597-022-01254-0

---
#### M2. Exploration: 12 weeks (108'000 CHF)
We add a buffer of twelve weeks to allocate to further investigations, including fine tuning MMB to different applications. These may end up in the paper too, but this would be subject to, among other factors, the outcome of this research, timing with respect to a suitable conference, and the shape of the paper. If not included in the paper, the findings will nonetheless be made available to the Polkadot community, for instance in the format of research reports.

Here are a few of the areas we will research, ordered by descending priority:
##### 1. Performance of different Merkle structures for a given sampling distribution

Assume you are given a database $X$, and you know that items will be queried from it by users following a specific distribution. What Merkle structure should you use to commit to $X$, to minimize the expected proof size of the queried items? For instance, we can intuitively guess that if the distribution is uniform, we should use a balanced Merkle tree, but if half of all queries are for one item, then we should place this item's leaf as a direct child of the root. 

We would like to make a thorough analysis of this question for commonly observed distributions, such as exponential distributions and zeta distributions. This would be a valuable guide for blockchain protocol designers, so they know exactly what Merkle structure to use for each use case.

In fact, we conjecture that MMB and their variants will be ideal structures for various members of the family of zeta distributions, which are common in real-life scenarios.

##### 2. XCMP
If, as currently envisioned, XCMP has unordered message delivery, then MMBs are a suitable data structure to use as a drop-in replacement for MMRs. The advantages of MMBs for this use case are: 

- much shorter proofs (and hence smaller operational costs) since, again, queries and authentication will be made mostly for recent messages; and 
- on top of archival nodes who store full databases, MMBs allow for the network to have lighter nodes who only store the last $k$ messages (for an adecuate value of $k$), and whose running costs depend only on $k$ and not on $n$  (the total number of messages), yet produce the same membership proofs for recent items than archival nodes.

This flavor of MMBs would not be append-only (vis-à-vis for MMRs), but would require an update protocol allowing leaves to be replaced. An update proof replacing the prior item $x$ with item $x'$ could naively be achieved by first providing a membership proof for $x$ relative to the current MMB root $r$ (the known commitment), and then re-using the verified proof items to calculate the new root $r'$ from $x'$. Again, shorter proofs would make this update faster.

Note that the benefit of shorter proofs for recent items in MMB is doubled in an (on-chain!) update protocol since the co-path for the item has to be traversed twice: once for the membership proof, and then again for the calculation of the new root.

##### 3. Account-based chain analysis
The UTXO analysis we will include in the paper substantiates our hypothesis, but extending it to account-based chains is more applicable to the DOT🡘ETH bridge.
<!--- TODO: extend --->

<!--- TODO: add
##### POPOS (TODO)
https://arxiv.org/pdf/2209.08673.pdf
--->

##### 4. Flyclient integration
https://eprint.iacr.org/2019/226.pdf
The light-client protocol Flyclient is a good candidate for MMB integration since its sampling protocol is well aligned with the asymmetric distribution of membership proof sizes of MMBs: to have a probabilistic guarantee of correctess in the construction of a PoW blockchain, Flyclient samples blocks at random, but with a heavy bias towards more recent blocks. Hence, having a commitment scheme that gives recent blocks shorter proofs, may make the Flyclient implementation more efficient. 


#### M3. PoC implementation: **6 weeks (54'000 CHF)** 

It is common for research papers about new algorithms, to include a proof-of-concept implementation, to show there are no hidden obstacles to its applicability. We will publish a Clojure implementation as open source, and refer to it in the paper. We highlight that this implementation is already at an advanced stage, hence the short time budget, and that it will be also an invaluable tool for us as we build the (much more complex) Rust implementation of the library. 

   1. Implement [increment proofs](#Writing-budget-12-weeks): 3 weeks
   2. Refactor implementation to facilitate specification process: 1 week
   3. Profiling and performance improvements: 2 weeks
      
*The Clojure implementation currently exhibits worse asymptotic behavior than we know is theoretically possible. Profiling and performance improvements here are restricted to only cover improvements that will also reflect in the specification and Rust implementation of the library.* 

Each one of the three milestones in the Research Track will be considered delivered when
- It has been made publicly available, and 
- Its content has been greenlit by at least two members of the research team at Web3 Foundation. 
      
---
### Implementation Track

The current implementation of MMBs is available here: 
<https://github.com/w3f/merkle-mountain-belt-clj>

We propose developing a Rust library readily integrateable by relevant Rust pallets such as `pallet-beefy` to be used instead of [`nervosnetwork:merkle-mountain-range`](https://github.com/nervosnetwork/merkle-mountain-range). The library will be licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0.txt). Our time estimate for the implementation of a production-ready MMB library on top of the existing MMR library from Nervos is 41 FTE weeks (i.e. about 9 months) of engineering:
<!---While Nervos' MMR library has a few...--->

#### M4. Spec MMB implementation for porting to Rust: **2 weeks (18'000 CHF)**
   *In particular, this will cover interfaces required for integrations with other pallets (e.g. `pallet-beefy`) as currently provided by `pallet-mmr`*
#### M5. Rust implementation: unbagged MMBs: **18 weeks (162'000 CHF)** 
   1. data storage: 4 weeks
   2. U-MMBs: 12 weeks
   3. membership & increment proofs (for U-MMBs only): 2 weeks
#### M6. Rust implementation: bagged MMBs: **15 weeks (135'000 CHF)**
   1. forward-bagged MMBs: 3 weeks
   2. double-bagged MMBs: 8 weeks
   3. membership & [increment proofs](#Writing-budget-12-weeks) (forward- and double-bagged MMBs): 4 weeks
   
Each one of the three milestones in the Implementation Track will be considered delivered when
- It has been made publicly available, and 
- Its content has either been greenlit by at least two members of the Polkadot Technical Fellowship of rank II or higher, or been deployed within Polkadot or Kusama (implicit greenlighting). 

## Cost of Proposal upon Completion

For the 36 weeks of research, we request 324'000 CHF, and for the 35 weeks of implementation, we request 315'000 CHF, totalling 639'000 CHF. 

However, we consider that the proposal merely being *completed* is much different than it being *successful*, and these outcomes should be rewarded distinctly. In fact, we see the potential for our project to be very successful. Hence, to put our money where our mouth is, we are willing to cut our payment to only 70% upon completion, with the rest paid upon success. 

It will work as follows. Once M1 is delivered, the payment will be of $162'000\mbox{ CHF} \times 0.7 = 113'500\mbox{ CHF}$, and so on for each milestone. Hence, the total cost of the proposal upon completion of milestones 1 through 6 will be $639'000\mbox{ CHF} \times 0.7 = 447'300$ CHF. 

*The cost of each milestone will be due in USDC based on the [Swiss National Bank USD/CHF foreign exchange rate](https://www.snb.ch/en/the-snb/mandates-goals/statistics/statistics-pub/current_interest_exchange_rates) at the date of submission.*

The remaining 30% of 639'000 CHF will only be paid if and when the project meets all three of the success criteria defined in the [success reward section](#Success-Reward). In particular, one of these criteria requires that the effect of the proposal is economically net-positive for the community. 

*This 30% portion, i.e., $639'000\mbox{ CHF} \times 0.3 = 191'700 \mbox{ CHF}$, will be due in DOT based on the [Swiss National Bank USD/CHF foreign exchange rate](https://www.snb.ch/en/the-snb/mandates-goals/statistics/statistics-pub/current_interest_exchange_rates), and the 7-day EMA of USD/DOT via [Subscan](https://polkadot.subscan.io/tools/price_converter) at the date of submission.*

Notice that the success reward structure contains additional rewards, one fixed and one proportional to our library's incurred savings. Hence, the total cost of the proposal may be greater than 639'000 CHF if the deployment is an economic success, but strictly in proportion to the savings it brings to the community. See the following section for details.

## Success Reward
The milestones we request funding for above all constitute deliverables that do not depend on externalities, and where the scope of the work is thus more straightforward to estimate. 

However, even if our paper is technically sound, acceptance of the paper by the first conference we submit it to is all but guaranteed. Likewise, the upgrade of MMRs to MMBs within BEEFY on Polkadot requires collaboration with multiple other stakeholders, and thus estimating the required work on our part is unreliable.

### Success Criteria

We therefore define the following three sucess criteria for our project, which are subject to externalities. Any success reward will only be paid once all three of these creteria are met.

#### SC1. Academic success: conference paper presentation
<a id="publication-location"></a>
In addition to the open-access [arXiv](https://arxiv.org/), we plan to submit the paper to the proceedings of a suitable conference. The choice of conference will be made once the paper is sufficiently close to completion so that we can submit it by the proceedings deadline without sacrificing scope & quality of the paper. For instance, *Advances in Financial Technologies* or *Financial Cryptography* would be fine candidates for submission. 

We remark that this goal demands considerable resources, including possibly submitting the paper to multiple conferences, addressing the paper's submission feedback, conference acceptance fees, and of course travel and accommodation costs for attending the conference where we will present our work. We will cover these costs privately.

#### SC2. Engineering Success: Polkadot deployment
Once the library is fully developed and ready for use in BEEFY, it will still have to be deployed. This intense process will not only require our continuous involvement, but also close collaboration with the Technical Fellowship, and for some steps also the Snowfork and/or the Hyperbridge team.

Likewise, the code will have to be audited before deployment on Polkadot. This would either be covered by a separate proposal[^audit-cost] or via e.g. an SRLabs retainer.

Since these upcoming steps involve multiple moving parts, it is hard to provide a reliable time estimate for completing these.

This work for this success criterion is deemed complete once MMBs are deployed within BEEFY on Polkadot/JAM and used by at least one bridge, such as Snowfork or Hyperbridge.

#### SC3. Economic success: Breakeven of costs for community
Once the library is deployed within BEEFY, it will immediately start saving users' transaction fees. We consider the MMB library to be an economic success once the accrued fee savings it produces for the community exceed the fixed cost of our proposal. 

This success criterion will therefore be reached once the Polkadot/JAM ecosystem's cost savings of MMB against MMR exceeds the sum of 639'000 CHF[^success-reward-calculation-method] -- corresponding to the full cost of completing milestones 1 through 6  -- and the to-be-determined audit cost. Notice we internally will cover the cost of achieving success criteria 1 & 2, save for audits. This threshold ensures that any success rewards paid cannot result in a net financial loss for the community.

### Reward Structure

As we mentioned before, we think that our project has the potential to be very successful, far beyond mere completion, and bring lots of value to the community. We considering it fitting to have a long-term incentivization structure in place, for the following reasons:

1. This approach falls in line with the innovative and -- within crypto -- recently popularized mechanism of retroactive public goods funding (*retroPGF*, https://app.optimism.io/retropgf, https://unchainedcrypto.com/retroactive-public-goods-funding/, https://forum.polkadot.network/t/a-price-discovering-treasury-proposal/9083/10) to give open-source projects that have high impact but no classical exit mechanism -- like a token or an IPO -- a share of their work's upside.
2. It gives us a clear incentive to continue both maintaining and improving the efficiency of the MMB library, without requiring additional treasury funding. As experts in Polkadot bridges and authentication structures, we are confident we can bring further optimizations to the library for years to come, leading to even more cost savings. This aligns our long-term incentives with those of the community.
3. Usually, there is only a vague sense of how to quantify a project's impact. We are in the unusual position of having a precise and sensible metric (gas cost saving) to measure our proposal's impact.

We propose a reward structure with two components, both of which will be triggered at the moment that all three success criteria are met:

- The remaining 30% of the cost of completion of Milestones 1-6, that was mentioned in the [Cost of Proposal upon Completion](#Cost-of-Proposal-upon-Completion) section, i.e., $639'000\mbox{ CHF} \times 0.3 = 191'700 \mbox{ CHF}$.
- A variable reward consisting of 20%-30% share of the provable cost savings incurred by our implementation of MMB against the cost that would have been incurred by remaining with MMR, to be rewarded over 10 years (subject to our team continuing to maintain[^maintenance-scope] our MMB library), starting from the moment all three success criteria are met.
Specifically, as decided by community vote: 
Of annual savings below $1.4M (the upper bound of our cost saving estimate), we receive 30% as success reward. Of annual savings above $1.4M, we receive 20% as success reward.

We highlight that the third success criterion ensures that these rewards will only be paid once a net positive economic outcome is guaranteed for the community. In the unlikely scenario that the total cost savings never exceed the upfront cost of the proposal, there will be no success rewards. 

If the community prefers, we can also reduce the savings share for a perpetual fee - we are happy to receive input on this.

We'd like to explicitly point out that the payout of these success rewards will *not* occur automatically (such as via integration into Snowbridge's smart contracts), but will be subject to approval by the Treasury, where we will submit referenda that include a calculation of the actual cost savings of the MMB library against the prior MMR implementation, provable using on-chain data[^success-reward-calculation-method]. In particular, the variable rewards will require a yearly referendum to be approved. 

## Team 


### Alfonso Cevallos 

![Tenerife2](https://hackmd.io/_uploads/HkkcEdMX1g.jpg =25%x)

- Mathematics PhD from EPFL (2016). Postdoc at ETH Zurich, Switzerland (2017-2018).
- Researcher at Web3 Foundation (2019-2024): 
focus on scalability, security and incentives of decentralized protocols.
- Co-author of the [2020 Polkadot implementation paper](https://arxiv.org/abs/2005.13456).
- [Co-designer](https://www.youtube.com/watch?v=OdbCgdQugKM) and [paper co-author](https://arxiv.org/abs/2004.12990) of Polkadot's Nominated Proof-of-Stake (NPoS). 
- [Paper co-author](https://arxiv.org/abs/2312.11408) on study about real-life security metrics in Polkadot validator elections. 
- [Paper co-author](https://eprint.iacr.org/2024/961) of ELVES block auditing, the basis of Polkadot's scalability. 
- Content creator and lecturer in several editions of the Polkadot Blockchain Academy. 

### Robert Hambrock

![image](https://hackmd.io/_uploads/Bk49rOzQ1g.png)


- Honours postgrad in Physics from UCT, South Africa (2016, with distinction). Worked primarily on [AdS/CFT correspondence](https://scholar.google.com/citations?user=rluJ3bAAAAAJ&hl=en&oi=ao). Visiting scientist at CERN 2017, 2018.
- Ethereum CBC Casper: co-author of [Rust implemenation](https://gitlab.com/TrueLevel/casper/core-cbc)
- switched over to primarily working on Polkadot ecosystem in Q1 2020 (also operate validator [🚂 Zugian Duck 🦆](https://insights.math-crypto.com/polkadot/15jrQX54HczCKJgtYYoKvzJ2kgyCyyA4kyvMv2bC8x9UtDpn/))
- at Web3 Foundation (2020-2022) as Quality Assurance Engineer and then Research Engineer, later at Parity (2022-2024) as Core Engineer on bridge team.
- ETH bridge: various work over the years, e.g. on coordination of Snowfork's initial w3f grant, work on [subsampling protocol](https://hackmd.io/ohOt4jAPT8uu-soJXHUq0Q), [cross-chain slashing implementation](https://github.com/paritytech/polkadot-sdk/pull/1903), [session skipping](https://github.com/Snowfork/snowbridge/pull/1215), and [fallback governance proposal](https://hackmd.io/MlVt_yt9TQC8PvMZY8WCcQ) (part of Snowfork's [improvement roadmap](https://spinamp.mypinata.cloud/ipfs/QmZT1Q9QF45286CxM8qkDuGCr1VCkHQ1VCjFEe89khprH5))
- MMRs: fixed [security vulnerability](https://github.com/nervosnetwork/merkle-mountain-range/commit/8e333a23294008bb48c729581f229bb59cf4f95c) & implemented [ancestry proofs](https://github.com/paritytech/merkle-mountain-range/pull/1)
- MMB PoC implementation in Clojure: [github.com:w3f/merkle-mountain-belt-clj](https://github.com/w3f/merkle-mountain-belt-clj)
- MMB presentation: [Sub0 2022: BEEFY: How Make Proofs Smol?](https://www.youtube.com/watch?v=uCwXbD9v3ew)
<!--- - Early work on [Sunny King style PoS](https://github.com/gridcoin-community/Gridcoin-Research) --->

![favicon@4x](https://hackmd.io/_uploads/Bya9lnvmyl.png =15%x)

### Cryp GmbH
- R&D consultancy founded 2019 in Biel, Switzerland
- Specialises on cross-chain protocols
- History of consulting for variety of both local (e.g. Web3 Foundation & SwissQuote) and international clients (e.g. Parity & Tari Labs)
- Researched & implemented [BTC⇆XMR atomic swaps](https://github.com/farcaster-project) (successfully completed in Q1 2023 from Monero community's [CCS crowdfund grant](https://ccs.getmonero.org/proposals/h4sh3d-atomic-swap-implementation.html)). This has spawned a broad ecosystem for atomic swaps with Monero, such as [ETH⇆XMR](https://github.com/AthanorLabs/atomic-swap), [NMC⇆XMR](https://www.namebrow.se/name/d/atomic-trophy/), and various other pairs via [BasicSwap](https://github.com/tecnovert/basicswap/tree/master).

[^hyperbridge-operational-cost]: For Hyperbridge, MMBs will also reduce the operational cost of the bridge, since consensus update proofs already prove membership of the last leaf in the MMR root commitment: https://github.com/polytope-labs/hyperbridge/blob/8e129796b4b7b0b8e7612625cc4d2592b0818b3b/evm/src/consensus/BeefyV1.sol#L167, https://github.com/polytope-labs/hyperbridge/blob/8e129796b4b7b0b8e7612625cc4d2592b0818b3b/evm/src/consensus/ZkBeefy.sol#L151

[^comparison]: For recent items, MMBs produce a membership proof that is on average at most one hash bigger than the corresponding proof with a hash chain.

[^n-sampling-range]: 10 blocks per minute: `10*60*24*365*2=10512000`, `10*60*24*365*7=36792000`

<!---
[^increment-proofs]: Increment proofs (also known as ancestry proofs) are used to prove that the current state of the commitment scheme (such as an MMR or MMB) descends from a prior canonical state, i.e., it was updated only with appends (and no deletions). They are for instance used in the slashing protocol of the bridge: https://github.com/paritytech/polkadot-sdk/pull/1903
--->

[^relayer-frequency]: Relayers frequently prove inclusion of a message in a parachain header. In fact on Snowbridge, all messages on a particular channel have to be processed [sequentially](https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/Gateway.sol#L141-L149). Hence if relayers are working reliably, then only recent messages are unprocessed at any given time. 

[^mmr-leaf-content]: The MMR leaf in BEEFY contains various metadata such as the parent of the relay chain block, descriptors of the `nextAuthoritySet`, and the leaf version. The core data it commits to though is the root of all parachain headers: https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/BeefyClient.sol#L107-L123. How this `parachainHeadsRoot` is used for verifying cross-chain messages varies between bridge designs:  
When Snowbridge's Ethereum client receives an inbound cross-chain message, it checks that the message [is committed to in a parachain header](https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/Verification.sol#L108-L111), and that said parachain header is in turn [committed to in the BEEFY MMR root stored on-chain](https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/Verification.sol#L125-L128).
When Hyperbridge's Ethereum client receives inbound cross-chain messages (`handlePostRequests`), they are checked for inclusion against a separate MMR root that commits to all ISMP requests and responses: https://github.com/polytope-labs/hyperbridge/blob/8e129796b4b7b0b8e7612625cc4d2592b0818b3b/evm/src/modules/HandlerV1.sol#L151-L157.

[^treasury]: We support the idea of a subsidy from Treasury, especially at the beginning of a bridge operation, to help the bridge gain users. Yet in the long term, what will make the bridge achieve dominance in the market is having an intrinsically superior technology that minimizes both user & operational costs - this is what our MMB upgrade offers.

[^randao]: After a new MMR/MMB root is relayed via [`submitInitial`](https://github.com/Snowfork/snowbridge/blob/main/contracts/src/BeefyClient.sol#L250-L301) on Snowbridge, there is a mandatory delay of at least 3 epochs, i.e. 19 minutes, before the BEEFY light client's storage is updated with the new root via [`submitFinal`](https://github.com/Snowfork/snowbridge/blob/main/contracts/src/BeefyClient.sol#L343-L391) due to [RanDAO biasability](https://eth2book.info/altair/part3/config/preset/#max_seed_lookahead). Hence, the overall latency of a message sent 15 minutes prior to the root update being initiated is over 30 minutes. 

[^glassnode-maximum-range]: This is the maximum averaging range on Glassnode.

[^audit-cost]: Based on our current MMB implementation in Clojure, we have received a quote from Oak Security of 74'800 USD for auditing the Rust library. We reckon Oak Security is in an ideal position to audit the MMB implementation since they have already audited [various iterations of Snowbridge](https://github.com/oak-security/audit-reports/tree/main/Snowbridge) and are therefore very familiar with the usage context.

[^success-reward-calculation-method]: At a high-level, for the case of Snowbridge, we can calculate the cost saving as follows: `gasSaving = gasCostPerProofItem * (proofItemsMMR(leafPosition, accumulatorSize) - proofItemsMMB)`, where `proofItemsMMR(leafPosition, accumulatorSize)` can be calculated as in [here](https://github.com/nervosnetwork/merkle-mountain-range/compare/master...Lederstrumpf:merkle-mountain-range:mmr_profile_proof_size), the `accumulatorSize` can be retrieved from [`BeefyClient.latestBeefyBlock`](https://github.com/Snowfork/snowbridge/blob/77b4c04e3663e0fabfb66b0a5c261e271fe9d2ec/contracts/src/BeefyClient.sol#L156-L157), the leafPosition can be determined from [`MMRLeaf.parentNumber`](https://github.com/Snowfork/snowbridge/blob/77b4c04e3663e0fabfb66b0a5c261e271fe9d2ec/contracts/src/BeefyClient.sol#L111-L112), and `proofItemsMMB` will just be the length of the MMB [`leafProof`](https://github.com/Snowfork/snowbridge/blob/77b4c04e3663e0fabfb66b0a5c261e271fe9d2ec/contracts/src/BeefyClient.sol#L348) once MMBs are deployed.

[^discussions-on-current-cost]: The current cost of DOT→ETH bridging -- even with the recent decrease to ~6 DOT per transfer -- has been raised in the community as a growth painpoint for growth of Snowbridge usage. See for instance related discussions here: [1](https://polkadot.subsquare.io/referenda/1127), [2](https://x.com/Erwin_Schroedy/status/1831266621614145603), [3](https://x.com/ClaraVanStaden/status/1831426063551066292)

[^original-proposal]: Links: [V1 of proposal](https://hackmd.io/@MerkleMountainBelts/MMB-Funding-Proposal) and [version tracking for changes](https://github.com/Cryp-GmbH/mmb-proposal/compare/original-proposal...main) since [original post](https://polkadot.polkassembly.io/post/2392)

[^maintenance-scope]: Maintenance of the MMB library at least includes:   1. addressing bug reports, 2. addressing security issues of the library - in particular any coordinated vulnerability disclosures, and 3. ensuring the MMB library API remains compatible with Polkadot/JAM.
