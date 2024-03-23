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
- To register, one must mint a Nouns Client NFT.
- For a client to withdraw their rewards they need:
  1.  To hold the client NFT with the relevant ID, and
  2.  For their client ID to be approved by the DAO.
- The NFT owner may optionally associate a name and description with their NFT to identify themselves.
- The client NFT is launching with basic art thanks to krel, and the DAO has the option to update it at a later time.

We chose to use clientId instead of simply wallet addresses for a few reasons:

- Requires less bits for storage.
- Allows the client to change their managing address without creating a new client entity.

ClientID 0 (zero) is reserved and means this client should not be rewarded. Interactions on nouns.wtf for example will be with clientId zero. Actions done by clientId zero will still be competing against other clients, but the rewards are kept in the rewards pool.

## Rewards

All rewards will be calculated as a % of the DAO’s auctions revenue.

The benefits we see in using a % of revenue are:

- Incentive alignment between clients and the dao for auction revenue to increase.
- Easier to budget a % of income than pay-per-use which is less predictable.

### DAO Approval

A client may only claim rewards after the DAO has approved their clientId to withdraw rewards. The DAO approves clients
via a DAO proposal to call `Rewards.setClientApproval`.
DAO approval is required in order to avoid DAO members to collect rewards for their own actions.

### Admin account

An admin wallet is given permission to pause & unpause the contract. This is in case the Rewards contract is somehow abused.

### Bidding on auctions (only winning bid)

The client which facilitates the _winning_ bid of an auction, is entitled to a % (in bips) of that auction’s settled ETH.

The % to use is a parameter adjustable by the DAO.

Example:

If the % is set to be 100 bips, and the winning bid was 15 ETH, then the client which facilitated the bid can claim 0.15 ETH.

This parameter is named `auctionRewardBps` in the contract.

### Creating & voting on proposals

Rewards for proposals are defined as a % of auction revenue in a certain period of time, and split among all the proposals created during that period.

2 parameters are configured:

1. `proposalRewardBps`: bips of auction revenue for creating proposals
2. `votingRewardBps`: bips of auction revenue for casting votes

To ensure the rewards aren’t skewed by too little data when calculating rewards, we require the time period to be either:

- At least 1 proposal and at least 2 weeks long (configurable param: `minimumRewardPeriod`).

  OR

- Have at least 5 (configurable param: `numProposalsEnoughForReward`) proposals created in that period.

When calling the contract to calculate rewards (`updateRewardsForProposalWritingAndVoting`), the caller passes the last proposal id to reward which is used to mark the end of the period. The start of the period is marked by the proposal id passed in the previous invocation. In both cases, the proposal creation time is used.

To not incentivize spam proposals, a proposal is eligible for rewards only if:

- Voting has ended, and at least 10% (configurable param: `proposalEligibilityQuorumBps`) of total supply voted For
  - This is separate from the DAO’s quorum parameters. The separation is mostly to avoid gas costs of reading the dynamic-quorum parameters for each proposal.

Rewards for voting are split between all the clients which facilitated voting, weighted by the numbers of votes cast.

Example:

- During a 2 week period, the auction revenue was 210 ETH.
- `proposalRewardBps` is set to 200 bips = 4.2 ETH
- `votingRewardBps` is set to 100 = 2.1 ETH

During that period 3 proposals were created:

1. Proposal id: 100
   1. Created via clientId 1
   2. Got 20 votes via clientId 1, 30 votes via clientId 2, and 40 votes without clientId (same as clientId zero)
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
- 40 votes cast without a clientId (or clientId zero). Their portion of the reward is kept in the pool.

### Configurable params

- `auctionRewardBps`: bips of auction revenue rewarding auction bidding
- `proposalRewardBps`: bips of auction revenue for rewarding writing proposals
- `votingRewardBps`: bips of auction revenue for rewarding votes
- `minimumRewardPeriod`: time periods longer than this are sufficient for calculating proposals rewards
- `numProposalsEnoughForReward`: this number of proposals is sufficient for calculating proposals rewards
- `proposalEligibilityQuorumBps`: bips of For votes out of total supply required for a proposal to be eligible for rewards

## Technical design

A side contract called `Rewards` will be responsible for distributing rewards.

In order to reduce risk, the contract will not have access to the main treasury. Instead, the DAO will fund this contract with WETH via proposal. This also has the benefit of easier upgrades, since we don’t need to do big audits for high risk changes. If this becomes a core part of the protocol, we may consider tighter integration in the future.

### Required additions to core contracts

#### NounsAuctionHouse

1. The auction house contract will be upgraded to `NounsAuctionHouseV2` which maintains a historic price feed in the contract’s storage.
   1. This will enable `Rewards` to query the auction revenue during a specific period or a specific auction.
   2. An additional `createBid` function will be added with a `clientId` parameter which will also be saved in the historic price data.

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

#### NounsDAOLogic

Several new data points will saved in the _Proposal_ struct:

1. `clientId`: the clientId which facilitated `propose` / `proposeBySigs`.
2. `voteClients`: a mapping which will hold the number of votes cast by each client on a specific proposal.
3. `creationTimestamp`: timestamp of the block in which the proposal was created

##### Gas cost differences

- propose / proposeBySigs
  - No additional gas cost to save clientId assuming it’s packed with existing storage variables written at the same time.
- updateProposal / updateProposalDescription / updateProposalTransactions / updateProposalBySigs
  - +5K gas to save the lastUpdatedClientId.
- castRefundableVote(WithReason)
  - +20K gas for the first time voting from a specific clientId, +5K gas when the clientId has already been used to vote for this proposal.

### Rewards contract

The contract will use WETH by default to pay clients. We decided to use WETH in order to have the flexibility to switch to Lido’s stETH token for example.

#### Public write functions

```solidity
function registerClient(string calldata name, string calldata description) external whenNotPaused returns (uint32)
```

- Mints a new NounsClient NFT. `name` and `description` are used to identify the client.
- The id of the NFT is used as the clientId.
  - ID space is limited to uint32 in order to reduce storage space required in the core DAO contracts, but still large enough that it is unreasonable that it will be entirely consumed.
- Anyone can call this function.

```solidity
function updateClientMetadata(uint32 tokenId, string calldata name, string calldata description) external
```

- Updates a client's `name` & `description`.
- Only the owner of the NFT can call this.

```solidity
function updateRewardsForAuctions(uint32 lastNounId) public whenNotPaused
```

- Distribute rewards for auction bidding since the last update until auction with id `lastNounId`.
- If an auction's winning bid was called with a clientId, that client will be reward with `params.auctionRewardBps` bips of the auction's settlement amount.
- At least `minimumAuctionsBetweenUpdates` must happen between updates.
- Gas spent is refunded in `ethToken` (WETH by default).
- Gas is refunded only if at least one auction was rewarded.

```solidity
function updateRewardsForProposalWritingAndVoting(
        uint32 lastProposalId,
        uint32[] calldata votingClientIds
    ) public whenNotPaused
```

- Distribute rewards for proposal creation and voting from the last update until `lastProposalId`.
- A proposal is eligible for rewards if for-votes/total-votes >= params.proposalEligibilityQuorumBps.
- Rewards are calculated by the auctions revenue during the period between the creation time of last proposal in the previous update until the current last proposal with id `lastProposalId`.
- Gas spent is refunded in `ethToken`.
- `lastProposalId` id of the last proposal to include in the rewards distribution. all proposals up to and including this id must have ended voting.
- `votingClientIds` votingClientIds array of sorted client ids that were used to vote on the eligible proposals in this rewards distribution. reverts if contains duplicates. reverts if not sorted. reverts if a clientId had zero votes. You may use `getVotingClientIds` as a convenience function to get the correct `votingClientIds`.

```solidity
function withdrawClientBalance(uint32 clientId, address to, uint96 amount) public whenNotPaused
```

- Withdraws the balance of a client
- The caller must be the owner of the NFT with id `clientId` and the client must be approved by the DAO.

#### Public read functions

```solidity
function clientBalance(uint32 clientId) public view returns (uint96)
```

- Returns the withdrawable balance of client with id `clientId`.

```solidity
function getVotingClientIds(uint32 lastProposalId) public view returns (uint32[] memory)
```

- Returns the clientIds that are needed to be passed as a parameter to `updateRewardsForProposalWritingAndVoting`.
- This is not meant to be called onchain because it may be very gas intensive.

```solidity
function getAuctionRevenue(
        uint256 firstNounId,
        uint256 endTimestamp
    ) public view returns (uint256 sumRevenue, uint256 lastAuctionId)
```

- Returns the sum of revenue via auctions from auctioning noun with id `firstNounId` until timestamp of `endTimestamp.

#### Admin functions (only callable by the DAO)

```solidity
function setClientApproval(uint32 clientId, bool approved) public onlyOwner
```

- Set whether the client is approved to withdraw their reward balance.
- Anyone can mint a client NFT and start earning rewards, but only approved clients can withdraw.
- This way the DAO helps mitigate abuse.

```solidity
function setParams(RewardParams calldata newParams) public onlyOwner
```

- Update params of reward distribution.

```solidity
function setAdmin(address newAdmin) public onlyOwner
```

- Update the admin wallet which is allowed to pause/unpause the contract.

```solidity
function setETHToken(address newToken) public onlyOwner
```

- Update the ERC20 token to use for transfering rewards.

```solidity
function withdrawToken(address token, address to, uint256 amount) public onlyOwner
```

- Withdraw an `amount` of `token` to `to`.
- Used in case of shutting down this contract or changing to a different token.

```solidity
function pause() public onlyOwnerOrAdmin
```

- Pause the contract

```solidity
function unpause() public onlyOwnerOrAdmin
```

- Unpause the contract

```
function setDescriptor(address descriptor_) public onlyOwner
```

- Update the descriptor for the NFT (for changing art).
