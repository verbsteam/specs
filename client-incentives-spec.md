# Client Incentives V1 spec

The following protocol functions will earn the facilitating clients rewards:

1. Creating proposals
   1. `propose`
   2. `proposeBySigs`
2. Voting on proposals
   1. `castRefundableVote`
   2. `castRefundableVoteWithReason`
3. Bidding on auctions
   1. `createBid`

All of the functions mentioned above will have a `clientId` parameter which identifies the facilitating client.

## Clients IDs

- Anyone can register themselves as a client and get a clientId.
- To register, one must mint a NounsClient NFT.
- Whoever holds that NFT will have permission to claim rewards for that client.
- The NFT owner may optionally associate a URL with their NFT to identify themselves.
- The NFT art is undetermined at this stage, but will have the option to be updated after launch.

We chose to use clientId instead of simply wallet addresses for a few reasons:

- Requires less bits for storage
- Allows the client to change their managing address without creating a new client entity

## Rewards

All rewards will be calculated as a % of the DAO’s auction revenue.

The benefits we see in using a % of revenue are:

- Incentive alignment between clients and the dao for auction revenue to increase
- Easier to budget a % of income than pay-per-use which is less predictable

### Bidding on auctions (only winning bid)

The client which facilitates the _winning_ bid of an auction, is entitled to a % of that auction’s settled ETH.

The % to use is a parameter adjustable by the DAO.

Example:

If the % is set to be 1%, and the winning bid was 15 ETH, then the client which facilitated the bid can claim 0.15 ETH.

### Creating & voting on proposals

Rewards for proposals are defined as a % of auction revenue in a certain period of time, and split among all the proposals created during that period.

2 parameters are configured:

1. _percentRewardProposals_: % of auction revenue for writing proposals
2. _percentRewardProposalVotes_: % of auction revenue for votes

To ensure the rewards aren’t skewed by too little data when calculating rewards, we require the time period to be either:

- At least 1 proposal and at least 2 weeks long (configurable param)

  OR

- Have at least 10 (configurable param) proposals created in that period

When calling the contract to calculate rewards, the caller passes the last proposal id to reward which is used to mark the end of the period. The start of the period is marked by the proposal id passed in the previous invocation. In both cases, the proposal creation time is used.

To not incentivize spam proposals, a proposal is eligible for rewards only if:

- Voting has ended, and at least 10% (configurable param) of total supply voted For
  - This is separate from the DAO’s quorum parameters. The separation is mostly to avoid gas costs of reading the dynamic-quorum parameters for each proposal.

Rewards for voting are split between all the clients which facilitated voting, weighted by the numbers of votes cast.

Example:

- During a 2 week period, the auction revenue was 210 ETH.
- _percentRewardProposals_ is set to 2% = 4.2 ETH
- _percentRewardProposalVotes_ is set to 1% = 2.1 ETH

During that period 3 proposals were created:

1. Proposal id: 100
   1. Created via clientId 1
   2. Got 20 votes via clientId 1, 30 votes via clientId 2, and 40 votes without clientId
2. Proposal id: 101
   1. Created via clientId 2
   2. Got 30 votes via clientId 1
3. Proposal id 102:
   1. Received less than 10% of supply For votes -> not eligible for rewards

The 4.2 ETH reward is split between proposals 100 & 101, 2.1 ETH each.  
Proposal 100 rewards clientId 1 with 2.1 ETH, and proposal 101 rewards clientId 2 with 2.1 ETH.
The 2.1 ETH reward for votes is split among the eligible proposals based on the amount of votes.

There were a total of 120 votes cast in proposals 100 & 101:

- clientId 1 cast 50 (41.6%) votes and receives 0.87 ETH
- clientId 2 cast 30 (25%) votes and receives 0.525 ETH
- 40 votes cast without a clientId and no reward is allocated for them

### Configurable params

- _auctionRewardBps_: bips of auction revenue rewarding auction bidding
- _proposalsRewardBps_: bips of auction revenue for rewarding writing proposals
- _proposalVotesRewardBps_: bips of auction revenue for rewarding votes
- _proposalsRewardMinimumTimePeriod_: time periods longer than this are sufficient for calculating proposals rewards
- _proposalsRewardMinimumNumProposals_: this number of proposals is sufficient for calculating proposals rewards
- _minimumProposalQuorumForRewardBps_: bips of For votes out of total supply required for a proposal to be eligible for rewards

## Technical design

A side contract called _NounsClientIncentives_ will be responsible for distributing rewards.

In order to reduce risk, the contract will not have access to the main treasury. Instead, the DAO will fund this contract with WETH via proposal. This also has the benefit of easier upgrades, since we don’t need to do big audits for high risk changes. If this becomes a core part of the protocol, we may consider tighter integration in the future.

### Required additions to core contracts

#### NounsAuctionHouse

1. The auction house contract will be upgraded to [NounsAuctionHouseV2](https://github.com/nounsDAO/nouns-monorepo/pull/799) which maintains a historic price feed in the contract’s storage.
   1. This will enable _NounsClientIncentives_ to query the auction revenue during a specific period or a specific auction
2. _getClientIdOfWinningBid(uint256 nounId)_:

- Returns the clientId of the winning bid for a specific nounId
- The clientId will need to be added to the historic price feed saved in storage

##### Gas cost differences

<table>
  <tr>
   <td>
   </td>
   <td>NounsAuctionHouse
   </td>
   <td>NounsAuctionHouseV2
   </td>
   <td>NounsAuctionHouseV2 saving clientId
   </td>
  </tr>
  <tr>
   <td>createBid
   </td>
   <td>81,452
   </td>
   <td>34,919
   </td>
   <td>Should be same as NounsAuctionHouseV2 assuming we pack it with existing storage vars
   </td>
  </tr>
  <tr>
   <td>settleCurrentAndCreateNewAuction
   </td>
   <td>218,092
   </td>
   <td>207,740 if slots used for historic data were warmed up
   </td>
   <td>Approximately 212,740 assuming slots are warmed up
   </td>
  </tr>
</table>

#### NounsDAOLogicV3

Several new data points will saved in the _Proposal_ struct:

1. _createdClientId_: the clientId which facilitated _propose _/ _proposeBySigs_
2. \_votesPerClientId: \_a mapping which will hold the number of votes cast by each client on a specific proposal
3. _createdTimestamp_: timestamp of the block in which the proposal was created

##### Gas cost differences

- propose / proposeBySigs
  - No additional gas cost to save clientId assuming it’s packed with existing storage variables written at the same time.
- updateProposal / updateProposalDescription / updateProposalTransactions / updateProposalBySigs
  - +5K gas to save the lastUpdatedClientId.
- castRefundableVote(WithReason)
  - +20K gas for the first time voting from a specific clientId, +5K gas when the clientId has already been used to vote for this proposal.

### NounsClientIncentives contract

The contract will use WETH by default to pay clients. We decided to use WETH in order to have the flexibility to switch to Lido’s stETH token for example.

_setERC20Address(address token)_

- Changes the ERC20 contract used for withdrawing balances
- Only owner (nouns dao) can call this
- Assumes the token is pegged 1:1 to ETH and has 18 decimals

_withdraw(address token, uint256 amount, address to)_

- Allow the owner (nouns dao) to withdraw `token` tokens from the contract

_withdrawClientBalance(uint32 clientId, uint256 amount, address to)_

- Withdraw token balance and decrease the balance of clientId
- Can be called only by the owner of NFT with id = clientId
- Can be paused & unpaused by an admin account (e.g. verbs team)

_updateRewardsForAuctions(uint256 lastNounId)_

- Updates the balance of clientIds which facilitated auctions since the last invocation up to `lastNounId`
- `lastNounId` must be a settled auction
- Can be paused & unpaused by an admin account (e.g. verbs team)
- Gas used for calling this function will be refunded. Because calling this function can benefit multiple clients, the gas is refunded to not make one client take the cost.
  - If the contract’s balance is empty, gas will not be refunded.

_fastforwardLastAuctionId(uint256 nounId)_

- Can be called by an admin account (e.g. verbs team)
- Advances the last auction nounId to `nounId`
- This is used in case many auctions have been been settled without a clientId, e.g. while auctions are still on nouns.wtf
- `nounId` must be larger than the currently registered `lastAuctionId`
- `nounId` must be a settled auction

_updateRewardsForProposalWritingAndVoting(uint256 lastProposalId)_

- Updates the rewards for all unprocessed proposals up to and including _lastProposalId_
- All proposals included must be done with voting
- Can be paused & unpaused by an admin account (e.g. verbs team)
- Gas used for calling this function will be refunded. Because calling this function can benefit multiple clients, the gas is refunded to not make one client take the cost.
  - If the contract’s balance is empty, gas will not be refunded.

### NounsClient contract

An ERC721 used to identify a client.

_registerClient()_

- Mints a new NounsClient
- The id of the NFT is used as the clientId
- ID space is limited to uint32 in order to reduce storage space required in the core DAO contracts, but still large enough that it is unreasonable that it will be entirely consumed.

_addClientDescription(uint32 clientId, string description)_

- Set a description for the client, e.g. a website
- Only allowed by the owner of NFT with clientId
