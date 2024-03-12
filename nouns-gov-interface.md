This document organizes the available functions by the following sections:

- [Delegation](#delegation)
- [Creating proposals](#creating-proposals)
- Updating & canceling proposals
- Voting on proposals
- Executing proposals

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

TODO
