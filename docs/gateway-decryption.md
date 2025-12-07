# Gateway Decryption - Corporate Governance Example

## Overview

Gateway-based decryption is the mechanism by which encrypted results are revealed in FHEVM applications. This example demonstrates asynchronous decryption for revealing final voting results.

## The Problem: Why Async Decryption?

### Decryption Cannot Be Synchronous

```solidity
// ❌ IMPOSSIBLE: Synchronous decryption in view function
function getResults() external view returns (uint256, uint256) {
    // Cannot decrypt here - view functions cannot call external services
    return (
        TFHE.decrypt(yesVotes),   // ERROR!
        TFHE.decrypt(noVotes)     // ERROR!
    );
}

// ❌ IMPOSSIBLE: Synchronous decryption in state function
function closeResolution() external {
    // Cannot wait for decryption - would hang the transaction
    uint256 yes = TFHE.decrypt(yesVotes);  // ERROR!
}
```

### Solution: Asynchronous Gateway Pattern

```solidity
// ✅ CORRECT: Request decryption asynchronously
function closeResolution(uint256 _resolutionId) external {
    // Step 1: Request decryption
    uint256[] memory cts = new uint256[](2);
    cts[0] = FHE.toUint256(resolutions[_resolutionId].yesVotes);
    cts[1] = FHE.toUint256(resolutions[_resolutionId].noVotes);

    Gateway.requestDecryption(
        cts,
        this.resolveResolution.selector,  // Callback function
        _resolutionId,                    // User data (resolution ID)
        block.timestamp + 100,            // Timeout
        false                             // Not unlimited
    );

    // Step 2: Continue execution
    // Function returns immediately
}

// Step 3: Gateway calls callback with results
function resolveResolution(
    uint256 _resolutionId,
    uint256[] memory decryptedVotes
) public onlyGateway {
    // Now we have decrypted results!
    uint256 yesVotes = decryptedVotes[0];
    uint256 noVotes = decryptedVotes[1];

    // Process results
    bool passed = yesVotes > noVotes;
    emit ResolutionClosed(_resolutionId, passed);
}
```

## Complete Decryption Workflow

### Step 1: Initialize Encrypted State

```solidity
struct Resolution {
    euint32 yesVotes;  // Encrypted during voting
    euint32 noVotes;   // Encrypted during voting
    bool closed;
    uint256 finalYes;  // Plaintext, filled after decryption
    uint256 finalNo;   // Plaintext, filled after decryption
}
```

### Step 2: Accumulate Encrypted Votes

```solidity
function castVote(uint256 _resolutionId, einput _vote, bytes calldata proof)
    external
    onlyBoardMember
{
    ebool vote = TFHE.asEbool(_vote, proof);
    euint32 power = TFHE.asEuint32(boardMembers[msg.sender].votingPower);

    // All operations on encrypted data
    euint32 contribution = TFHE.select(vote, power, TFHE.asEuint32(0));
    resolutions[_resolutionId].yesVotes = TFHE.add(
        resolutions[_resolutionId].yesVotes,
        contribution
    );

    FHE.allowThis(resolutions[_resolutionId].yesVotes);
    FHE.allow(resolutions[_resolutionId].yesVotes, msg.sender);
}
```

### Step 3: Request Decryption

```solidity
function closeResolution(uint256 _resolutionId) external {
    Resolution storage res = resolutions[_resolutionId];

    // Verify voting period ended
    require(block.timestamp > res.endTime, "Voting still active");
    require(!res.closed, "Already closed");

    res.closed = true;

    // Prepare encrypted values for decryption
    uint256[] memory cts = new uint256[](2);
    cts[0] = FHE.toUint256(res.yesVotes);
    cts[1] = FHE.toUint256(res.noVotes);

    // Request decryption from gateway
    Gateway.requestDecryption(
        cts,                                  // Values to decrypt
        this.resolveResolution.selector,      // Callback function selector
        _resolutionId,                        // User data (resolution ID)
        block.timestamp + 100,                // Deadline for decryption
        false                                 // Not unlimited size
    );

    emit DecryptionRequested(_resolutionId);
}
```

### Step 4: Gateway Calls Callback

```solidity
/**
 * @notice Gateway callback - called after decryption
 * @dev ONLY called by gateway after successful decryption
 */
function resolveResolution(
    uint256 _resolutionId,
    uint256[] memory decryptedVotes
) public onlyGateway {
    require(decryptedVotes.length == 2, "Invalid results length");

    Resolution storage res = resolutions[_resolutionId];
    require(res.closed, "Resolution not closed yet");

    // Now we have plaintext results!
    uint256 yesVotes = decryptedVotes[0];
    uint256 noVotes = decryptedVotes[1];

    // Store results
    res.finalYes = yesVotes;
    res.finalNo = noVotes;

    // Determine outcome
    bool passed = yesVotes > noVotes;
    uint256 totalVotes = yesVotes + noVotes;

    // Emit result event
    emit ResolutionClosed(_resolutionId, passed, yesVotes, noVotes);
}
```

## Gateway Interaction Details

### Gateway.requestDecryption() Parameters

```solidity
Gateway.requestDecryption(
    uint256[] memory ciphertexts,    // Encrypted values to decrypt
    bytes4 callbackSelector,          // Function selector to call
    uint256 userData,                 // Data passed to callback
    uint256 deadline,                 // Deadline timestamp
    bool unlimited                    // Size limit flag
)
```

### Parameters Explained

| Parameter | Purpose | Example |
|-----------|---------|---------|
| **ciphertexts** | Encrypted values | `[FHE.toUint256(yesVotes), FHE.toUint256(noVotes)]` |
| **callbackSelector** | Which function to call | `this.resolveResolution.selector` |
| **userData** | Data for callback | `_resolutionId` |
| **deadline** | Max wait time | `block.timestamp + 100` |
| **unlimited** | Size restrictions | `false` (use limits) |

## Timeline Example

```
T=0:   function castVote() called
       -> yesVotes = enc(0) ➜ enc(5) ➜ enc(8)
       -> noVotes = enc(0) ➜ enc(0)

T=168 hours (7 days):
       Voting period ends

T=168h + 1min:
       function closeResolution() called
       -> Gateway.requestDecryption() ✓
       -> Function returns immediately

T=168h + 2min:
       Gateway finishes decryption
       -> resolveResolution() called with decrypted values
       -> yesVotes = 8 (plaintext)
       -> noVotes = 0 (plaintext)
       -> Results finalized
```

## Best Practices

1. **Always Include Timeout**
   ```solidity
   block.timestamp + 100  // 100 second timeout
   ```

2. **Validate Callback Input**
   ```solidity
   require(decryptedVotes.length == expectedLength, "Invalid length");
   ```

3. **Mark as Processed**
   ```solidity
   res.finalized = true;  // Prevent double-processing
   ```

4. **Emit Events**
   ```solidity
   emit DecryptionRequested(_resolutionId);
   emit ResolutionFinalized(_resolutionId);
   ```

5. **Check Gateway Authorization**
   ```solidity
   function resolveResolution(...) public onlyGateway {
       // Only gateway can call
   }
   ```

## Summary

Gateway decryption enables:
- ✅ Asynchronous result revelation
- ✅ Privacy preservation during voting
- ✅ Selective disclosure (only final results)
- ✅ Verifiable outcomes
- ✅ Privacy-preserving governance

The pattern:
1. Request decryption after voting ends
2. Gateway decrypts asynchronously
3. Callback processes results
4. Results become public (only when ready)

---

**Chapter**: decryption
**Difficulty**: Advanced
**Prerequisites**: FHEVM basics, understanding of callbacks
