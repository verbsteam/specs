## Proposals

```solidity

// Core contract storage

struct ProposalCore {
    // See `hashProposal`
    bytes32 proposalHash;
    uint48 voteStart;
    uint32 voteDuration;
    bool executed;
    bool canceled;
    uint32 clientId;
}

mapping(uint proposalId => ProposalCore) proposals;

/**
* @notice Creates a new proposal
* @param nounIds A list of noun ids that are used to create the proposals. The list must be at least `proposalThreshold` in size. The caller (msg.sender) must be the owner or delegate of the nouns.
* These IDs must not be part of a live proposal. (TBD)
* @param targets Target addresses for proposal calls
* @param values Eth values for proposal calls
* @param signatures Function signatures for proposal calls
* @param calldatas Calldatas for proposal calls
* @param description String description of the proposal
* @param clientId Optional: The ID of the client that faciliated creating the proposal
*/
function propose(
    uint256[] nounIds,
    address[] targets,
    uint256[] values,
    string[] signatures,
    bytes[] calldatas,
    string description,
    uint32 clientId
) external returns (uint proposalId);

/**
* @notice Creates a hash of the proposal actions & nounIds. Its purpose is to reduce the amount of data needed to be saved onchain.
* The hash is:
* keccak256(
*     proposalId,
*     keccak256(nounIds),
*     keccak256(targets, values, signatures, calldatas)
* )
*/
function hashProposal(
    uint256 proposalId,
    uint256[] nounIds,
    address[] targets,
    uint256[] values,
    string[] signatures,
    bytes[] calldatas
) external returns (bytes32) {
    bytes32 nounIdsHash = keccak256(abi.encode(nounIds));
    bytes32 actionsHash = keccak256(abi.encode(targets, values, signatures, calldatas))
    return keccak256(abi.encode(proposalId, nounIdsHash, actionsHash);
}


/**
* @notice Update a proposal transactions and description.
* Only the proposer can update it, and only during the updateable period.
* @param proposalId Proposal's id
* @param nounIds The nounIds used to create the proposal. Need to be passed because the contract only saves the hash of the nounIds
* @param TBD: rootHash?
* @param targets Updated target addresses for proposal calls
* @param values Updated eth values for proposal calls
* @param signatures Updated function signatures for proposal calls
* @param calldatas Updated calldatas for proposal calls
* @param description Updated description of the proposal
* @param updateMessage Short message to explain the update
*/
function updateProposal(
        uint256 proposalId,
        uint256[] nounIds,
        address[] targets,
        uint256[] values,
        string[] signatures,
        bytes[] calldatas,
        string description,
        string updateMessage
) external {

// TBD: function proposeOnTimelockV1

```

## Proposals by sigs

TODO

```
// TODO: function proposeBySigs
// TODO: function cancelSig
```

## Delegation

TODO

## Voting

TODO
