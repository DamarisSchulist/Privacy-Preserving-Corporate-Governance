# Encrypted State Management - Corporate Governance Example

## Overview

Encrypted state management is fundamental to FHEVM applications. This example demonstrates how to store sensitive data as encrypted types that remain encrypted on-chain throughout the voting period.

## Key Concept

In traditional smart contracts, all state is public:
```solidity
// ❌ UNSAFE: Public vote counts
mapping(uint256 => uint256) public yesVotes;
mapping(uint256 => uint256) public noVotes;
```

With FHEVM, we use encrypted types:
```solidity
// ✅ SECURE: Encrypted vote counts
mapping(uint256 => euint32) private yesVotes;
mapping(uint256 => euint32) private noVotes;
```

## Implementation in Corporate Governance

### Data Structure

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint32, externalEuint32 } from "@fhevm/solidity/lib/FHE.sol";

contract CorporateGovernance {

    struct Resolution {
        uint256 id;                  // ✅ Public: metadata
        string title;                // ✅ Public: metadata
        string description;          // ✅ Public: metadata
        uint256 startTime;          // ✅ Public: metadata
        uint256 endTime;            // ✅ Public: metadata
        bool active;                // ✅ Public: state
        euint32 yesVotes;          // 🔐 ENCRYPTED: sensitive data
        euint32 noVotes;           // 🔐 ENCRYPTED: sensitive data
        address creator;            // ✅ Public: metadata
        uint256 requiredQuorum;     // ✅ Public: rules
    }

    mapping(uint256 => Resolution) private resolutions;
    mapping(address => bool) private boardMembers;
}
```

### Why This Matters

**Without encryption:**
- Everyone can see every individual vote
- Votes are recorded on-chain permanently
- Leads to voter intimidation and coercion
- Violates privacy requirements

**With encryption:**
- Individual votes encrypted and hidden
- Only final tallies revealed
- Privacy maintained throughout voting
- Still maintains transparency and auditability

## Workflow

### 1. Vote Storage (Encrypted)

```solidity
function castVote(
    uint256 _resolutionId,
    externalEuint32 _encryptedVote,
    bytes calldata inputProof
) external onlyBoardMember {

    // Convert external encrypted input to internal encrypted type
    euint32 encryptedVote = FHE.fromExternal(_encryptedVote, inputProof);

    // Get voting power
    euint32 votingPower = FHE.asEuint32(boardMembers[msg.sender].votingPower);

    // Add encrypted vote to encrypted counter (homomorphic operation)
    // Vote remains encrypted on-chain
    resolutions[_resolutionId].yesVotes = FHE.add(
        resolutions[_resolutionId].yesVotes,
        FHE.select(encryptedVote, votingPower, FHE.asEuint32(0))
    );

    // Grant permissions to access encrypted value
    FHE.allowThis(resolutions[_resolutionId].yesVotes);
    FHE.allow(resolutions[_resolutionId].yesVotes, msg.sender);
}
```

**Key Points:**
- `euint32` type signals this value is encrypted
- Encrypted values can only be operated on with FHE operations
- Cannot be returned from view functions
- Cannot be used in regular arithmetic

### 2. Encrypted Retrieval

```solidity
// ✅ CORRECT: Can retrieve encrypted value from view function
function getEncryptedVotes(uint256 _resolutionId)
    external
    view
    returns (euint32)
{
    return resolutions[_resolutionId].yesVotes;
}
```

**However:** This doesn't actually reveal the value - it's still encrypted!

### 3. Decryption (Only for Final Results)

```solidity
function closeResolution(uint256 _resolutionId) external {
    require(!resolutions[_resolutionId].active, "Still voting");

    // Request decryption of final vote counts
    uint256[] memory cts = new uint256[](2);
    cts[0] = FHE.toUint256(resolutions[_resolutionId].yesVotes);
    cts[1] = FHE.toUint256(resolutions[_resolutionId].noVotes);

    Gateway.requestDecryption(
        cts,
        this.resolveResolution.selector,
        _resolutionId,
        block.timestamp + 100,
        false
    );
}

function resolveResolution(uint256 _resolutionId, uint256[] memory decryptedVotes)
    public
    onlyGateway
{
    uint256 yesVotes = decryptedVotes[0];
    uint256 noVotes = decryptedVotes[1];

    bool passed = yesVotes > noVotes;
    emit ResolutionClosed(_resolutionId, passed);
}
```

## Common Pitfalls

### ❌ Pitfall 1: Trying to Access Encrypted Value Directly

```solidity
// DON'T do this:
function getBadVotes() external view returns (uint256) {
    return uint256(resolutions[0].yesVotes);  // ❌ Type error!
}
```

**Solution**: Use proper FHE operations
```solidity
// ✅ DO this:
function getEncryptedVotes() external view returns (euint32) {
    return resolutions[0].yesVotes;  // Safe to return encrypted value
}
```

### ❌ Pitfall 2: Storing Unencrypted Sensitive Data

```solidity
// DON'T do this:
mapping(uint256 => mapping(address => bool)) public hasVoted;  // Reveals voting
mapping(uint256 => mapping(address => uint256)) public voteChoice;  // Reveals choice
```

**Solution**: Only store encrypted sensitive data
```solidity
// ✅ DO this:
mapping(uint256 => mapping(address => bool)) private hasVoted;  // No value revealed
// Don't store individual votes at all - only encrypted tallies
```

### ❌ Pitfall 3: Trying to Decrypt in Wrong Place

```solidity
// DON'T do this:
function getResults() external view returns (uint256, uint256) {
    // ❌ Can't decrypt in view function!
    return (
        FHE.decrypt(resolutions[0].yesVotes),
        FHE.decrypt(resolutions[0].noVotes)
    );
}
```

**Solution**: Use gateway callback
```solidity
// ✅ DO this:
function closeResolution(uint256 id) external {
    // Request decryption
    Gateway.requestDecryption(
        cts,
        this.resolveResolution.selector,
        id,
        block.timestamp + 100,
        false
    );
}

function resolveResolution(uint256 id, uint256[] memory decrypted) public onlyGateway {
    // ✅ Safe to process decrypted results here
}
```

## Best Practices

### 1. Minimize Public Data

```solidity
// ✅ GOOD: Only essential metadata is public
struct Resolution {
    string title;        // Public
    string description;  // Public
    uint256 endTime;     // Public
    euint32 yesVotes;   // Encrypted
    euint32 noVotes;    // Encrypted
}
```

### 2. Use Proper Access Control

```solidity
// ✅ GOOD: Combine access control with encryption
function castVote(...) external onlyBoardMember {
    euint32 vote = FHE.fromExternal(_encryptedVote, inputProof);
    // ... encrypted operations
}
```

### 3. Always Grant Permissions

```solidity
// ✅ GOOD: Both permissions granted
FHE.allowThis(encryptedValue);
FHE.allow(encryptedValue, msg.sender);
```

## Testing Encrypted State

```typescript
import { expect } from "chai";
import { ethers, fhevm } from "hardhat";
import { FhevmType } from "@fhevm/hardhat-plugin";

describe("Encrypted State", function () {
    it("vote counts remain encrypted during voting", async () => {
        // Create resolution
        const tx = await contract.createResolution("Test", "Description", 5);
        await tx.wait();

        // Cast vote (encrypted)
        const encrypted = await fhevm
            .createEncryptedInput(contractAddr, alice.address)
            .add1(1)  // yes vote
            .encrypt();

        const voteTx = await contract
            .connect(alice)
            .castVote(0, encrypted.handles[0], encrypted.inputProof);
        await voteTx.wait();

        // Get encrypted votes - still encrypted
        const encryptedVotes = await contract.getEncryptedVotes(0);

        // Cannot directly compare - must use FHE functions
        expect(encryptedVotes).not.to.equal(0);

        // Close resolution to trigger decryption
        const closeTx = await contract.closeResolution(0);
        await closeTx.wait();

        // Wait for gateway callback and verify results
        // Results now available in decrypted form
    });
});
```

## Summary

Encrypted state management is essential for:
- **Privacy**: Sensitive data remains hidden
- **Security**: Cannot be manipulated externally
- **Compliance**: Meets regulatory requirements
- **Trust**: No need to trust data handlers

This example demonstrates how FHEVM enables practical applications while maintaining strong privacy guarantees.
