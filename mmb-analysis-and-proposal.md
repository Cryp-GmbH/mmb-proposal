# Merkle Mountain Belts: Cost saving analysis + funding proposal 
![Groupe Logo noir@4x](https://hackmd.io/_uploads/Hkdh8Idw0.png =10%x)
#### Cryp GmbH
##### Alfonso Cevallos, Robert Hambrock

## Table of Contents

0. [Intro](#Intro)
1. [MMB overview](#MMB-overview)
    a. [Technical features](#Technical-features)
    b. [Faster append times](#Faster-update-times)
    c. [Shorter membership proofs](#Shorter-membership-proofs)
2. [Cost saving analysis against MMRs](#MMBs-in-BEEFY)
    a. [Cost savings per transaction](#Cost-Estimate)
    b. [Cost savings in overall bridge traffic](#Traffic-Estimate)
3. [How the MMB data structure looks like](#How-the-MMB-data-structure-looks-like)
    a. [U-MMB](#Unbagged-MMB-U-MMB)
    b. [F-MMB](#MMB-with-forward-bagging-F-MMB)
    c. [MMB](#MMB-with-double-bagging-MMB)
4. [Research & Implementation Timeframes](#Timeframes)
    a. [Research Timeframe](#Research-Timeframe)
    b. [Implementation Timeframe](#Implementation-Timeframe)
5. [Success Reward Milestones](#Success-Reward)
6. [Cost of Proposal](#Cost-of-Proposal)
7. [Team](#Team)

<!---
    d. [Comparison of MMB flavors](#Comparison-of-MMB-flavors)
--->

## Intro 

Imagine this: by the beginning of 2026, decentralized applications have gained a significant foothold in the world finances. With them, blockchain platforms such as Polkadot and Ethereum move millions of dollars worth of value every day, and a successful DOT-ETH bridge is one of the top five cross-chain bridges in use. To realize this future for Polkadot, the time is NOW to build the most efficient and secure cross-chain bridge out there. Yet this will not happen if the related bridging costs are as high as they are now.

We propose developing a brand new data structure called Merkle Mountain Belt (MMB), namely both its theoretical and implementation sides, that would replace the use of MMRs in the BEEFY protocol used by bridges like Snowbridge and Hyperbridge. This change will considerably reduce the size of cross-chain message proofs in the bridge[^hyperbridge-operational-cost], in turn reducing the transaction costs of users, with estimated savings of one to two million dollars [per year](#Traffic-Estimate), at current prices.

We also propose writing a research paper about the new data structure, and publishing it in a top rated computer science conference. Polkadot has always been a thought leader in blockchain technology, being one of the few communities that pioneers top quality research, and not just bells and whistles. This proposal honors this tradition. MMRs are currently widely used across multiple blockchains, and improving upon them with MMBs would heighten Polkadot's impact on and reputation within the wider ecosystem.

The members of our [team](#Team) are highly qualified and have worked in Polkadot since pre-mainnet days. Our principal researcher Alfonso has a Ph.D. in mathematics and is a co-author of many Polkadot papers, including the [2020 overview paper](https://arxiv.org/abs/2005.13456). Our principal developer Robert has been one of the earliest stewards of the Polkadot-Ethereum bridge, going back to its initial W3F grant, design of protocols, and implementation.

## MMB overview

:::info
**TLDR of MMBs & cost analysis:**
Merkle Mountain Belts (MMBs) are a data structure that can serve as a plug-in replacement for Merkle Mountain Ranges (MMRs), to produce BEEFY commitments, which are currently used by the DOT-ETH bridges.
Over the next 5 years, MMBs would save [over 40% of proof size](https://docs.google.com/spreadsheets/d/1CIANkTytd6qVo_oc_r2EYzdschoACzgyIa5OiMmPy0I/edit?gid=174519946#gid=174519946) against the current MMR implementation. From our gas and cost estimates, this corresponds to $1.25 per transaction.
<!---If Polkadot's BEEFY bridges have comparable volume--->
:::

*See also the following presentation giving an MMB overview:*
[Sub0 2022: BEEFY: How Make Proofs Smol?](https://www.youtube.com/watch?v=uCwXbD9v3ew)

Balanced Merkle trees, hash chains, and Merkle Mountain Ranges (MMRs) are three examples of data structures widely used in blockchain as *commitment schemes*: they allow full nodes to produce a compact *commitment* to some database $X$, and then produce a small *membership proof* (a.k.a. Merkle proof) to show to a *verifier* that an item $x\in X$ is part of the committed database. 

These commitment schemes are used everywhere in blockchain and allow for external entities to query specific data efficiently, without having to follow consensus, download the full blockchain state, nor trust any one full node. The data structures mentioned above have different properties, and one or other will be chosen depending on the specific use case. The right choice of data structure can drastically reduce the running costs of the corresponding verifier. 

The Merkle Mountain Belt is a new commitment scheme in the same family of Merkle structures, optimized specifically for the use case of BEEFY, the protocol designed for efficient DOT-ETH bridging. Via BEEFY, a smart contract on Ethereum periodically receives a digest on the current state of Polkadot, where the refresh rate is in the order of minutes to a couple hours, and verifiers are Ethereum users who typically query events as soon as a digest committing to them is made available.

To oversimplify, we can say that if most queries from verifiers are strictly for data from the latest block, then the most efficient commitment scheme would be a hash chain, while if queries tend to be for very old data, the most efficient scheme would be an MMR. The MMB is a "best of both worlds" structure whose performance is always comparable to the better of the two schemes, for any possible sampling distribution, but crucially, it outperforms them both in the case that queries are concentrated on events that took place in the time range from the last minute to the last day.[^comparison]

### Technical features

Consider an item list $X=(x_1, x_2, \cdots, x_n)$ onto which new items are constantly being appended. This could be one of many databases, such as XCMP messages, but in our primary use case items correspond to Polkadot relay chain blocks[^mmr-leaf-content]. MMB is a data structure built on top of $X$ that produces a compact commitment to it. Its features are: 

* faster updates after each append (specifically: constant time append operations), and
* shorter membership proofs for recently appended items.


|  | Chain | MMR | MMB |
| -------- | -------- | -------- | --- |
| Update time per append | $O(1)$ | $O(\log n)$ | $O(1)$ | 
| Proof size of $k$-th newest item | $O(k)$ | $O(\log n)$ | $O(\log k)$ |

In this table, we assume $1\leq k\leq n$, where $n:=|X|$ is the current number of items in list $X$, and $k$ is the recency of the item for which we need a membership proof. In our use case $k$ is typically low, say between $10$ and $500$, because we mostly care about proving stuff about recent events. Also keep in mind that Polkadot  currently has north of $20$ million blocks produced, so we are dealing with a use case where $n$ quickly reaches the order of tens of millions.
<!---For the sake of comparison, in the following analysis we take $n=2^{24}$ as a canonical value.--->
For the sake of comparison, in the following analysis, we average $n$ over the first 5 years after anticipated MMB deployment within BEEFY on Polkadot, that is $10512000\leq n \leq 36792000$[^n-sampling-range].

### Faster update times

The main advantage of a hash chain is that it takes a single hash computation to append a new item. On the other hand, MMR and any other structure that attempts to stay balanced will pay an update cost of $O(\log n)$ per append, as the new leaf is immediately buried deep. In our use case, MMR requires on average about $\frac{1}{2}\log_2 n \approx 12$ hash computations (and growing) per append. 

Surprisingly, MMB achieves a **constant** cost per append: between 3 and 5 hash computations regardless of the value of $n$. To do so, it maintains a slightly unbalanced structure, where recent leaves are more shallow, though not completely unbalanced as with a hash chain.

In our use case, having a faster append time is important for block authors, who have to perfom the update on-chain and on the fly as they generate blocks. In other use cases that, e.g., employ ZK proofs, having constant-complexity operations is a hard constraint, so they might not be able to use MMRs with acceptable cost, but they can use MMBs.

### Shorter membership proofs

However, the killer feature of MMB for our use case is its slight unbalancedness, which makes recently appended items have shorter membership proofs. This leads to lower operational costs because bridge users are much more likely to care about recent events[^relayer-frequency]. 

In particular, Ethereum users will use the latest BEEFY commitment stored on the bridge to authenticate a cross-chain message that originated from Polkadot, such as a deposit. They will typically act as soon as there is a such a commitment to their event of interest. Hence, in order to ensure user friendliness, we need two things:

1. Short intervals between commitment updates in the bridge. Popular cross-chain bridges update their commitment every 15 minutes to 30 minutes. This could be reduced to five minutes, or increased to a couple of hours, depending on the chosen trade-off between operational cost and user friendliness. 
2. A structure, like MMB, that minimizes the membership proof size of leaves added in the last five minutes, to the last couple of hours.

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

*Average proof size (in hashes) of an item appended at different recency values (i.e., time from the present), assuming that a new item is appended every 6 seconds. The results are averaged across ($10512000\leq n \leq 36792000$), i.e., the size of the list between 2-7 years of appending.*

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



<a id="MMBs-in-BEEFY"></a>

## Cost savings analysis against MMRs

In an Ethereum bridge, the BEEFY light client on Ethereum stores a commitment to Polkadot's state. Currently, this commitment is an MMR root. Concretely, the MMR root commits to the leaves of the MMR, and in turn the leaves of the MMR contain commitments to parachain headers. As such, from the MMR root, one can prove to Ethereum that some event occurred on Polkadot.

When a message (that, e.g., encodes a transaction) goes over the bridge, the message producer or a relayer proves to the BEEFY light client that this message is authentic by attaching an MMR proof and a header proof to the message:
1. The MMR proof shows that the MMR (whose root is held in the BEEFY light client) commits to a leaf, which in turns commits to a particular parachain header.
2. The header proof shows that this parachain header indeed contains the claimed content, such as a message.

As the MMR grows over time, so grow in size the membership proofs that have to be relayed to the BEEFY light client. In contrast to MMRs, in MMBs the membership proof size is *independent* of the size of the MMB - its asymptotic behavior is instead governed by the recency of the leaf to be proven. Hence if membership proofs are primarily for recent items, then MMBs on average have much smaller membership proofs than MMRs.

<a id="Cost-Estimate"></a>

### Cost savings per transaction

We calculate the expected transaction cost saving from switching from MMRs to MMBs in the first 5 years after we conservatively expect the MMB upgrade to go live, namely Q1 2026. Since this is approximately 2 years after BEEFY got activated on Polkadot (February 2024), we'll average the MMR size over ($10512000\leq n \leq 36792000$), as done before in the [table above](#proof-size-comparison).

To compare both schemes, we consider the membership proof of leaves up until the 150th most recent leaf. This value $k\leq150$ corresponds to querying a message that was included in the Polkadot state up to 15 minutes ago (+ 24 minutes; see note on RanDAO[^randao]). For $k\leq150$, the average MMR proof size (measured in hashes) is 16.65, against 9.95 for MMB proofs, hence MMBs save 6.7 proof items: a proof size reduction of over 40%.

Via gas profiling from Snowfork's implementation, every additional proof item in the membership proof of an MMR leaf adds approximately 2'568.34 gas of computation cost on Ethereum: [`Lederstrumpf/snowbridge/rhmb/profile-proof-item-gas-cost`](https://github.com/Snowfork/snowbridge/compare/main...Lederstrumpf:snowbridge:rhmb/profile-proof-item-gas-cost?expand=1#diff-ad43af2a8fe32a0a552c025fc1d86308adf4ee1e4d45f83070cce458c461c818R85-R87)

Assuming a gas price of 26.99 GWEI (365 day SMA via [Glassnode](https://studio.glassnode.com/metrics?a=ETH&category=&chartStyle=column&ema=0&from_exchange=aggregated&m=fees.GasPriceMean&mAvg=365&mMedian=0&resolution=24h&s=1687097402&to_exchange=aggregated&u=1718633402&utm_campaign=woc_02_2023&utm_medium=insights_woc&utm_source=gn_insights&zoom=365)), and an ETH price of $2'452.41/ETH (365 day SMA via [Glassnode](https://studio.glassnode.com/metrics?a=ETH&category=Market&chartStyle=column&ema=0&from_exchange=aggregated&m=market.PriceUsdClose&mAvg=365&mMedian=0&resolution=24h&s=1687097402&to_exchange=aggregated&u=1718633402&utm_campaign=woc_02_2023&utm_medium=insights_woc&utm_source=gn_insights&zoom=365)), the cost per proof item is 17.5 cents.

Consequently, since an average proof of a leaf in an MMR of depth up to $k\leq150$ is 16.65, expected proof verification cost for such a leaf is $2.91. In contradistinction, with MMBs the expected proof size is 9.95, hence the cost goes down to $1.74. Using an MMB rather than an MMR for the BEEFY commitment thus would effect a saving of 6.7 proof items, or $1.17 per message sent to Ethereum.
<!---committed list of size $n=2^{24}$ (current Kusama/Polkadot block counts),--->

For more calculation details and a 1000-day[^glassnode-maximum-range] SMA comparison, see the [Cost Analysis Spreadsheet](https://docs.google.com/spreadsheets/d/1CIANkTytd6qVo_oc_r2EYzdschoACzgyIa5OiMmPy0I/edit?gid=174519946#gid=174519946).

***Note:** this cost is saved by users of the bridge and - if it plans to subsidize transactions - the Polkadot treasury[^treasury]. This saving makes the Polkadot🡒Ethereum bridge more competitive with other bridges, increasing its utilization, and in turn increasing the aggregate cost saving. Note that these fees would otherwise just be burned on Ethereum, hence would go to no benefit toward the Polkadot ecosystem.*

<a id="Traffic-Estimate"></a>

### Cost savings in overall bridge traffic

To estimate the transaction numbers we can expect for a successful DOT🡘ETH bridge once it is mature, we take the mean transaction numbers of the 5 most utilized bridges (by monthly volume) as a proxy.
<!--- TODO: maybe make the following a footnote to improve flow --->
Our estimate is thus based on the assumption that at least one Polkadot-Ethereum bridge is a success. This is the same reason we chose 15 minutes (+ 24 minutes ranDAO [^randao] for Snowbridge) as a reference latency, given that such latency is offered by competitors like Stargate and Optimism, and even lower for trusted bridging setups.

A comparison (using data from 19 June 2024) is available here: [Cost Analysis Spreadsheet](https://docs.google.com/spreadsheets/d/1CIANkTytd6qVo_oc_r2EYzdschoACzgyIa5OiMmPy0I/edit?gid=174519946#gid=174519946). From it, we expect an average of 3'054.04 transactions per day, or 1'114'724.60 transactions per year.

**Result:** *The estimated cost saving from switching from MMRs to MMBs is in the range of $1'306'000/year - $1'977'000/year.*
<!-- *At a subsidization level of ..., this represents an annual cost saving of ... to the Polkadot treasury.* -->

## How the MMB data structure looks like

### Unbagged MMB (U-MMB)

In what follows we assume familiarity with the MMR data structure, and we build from there. Consider a list $X=(x_1, x_2, \cdots, x_n)$ we want to commit to. As in MMR, we build multiple *mountains*, where each mountain has an associated size $s$ and is a complete binary tree holding $2^s$ leaves. Each leaf is labeled with the hash evaluation of an item in $X$, and each non-leaf is labeled with the hash evaluation of the concatenation of its children. 

![MMB9](https://hackmd.io/_uploads/rkDvrCIDA.png)


This is the MMB structure for $n=9$ (without bagging), with three mountains of sizes $2,2,0$, whose peaks are in gray. Notice that, unlike in MMR, in MMB there may be mountains of the same size. 


We say two mountains of equal size $s$ are *mergeable*, because they could be merged into a mountain of size $s+1$ by creating a new peak node as the parent of the two old peaks. We follow this rule: whenever we append a new item to list $X$, we add it as a mountain of size $0$ (i.e., a singleton), and if there are mergeable pairs of mountains at that point, we merge *exactly one* such pair, namely the rightmost mergeable pair.


![image](https://hackmd.io/_uploads/rkyZbvQvA.png)

For instance, here we see two append updates, with the new leaf in green and the new merge peak in gray, and mountain peaks represented as triangles. When appending $x_{10}$ we merge two mountains of size $0$ into one of size $1$, and when appending $x_{11}$ we merge two mountains of size $2$ into one of size $3$. 

In contrast, MMR follows the rule that any mergeable pair is merged immediately, but the problem is that it can lead to a domino effect of up to $O(\log n)$ merges after a single append. With our lazy merging rule, we make sure that the complexity of each append remains bounded by a constant. And it can be proven that the rule leads to nice properties: namely, the number of mountains remains of order $O(\log n)$, and the $k$-th most recent leaf is at a depth of $O(\log k)$ in its mountain, regardless of the value of $n$.

### MMB with forward bagging (F-MMB)

Next, we *bag* the mountains, which means adding an upper layer of *bagging nodes* -- one per mountain -- where each bagging node is the parent of one mountain peak and the previous bagging node. We do this to group mountains into a single structure -- a "mountain range" -- with a single root whose hash acts as a commitment to the full list $X$.

![image](https://hackmd.io/_uploads/BkbFWaEwC.png)

In the figure we see two possible ways of bagging the same set of mountains: in a backward fashion on the left, and in a forward fashion on the right. In both cases, the root is in brown, peaks in gray, bagging nodes are represented as rhombi, and the symbol $\bullet$ represents a non-existent child node.

Backward bagging is usually used in MMR: it leads to a more balanced structure because larger mountains (on the left) end up closer to the root, so items with a large leaf-to-peak distance are compensated with a short peak-to-root distance. In contrast, forward bagging keeps the structure unbalanced as larger mountains end up farther from the root. But it preserves the property we want: that the $k$-th most recent leaf remains at a depth of $O(\log k)$ regardless of the value of $n$. And the other important advantage of forward bagging is that we don't need to bag mountains again from scratch after each append, as most bagging nodes don't need to be updated.

![image](https://hackmd.io/_uploads/SJO7LpEPC.png)

E.g., here we see a series of append updates, if we use a forward bagging. For simplicity, we hide all non-peak mountain nodes, representing each mountain only by its peak labeled with its mountain size, and we place bagging nodes on a single horizontal level. The new leaf is in green, the merge peak in gray, and we paint in brown the bagging nodes that needed to be updated.

We see that, with forward bagging, most of the time we only need to update a handful of nodes overall after an append, even when the number of mountains is very large, unlike MMR's backward bagging. 

However, the complexity of an append update is not always bounded by a constant, because from time to time we need to merge two large mountains on the far left side of the structure, in which case we update all bagging nodes above and to the right of the merge peak, i.e., almost all $O(\log n)$ bagging nodes. We solve this issue with our final ingredient.

### MMB with double bagging (MMB)

In the final version of MMB, we use neither forward not backward bagging, but we introduce a new type that we call double bagging. Instead of grouping all mountains into a single range, we first partition mountains into subsets, then use forward bagging within each subset to group them into a range, and finally use an extra layer of forward bagging to group all mountain ranges into a *mountain belt*.

![image](https://hackmd.io/_uploads/Bkz6tDBv0.png)

E.g., here is the final MMB structure for $n=1337$. There are $10$ mountains (represented only by their peaks) partitioned into $5$ ranges, each range is forward bagged with rhombi nodes, and finally the $5$ ranges are forward bagged with circle nodes into one belt, whose root is in brown.

The point of the mountain partitioning is to ensure that each mergeable pairs of mountains sits at the right end of a range. Hence, whenever we merge them, we only need to update one bagging node in its range. And we also only need to update one or two bagging nodes in the outer bagging, because the pair we are merging (the rightmost mergeable pair) always sits in the rightmost or second rightmost range.

![MMBs](https://hackmd.io/_uploads/BJLNCPHw0.png)

Here we see a series of append updates in MMB. Again, the new leaf is in green, the merge peak in gray, and we color in brown all bagging nodes that need to be updated. Notice that at most $4$ bagging nodes are updated, even when we merge large mountains.

The full structure will be presented in detail in the research paper, where we will show that we get all the properties we want: an append operation with a complexity bounded by a constant, and the $k$-th most recent leaf at depth $O(\log k)$ regardless of the value of $n$ (i.e. independent of the MMB's size).

<!---
### Comparison of MMB flavors

We highlight that in some scenarios it might be a good idea to implement one of the variants of MMB presented above. 

As U-MMBs omit the bagging step, they are much simpler to implement, and more importantly, they offer shorter membership proofs than MMB (about half the size), which in our use case translates to even smaller transaction costs for bridge users. On the down side, they require changes to the Ethereum light client and increase the data storage cost of the BEEFY commitment (i.e., increase the operational cost of the bridge). This is because the commitment would correpond to the list of hashes of all $O(\log n)$ mountain peaks. Full MMBs on the other hand require no changes to Snowbridge's Ethereum client.

F-MMBs are also simpler to implement than MMBs, because they use a simpler bagging process. Just as MMBs, in our use case they serve as a drop-in replacement to MMRs on the Ethereum light client. However, they have a $O(\log n)$ worst-case update time per append, against costant time for MMBs.  Likewise, F-MMBs have larger increment proofs[^increment-proofs] than MMBs.
--->

<a id="Timeframes"></a>

## Research & Implementation Timeframes
This funding request forks into a research and an implementation track. The key deliverables are
1. a research paper,
2. an analysis of implementation candidates for MMB in addition to BEEFY bridging, and
3. a Rust implemenation of Merkle Mountain Belts integrated as a frame pallet, together with any necessary changes to Snowfork's BEEFY client necessary to accommodate:
https://github.com/Snowfork/snowbridge/blob/main/contracts/src/BeefyClient.sol

The additional success milestones are described in the [success reward section](#Success-Reward). The key deliverables for this track are
1. publication of the MMB research paper in a well-respected journal, and
2. deployment of MMBs in Polkadot's BEEFY protocol.

### Research Timeframe 
---
#### M1. Paper: **18 weeks (162'000 CHF)**
The primary milestone of the research track is completing a paper on MMBs currently in the works. The timeframe for completion of this paper is so short since a lot of the work for the paper has already been completed by the team members in collaboration with Alistair Stewart at Web 3.0 Foundation. Alistair Stewart will also be a co-author on the paper.

This paper will present the novel data structures U-MMBs, F-MMBs, and MMBs. The paper will also compare the MMB flavors with existing data structures like hash chains and two variants of MMRs.
##### Writing budget: 12 weeks
Completion of paper, such as
1. adding missing proofs 
2. adding pseudo-algorithms, and
3. completing the analysis of amortized membership proof sizes.

##### UTXO analysis: 6 weeks

To substantiate our hypothesis that membership proofs for recent items of a committed data base are accessed much more frequently than ancient items, we analyse the lifespan of UTXOs before being spent. In particular, Bitcoin and ZCash will be analysed.
This time budget accounts for both data collection and analysis timeframes.
The analysis will employ a more differential methodology than exhibited here:
https://www.nature.com/articles/s41597-022-01254-0

---
#### M2. Exploration budget: 12 weeks (108'000 CHF)
We add a buffer of twelve weeks to allocate to further investigations. These may end up in the paper too, but this would be subject to, among other factors, the outcome of this research, timing with respect to a suitable conference, and the shape of the paper. If not included in the paper, the findings will nonetheless be made available to the Polkadot community, for instance in the format of research reports.

Here are a few of the areas we will research:
##### XCMP
If, as currently envisioned, XCMP has unordered message delivery, then MMBs are a suitable data structure to use as a drop-in replacement for MMRs. The advantages of MMBs for this use case are: 

- much shorter proofs (and hence smaller operational costs) since, again, membership queries will be made mostly for recent messages; and 
- on top of archival nodes who store full databases, MMBs allow for the network to have lighter nodes who only store the last $k$ messages (for an adecuate value of $k$), and whose running costs depend only on $k$ and not on $n$  (the total number of messages), yet produce the same membership proofs for recent items than archival nodes.

This flavor of MMBs would not be append-only (vis-à-vis for MMRs), but would require an update protocol allowing leaves to be replaced. An update proof replacing the prior item $x$ with item $x'$ could naively be achieved by first providing a membership proof for $x$ relative to the current MMB root $r$ (the known commitment), and then re-using the verified proof items to calculate the new root $r'$ from $x'$.

##### Account-based chain analysis
The UTXO analysis we will include in the paper substantiates our hypothesis, but extending it to account-based chains is more applicable to the DOT🡘ETH bridge.
<!--- TODO: extend --->

<!--- TODO: add
##### POPOS (TODO)
https://arxiv.org/pdf/2209.08673.pdf
--->

##### Flyclient integration
https://eprint.iacr.org/2019/226.pdf
Flyclient is a good candidate for MMB integration since its sampling protocol is well aligned with the asymmetric membership path length distribution of MMBs: Flyclient predominantly samples recent blocks rather than ancient ones, and MMB path lengths exhibit the same behavior. As such, a Flyclient implementation employing MMBs instead of MMRs would be much more efficient.
<!--- TODO: extend --->

---
### Implementation Timeframe

The current implementation of MMBs is available here: <https://github.com/w3f/merkle-mountain-belt-clj>

We propose developing a Rust library readily integrateable by relevant Rust pallets such as `pallet-beefy` to be used instead of [`nervosnetwork:merkle-mountain-range`](https://github.com/nervosnetwork/merkle-mountain-range). The library will be licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0.txt). Our time estimate for the implementation of a production-ready MMB library on top of the existing MMR library from Nervos is 41 FTE weeks (i.e. about 9 months) of engineering:
<!---While Nervos' MMR library has a few...--->

#### M3. Finish PoC in Clojure: **6 weeks (54'000 CHF)**
   1. implement ancestry proofs: 3 weeks
      *for instance, slashing of cross-chain equivocations requires ancestry proofs*
   2. refactor implementation to facilitate specification process: 1 week
   3. profiling and performance improvements: 2 weeks
      *the Clojure implementation currently exhibits worse asymptotic behavior than we know is theoretically possible. Profiling and performance improvements here are restricted to only cover improvements that will also reflect in the specification and Rust implementation of the library.*
#### M4. Spec MMB implementation for porting to Rust: **2 weeks (18'000 CHF)**
   *In particular, this will cover interfaces required for integrations with other pallets (e.g. `pallet-beefy`) as currently provided by `pallet-mmr`*
#### M5. Rust implementation: unbagged MMBs: **18 weeks (162'000 CHF)** 
   1. data storage: 4 weeks
   2. U-MMBs: 12 weeks
   3. membership & ancestry proofs (for U-MMBs only): 2 weeks
#### M6. Rust implementation: bagged MMBs: **15 weeks (135'000 CHF)**
   1. forward-bagged MMBs: 3 weeks
   2. double-bagged MMBs: 8 weeks
   3. membership & ancestry proofs (F-MMBs and double-bagged MMBs): 4 weeks
   
## Success Reward
The milestones we request funding for above all constitute deliverables that do not depend on externalities, and where the scope of the work is thus more straightforward to estimate. 

However, even if our paper is technically sound, acceptance of the paper by the first conference we submit it to is all but guaranteed. Likewise, the upgrade of MMRs to MMBs within BEEFY on Polkadot requires collaboration with multiple other stakeholders, and thus estimating the required work on our part is unreliable.

In particular, for these milestones, we reckon a long-term incentivization for completing these success metrics is an appropriate alignment.
We ask for 45'000 CHF fixed fee for completing these milestones, as well as a 30% share of the provable gas savings incurred by our implementation of MMBs against the gas cost that would have been incurred by remaining with MMRs, to be rewarded over the first 10 years post-deployment of MMBs on Polkadot. If the community prefers, we can also reduce the savings share for a perpetual fee - we are happy to receive input on this. In either scenario, if MMBs get replaced by a distinct technology in the future, the fee share would of course terminate.

We will submit new referenda for paying out these rewards every year, calculated from the total number of Polkadot cross-chain transactions that saved from these fees, and the aggregate gas cost of membership proofs saved by MMBs in these transactions.

### M7. Conference paper review & presentation
#### Conference
<a id="publication-location"></a>
In addition to the open-access [arXiv](https://arxiv.org/), we plan to submit the paper for the proceedings of a suitable conference. Choice of conference will be made once the paper is sufficiently close to completion that we can submit it by the proceedings deadline without sacrificing scope & quality of the paper.
For instance, *Advances in Financial Technologies* or *Financial Cryptography* would be fine candidates for submission.
The time budget here accounts for preparing the work to be presented at the conference as well as travel & presentation.

#### Review feedback etc.
Review and addressal of the paper's submission feedback. This milestone may be delayed if anyone provides a solid argument against the technical soundness of the paper. 

### M8. Polkadot deployment
Once the library is fully developed and ready for use in BEEFY, it will still have to be deployed. This intense process will not only require our continuous involvement, but also close collaboration with the Technical Fellowship, and for some steps also the Snowfork and/or the Hyperbridge team.

Likewise, the code will have to be audited before deployment on Polkadot. This would either be covered by a separate proposal or via e.g. an SRLabs retainer.

Since these upcoming steps involve multiple moving parts, it is hard to provide a reliable time estimate for completing these.

This work for this milestone is deemed complete once MMBs are deployed within BEEFY on Polkadot and used by at least one bridge, such as Snowfork or Hyperbridge.
   
## Cost of Proposal
Exclusive of the fixed portion of the [success reward](#Success-Reward), for the 30 weeks of research, we request 270'000 CHF, and for the 41 weeks of implementation, we request 369'000 CHF, totalling 639'000 CHF.  

The total DOT allocation will be based on the [Swiss National Bank USD/CHF foreign exchange rate](https://www.snb.ch/en/the-snb/mandates-goals/statistics/statistics-pub/current_interest_exchange_rates), and the 30-day EMA of USD/DOT via [Subscan](https://polkadot.subscan.io/tools/price_converter) onat the date of submission.

The 45'000 CHF fixed portion of the [success reward](#Success-Reward) will be claimed retroactively once the paper has been published in conference proceedings and MMBs have been deployed within BEEFY on Polkadot. The variable success rewards will be separately requested annually.

We currently reckon the best fit for this proposal is for OpenGov to fund the research milestones and the success reward, and for Technical Fellowship Committee to fund the implementation milestones, given that the Technical Fellowship stewards the Polkadot runtime and any upgrades to it.
We are open to input on this suggested funding split.

## Team
### Alfonso Cevallos
- Mathematics PhD from EPFL (2016). Postdoc at ETH Zurich, Switzerland (2017-2018).
- Researcher at Web3 Foundation (2019-2024): 
focus on scalability, security and incentives of decentralized protocols.
- Co-author of the [2020 Polkadot implementation paper](https://arxiv.org/abs/2005.13456).
- [Co-designer](https://www.youtube.com/watch?v=OdbCgdQugKM) and [paper co-author](https://arxiv.org/abs/2004.12990) of Polkadot's Nominated Proof-of-Stake (NPoS). 
- [Paper co-author](https://arxiv.org/abs/2312.11408) on study about real-life security metrics in Polkadot validator elections. 
- [Paper co-author](https://eprint.iacr.org/2024/961) of ELVES block auditing, the basis of Polkadot's scalability. 
- Content creator and lecturer in several editions of the Polkadot Blockchain Academy. 

### Robert Hambrock
- Honours postgrad in Physics from UCT, South Africa (2016, with distinction). Worked primarily on [AdS/CFT correspondence](https://scholar.google.com/citations?user=rluJ3bAAAAAJ&hl=en&oi=ao). Visiting scientist at CERN 2017, 2018.
- Ethereum CBC Casper: co-author of [Rust implemenation](https://gitlab.com/TrueLevel/casper/core-cbc)
- switched over to primarily working on Polkadot ecosystem in Q1 2020 (also operate validator [🚂 Zugian Duck 🦆](https://insights.math-crypto.com/polkadot/15jrQX54HczCKJgtYYoKvzJ2kgyCyyA4kyvMv2bC8x9UtDpn/))
- at Web3 Foundation (2020-2022) as Quality Assurance Engineer and then Research Engineer, later at Parity (2022-2024) as Core Engineer on bridge team.
- ETH bridge: various work over the years, e.g. on coordination of Snowfork's initial w3f grant, work on [subsampling protocol](https://hackmd.io/ohOt4jAPT8uu-soJXHUq0Q), [cross-chain slashing implementation](https://github.com/paritytech/polkadot-sdk/pull/1903), [session skipping](https://github.com/Snowfork/snowbridge/pull/1215), and [fallback governance proposal](https://hackmd.io/MlVt_yt9TQC8PvMZY8WCcQ) (part of Snowfork's [improvement roadmap](https://spinamp.mypinata.cloud/ipfs/QmZT1Q9QF45286CxM8qkDuGCr1VCkHQ1VCjFEe89khprH5))
- MMRs: fixed [security vulnerability](https://github.com/nervosnetwork/merkle-mountain-range/commit/8e333a23294008bb48c729581f229bb59cf4f95c) & implemented [ancestry proofs](https://github.com/paritytech/merkle-mountain-range/pull/1)
- MMB PoC implementation in Clojure: [github.com:w3f/merkle-mountain-belt-clj](https://github.com/w3f/merkle-mountain-belt-clj)
- MMB presentation: [Sub0 2022: BEEFY: How Make Proofs Smol?](https://www.youtube.com/watch?v=uCwXbD9v3ew)
<!--- - Early work on [Sunny King style PoS](https://github.com/gridcoin-community/Gridcoin-Research) --->
### Cryp GmbH
- R&D consultancy founded 2019 in Biel, Switzerland
- Specialises on cross-chain protocols
- History of consulting for variety of both local (e.g. Web 3.0 Foundation & SwissQuote) and international clients (e.g. Parity & Tari Labs)
- Researched & implemented [BTC⇆XMR atomic swaps](https://github.com/farcaster-project) (successfully completed in Q1 2023 from Monero community's [CCS crowdfund grant](https://ccs.getmonero.org/proposals/h4sh3d-atomic-swap-implementation.html)). This has spawned a broad ecosystem for atomic swaps with Monero, such as [ETH⇆XMR](https://github.com/AthanorLabs/atomic-swap), [NMC⇆XMR](https://www.namebrow.se/name/d/atomic-trophy/), and various other pairs via [BasicSwap](https://github.com/tecnovert/basicswap/tree/master).

[^hyperbridge-operational-cost]: For Hyperbridge, MMBs will also reduce the operational cost of the bridge, since consensus update proofs already prove membership of the last leaf in the MMR root commitment: https://github.com/polytope-labs/hyperbridge/blob/8e129796b4b7b0b8e7612625cc4d2592b0818b3b/evm/src/consensus/BeefyV1.sol#L167, https://github.com/polytope-labs/hyperbridge/blob/8e129796b4b7b0b8e7612625cc4d2592b0818b3b/evm/src/consensus/ZkBeefy.sol#L151

[^comparison]: For recent items, MMBs produce a membership proof that is on average at most one hash bigger than the corresponding proof with a hash chain.

[^n-sampling-range]: 10 blocks per minute: `10*60*24*365*2=10512000`, `10*60*24*365*7=36792000`

[^increment-proofs]: Increment proofs (also known as ancestry proofs) are used to prove that the current state of the commitment scheme (such as an MMR or MMB) descends from a prior canonical state, i.e., it was updated only with appends (and no deletions). They are for instance used in the slashing protocol of the bridge: https://github.com/paritytech/polkadot-sdk/pull/1903

[^relayer-frequency]: Relayers frequently prove inclusion of a message in a parachain header. In fact on Snowbridge, all messages on a particular channel have to be processed [sequentially](https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/Gateway.sol#L141-L149). Hence if relayers are working reliably, then only recent messages are unprocessed at any given time. 

[^mmr-leaf-content]: The MMR leaf in BEEFY contains various metadata such as the parent of the relay chain block, descriptors of the `nextAuthoritySet`, and the leaf version. The core data it commits to though is the root of all parachain headers: https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/BeefyClient.sol#L107-L123. How this `parachainHeadsRoot` is used for verifying cross-chain messages varies between bridge designs:  
When Snowbridge's Ethereum client receives an inbound cross-chain message, it checks that the message [is committed to in a parachain header](https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/Verification.sol#L108-L111), and that said parachain header is in turn [committed to in the BEEFY MMR root stored on-chain](https://github.com/Snowfork/snowbridge/blob/c8ea65c5b54c8befc40910a367fb0edf3f528b33/contracts/src/Verification.sol#L125-L128).
When Hyperbridge's Ethereum client receives inbound cross-chain messages (`handlePostRequests`), they are checked for inclusion against a separate MMR root that commits to all ISMP requests and responses: https://github.com/polytope-labs/hyperbridge/blob/8e129796b4b7b0b8e7612625cc4d2592b0818b3b/evm/src/modules/HandlerV1.sol#L151-L157.

[^treasury]: We support the idea of a subsidy from Treasury, especially at the beginning of a bridge operation, to help the bridge gain users. Yet in the long term, what will make the bridge achieve dominance in the market is having an intrinsically superior technology that minimizes both user & operational costs - this is what our MMB upgrade offers.

[^randao]: After a new MMR/MMB root is relayed via [`submitInitial`](https://github.com/Snowfork/snowbridge/blob/main/contracts/src/BeefyClient.sol#L250-L301) on Snowbridge, there is a mandatory delay of at least 3 epochs, i.e. 19 minutes, before the BEEFY light client's storage is updated with the new root via [`submitFinal`](https://github.com/Snowfork/snowbridge/blob/main/contracts/src/BeefyClient.sol#L343-L391) due to [RanDAO biasability](https://eth2book.info/altair/part3/config/preset/#max_seed_lookahead). Hence, the overall latency of a message sent 15 minutes prior to the root update being initiated is over 30 minutes. 

[^glassnode-maximum-range]: This is the maximum averaging range on Glassnode.
