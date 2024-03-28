This document organizes the available functions by the following sections:

- [Delegation](#delegation)
- [Creating proposals](#creating-proposals)
- [Updating & canceling proposals](#updating-and-canceling-proposals)
- [Voting on proposals](#voting-on-proposals)
- [Executing proposals](#executing-proposals)
- Forking?

## Delegation

`NounDelegationToken` is an ERC721 used for delegating Nouns voting power. For each Noun, a NounDelegationToken (NDT) can be minted with the same id. Once an NDT is minted, it must be used instead of the original Noun when voting or creating proposals.

The original Noun owner can always transfer or burn an NDT.

Users use the following functions to change delegations:

- [`setDelegationAdmin`](#setdelegationadmin)
- [`mint`](#mint)
- [`mintBatch`](#mintbatch)
- [`burn`](#mint)
- [`transferFrom`](#transferfrom)

#### `setDelegationAdmin`

```solidity
function setDelegationAdmin(address delegationAdmin) external
```

Updates the delegation admin of the caller (`msg.sender`) to `delegationAdmin`.
The delegation admin can mint, burn & transfer NDTs on behalf of the caller.

For example, if wallet A owns Noun #15, and sets wallet B as its delegation admin, then wallet B can mint a NDT with id #15, burn it, or transfer it.

The main usecase is for keeping Nouns in cold storage, and setting the delegation admin to a hot wallet.

#### `mint`

```solidity
function mint(address to, uint256 tokenId)
```

Mints a new NDT with id of `tokenId` to the wallet of `to`.

_Requirements:_

- Must be called by either:
  - The owner of Noun with id `tokenId`.
  - The "delegationAdmin" of the owner of Noun with id `tokenId`.

#### `mintBatch`

```solidity
function mintBatch(address to, uint256[] calldata tokenIds)
```

Similar to [`mint`](#mint), but performs this for all the tokenIds in `tokenIds`.

#### `burn`

```solidity
function burn(uint256 tokenId)
```

Burns the NDT with id `tokenId`.

_Requirements:_

- Must be called by either:
  - The owner of Noun with id `tokenId`.
  - The "delegationAdmin" of the owner of Noun with id `tokenId`.
  - TODO: should the NDT owner be able to burn?

#### `transferFrom`

```solidity
function transferFrom(address from, address to, uint256 tokenId)
```

Similar to a regular ERC721 `transferFrom`, but in addition to the NDT owner, the original Noun owner of id `tokenId`, or their delegation admin can do transfers.

## Creating proposals

#### `propose`

```solidity
function propose(
    uint256[] calldata tokenIds,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory description,
    uint32 clientId
) external returns (uint256)
```

Creates a proposal.
Returns the proposal id of the new proposal.

_Requirements:_

- `tokenIds` must be a unique ascending array of noun IDs.
- Caller (`msg.sender`) must be the delegate or owner of nouns with IDs `tokenIds`.
- The length of `tokenIds` must be at least `proposalThreshold()`.

_Difference from Gov Bravo_:

- There is no longer a limitation for the proposer to have only one active proposal.

#### `proposalThreshold`

```solidity
function proposalThreshold() public view returns (uint256)
```

Returns the number of nouns required to create a proposal.
The number of nouns need to be greater than the value returned by `proposalThreshold()`.

The value is set by `proposalThresholdBPS` multiplied by `adjustedTotalSupply` (supply of nouns excluding nouns controlled by the treasury).

#### `proposeOnTimelockV1`

```solidity
function proposeOnTimelockV1(
    uint256[] calldata tokenIds,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory description,
    uint32 clientId
) public returns (uint256)
```

Similar to `propose`, but the proposal actions are executed on `timelockV1`.

#### `proposeBySigs`

```solidity
function proposeBySigs(
    uint256[] calldata tokenIds,
    ProposerSignature[] memory proposerSignatures,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory description
) external returns (uint256)
```

Create a new proposal by pooling voting power in order to pass `proposalThreshold`.
The proposal is created by pooling the voting power of the caller, i.e. the proposer (`msg.sender`) and the signers.

- `tokenIds` represent the nouns used be the caller. They must be unique and ascending. The caller must be their delegate, or if there is no delegate, the owner of the nouns.
- Signers data is provided in `proposerSignatures`:

```solidity
struct ProposerSignature {
    /// @notice Signature of a proposal
    bytes sig;
    /// @notice The address of the signer
    address signer;
    /// @notice The timestamp until which the signature is valid
    uint256 expirationTimestamp;
    /// @notice The token IDs signer is using to back the proposal
    /// @dev Must be sorted ascending with no duplicates
    uint256[] tokenIds;
}
```

- The signature is an EIP-712 sig with the following typehash:

```solidity
bytes32 public constant PROPOSAL_TYPEHASH =
        keccak256(
            'Proposal(address proposer,address[] targets,uint256[] values,string[] signatures,bytes[] calldatas,string description,uint256 expiry)'
        );
```

#### `cancelSig`

```solidity
function cancelSig(bytes calldata sig) external
```

- Invalidates a signature that may be used for signing a new proposal.
- Once a signature is canceled, the sender can no longer use it again.
- If the sender changes their mind and want to sign the proposal, they can change the expiry timestamp in order to produce a new signature.
- The signature will only be invalidated when used by the sender. If used by a different account, it will not be invalidated.
- Cancelling a signature for an existing proposal will have no effect. Signers have the ability to cancel a proposal they signed if necessary.

## Updating and canceling proposals

#### `updateProposal`

```solidity
function updateProposal(
    uint256 proposalId,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory description,
    string memory updateMessage
) external
```

- Update a proposal transactions and description.
- Only the proposer can update it, and only during the updateable period.

#### `updateProposalDescription`

```solidity
function updateProposalDescription(
    uint256 proposalId,
    string calldata description,
    string calldata updateMessage
) external
```

- Updates the proposal's description. Only the proposer can update it, and only during the updateable period.

#### `updateProposalTransactions`

```solidity
function updateProposalTransactions(
    uint256 proposalId,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory updateMessage
) external
```

- Updates the proposal's transactions. Only the proposer can update it, and only during the updateable period.

#### `updateProposalBySigs`

```solidity
function updateProposalBySigs(
    uint256 proposalId,
    ProposerSignature[] memory proposerSignatures,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory description,
    string memory updateMessage
) external
```

- Update a proposal's transactions and description that was created with proposeBySigs.
- Only the proposer can update it, during the updateable period.
- Requires the original signers to sign the update.

#### `cancel`

```solidity
function cancel(uint256 proposalId) external
```

Cancels a proposal. Caller must be the proposer or a signer.

#### `veto`

```solidity
function veto(uint256 proposalId) external
```

Vetoes a proposal. Caller must be the vetoer.

## Voting on proposals

#### `castRefundableVote`

```solidity
function castRefundableVote(
  uint256[] calldata tokenIds,
  uint256 proposalId,
  uint8 support,
  uint32 clientId
) external
```

- Casts a vote on proposal with id `proposalId`.
- Caller must be the delegate of nouns with id `tokenIds`. If an ID has no delegate, the caller must be the owner.
- `support is the support value for the vote. 0=against, 1=for, 2=abstain

#### `castRefundableVoteWithReason`

```solidity
function castRefundableVoteWithReason(
    uint256[] calldata tokenIds,
    uint256 proposalId,
    uint8 support,
    string calldata reason,
    uint32 clientId
) public
```

Same as `castRefundableVote` but with a text "reason" for the vote.

## Executing proposals

#### `execute`

```solidity
function execute(
    uint256 proposalId,
    address[] memory targets,
    uint256[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas
) external
```

Executes proposal with id `proposalId`.
The proposal actions need to be passed because only their hash is stored onchain.

The proposal must have passed voting successfully, both quorum and majority of supporting votes.

The proposal can be executed after being queued for a duration after voting has ended. There is no `queue` function, a proposal automatically changes to queued state if the voting was succesful. Queue duration is defined by `queuePeriod`.

Proposals can't be executed while a fork is in progress.
