# Nouns Fungible Token V1 spec

## `TokenDeployer` contract

A factory contract that allows anyone to deploy a new instance of an NFT-backed fungible token.

### `deployToken`

```solidity=
function deployToken(
    address erc721Token,
    address owner,
    string calldata name,
    string calldata symbol,
    uint8 decimals,
    uint88 unitsPerNFT
)
```

Creates a new ERC1967 upgradeable proxy contract, with its implementation set as the NFT-backed fungible token contract.

Initializes the fungible token with the same parameters this deploy function accepts. See the token contract for more information.

Contracts are deployed using the `CREATE` operation, such that token addresses are harder to predict than when using `CREATE2`.

Q: why is it important that the address is harder to predict? Maybe it's actually more important that the address _is_ predictable so that we can set this address in the DAO contract in the same proposal as deploying the token?

Q: maybe mention that we plan the DAO to call this function via a proposal?

## `NFTBackedToken` contract

A UUPS upgradeable ERC20 token contract that supports depositing or redeeming ERC721 tokens of a specific token, in exchange for a fixed amount of fungible tokens.

Q: do we want it to be a `ERC20Permit`? (i.e. using ERC-2612)

For example for Nouns, we expect the fungible $nouns token to have the exchange rate of 1M $nouns for each Noun NFT deposited or redeemed.

### `initialize`

```solidity=
function initialize(
    address owner,
    string calldata name,
    string calldata symbol,
    uint8 decimals,
    address nft,
    uint88 amountPerNFT,
    address admin
)
```

Configures a new instance of an NFT-based fungible token:

- `owner`: the address that can initiate upgrades and set/revoke the admin role.
- `name`: the name of the ERC20 token.
- `symbol`: the symbol of the ERC20 token.
- `decimals`: the number of digits after the decimal point, commonly set to 18.
- `nft`: the address of the ERC721 token that is backing this ERC20 token.
- `amountPerNFT`: how many tokens to mint/burn upon a deposit/redemption of a backing ERC721 token.
  Q: maybe it's easier if `amountPerNFT` includes decimals?
- `admin`: the address of the admin account that can pause/unpause in emergencies (more info below).

### `deposit`

```solidity
function deposit(uint256[] calldata tokenIds, address to)
```

Transfers all ERC721 tokens in the `tokenIds` list from `msg.sender` to this contract.

Mints ERC20 tokens in the amount of `amountPerNFT * tokenIds.length * 10^decimals`, increasing the balance of `to`.

For example, if `amountPerNFT` is 1M and three NFTs are deposited, the `to` account receives 3M ERC20 tokens.

Requirements:

- This contract needs to be approved to transfer ERC721 tokens on behalf of `msg.sender`.
- This contract must not be paused.

### `redeem`

```solidity
function redeem(uint256[] calldata tokenIds, address to)
```

Burns ERC20 tokens in the amount of `amountPerNFT * tokenIds.length * 10^decimals`, decreasing the balane of `msg.sender`.

Transfers all ERC721 tokens in the `tokenIds` list from this contract to `to`.

Requirements:

- All `tokenIds` must be owned by this contract.
- `msg.sender` must have a sufficient balance of ERC20 tokens to burn.
- This contract must not be paused.

### `upgradeToAndCall`

```solidity
function upgradeToAndCall(address newImplementation, bytes memory data)
```

Changes the token contract's implementation to be `newImplementation` and calls a function on the new implementation if `data` is not empty.

Requirements:

- `msg.sender` must be `owner`.
- Upgrades must be enabled.

### `disableUpgrades`

```solidity
function disableUpgrades()
```

Sets a flag on the contract such that all future upgrade attempts will revert.

Requirements:

- `msg.sender` must be `owner`.

### `pause` and `unpause`

```solidity
function pause()
function unpause()
```

Pauses or unpauses the `deposit` and `redeem` functions.

Requirements:

- `msg.sender` must be `admin`. Q: shouldn't the DAO also be able to pause/unpause? or at least unpause? e.g. in case admin paused and then lost access.
- `admin` must not be burned, i.e. cannot equal the zero address.

### `burnAdminPower`

```solidity
function burnAdminPower()
```

Sets `admin` to the zero address, disabling the admin power to pause or unpause.

Requirements:

- `msg.sender` must be `admin` or `owner`.
- This contract must not be paused. Q: why does the contract need to be unpaused for this function?

## `NounsDAOLogic` contract

The DAO's fork mechanism needs an upgrade, to account for the fact that holding Nouns-backed ERC20 tokens is akin to holding Nouns, which are excluded from total supply and treasury-fair-share calculations.

### `adjustedTotalSupply`

The upgraded version will subtract from adjusted supply the amount of Noun NFTs backed by the amount of Nouns-backed ERC20 tokens owned by the treasury, rounded down.

Current calculation:

```solidity
nouns.totalSupply() - nouns.balanceOf(address(ds.timelock)) - forkEscrow.numTokensOwnedByDAO()
```

Updated calculation:

```solidity
nouns.totalSupply() - nouns.balanceOf(address(timelock)) - forkEscrow.numTokensOwnedByDAO() - (nounsFungibleToken.balanceOf(timelock) / nounsFungibleToken.amountPerNFT())
```

Q: make sure the calculation above takes decimals into account ^

Q: we need a admin function that sets `nounsFungibleToken`

### `setErc20TokensToIncludeInFork`

The upgraded version will revert if a proposal attempts to add the Nouns-backed ERC20 token address to the list of ERC20s that are sent to fork DAOs.

## Bridging to L2

NFT-backed ERC20 tokens are deployed on Ethereum mainnet (L1) such that it's easiest to bridge the token to any L2.

We have selected a few L2s to deploy the Nouns-backed token to:

- Optimism
- Zora
- Base

All of the above are built on the OP stack and all use the [Superchain Token List](https://github.com/ethereum-optimism/ethereum-optimism.github.io) as a source of truth of which tokens to include in their standard bridges. We intend for the Nouns-backed token ($nouns) to be approved on all three standard bridges.
