# Common Pitfalls & How to Avoid Them

This document covers the most common mistakes when building FHEVM applications, using the Corporate Governance example.

## 1. Missing Permission Grants

### The Problem

```solidity
// ❌ UNSAFE: No permissions granted
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external {
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
    euint32 power = FHE.asEuint32(boardMembers[msg.sender].votingPower);

    // ❌ Missing: FHE.allowThis() and FHE.allow()
    resolutions[_resolutionId].yesVotes = FHE.add(
        resolutions[_resolutionId].yesVotes,
        FHE.select(vote, power, FHE.asEuint32(0))
    );
}
```

**Error you'll see**: Transaction fails silently during FHE operation

### The Solution

```solidity
// ✅ SAFE: Both permissions granted
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external {
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
    euint32 power = FHE.asEuint32(boardMembers[msg.sender].votingPower);

    euint32 contribution = FHE.select(vote, power, FHE.asEuint32(0));

    // ✅ FHE.allowThis() - contract permission
    FHE.allowThis(contribution);

    // ✅ FHE.allow() - user permission
    FHE.allow(contribution, msg.sender);

    resolutions[_resolutionId].yesVotes = FHE.add(
        resolutions[_resolutionId].yesVotes,
        contribution
    );
}
```

**Why both are needed:**
- `FHE.allowThis()` - Allows the contract to use this encrypted value
- `FHE.allow()` - Allows the user to decrypt this encrypted value (if needed)

---

## 2. Invalid or Missing Input Proofs

### The Problem

```solidity
// ❌ UNSAFE: Empty proof
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof  // Will be ignored!
) external {
    // ❌ Empty proof provided
    ebool vote = TFHE.asEbool(_encryptedVote, new bytes(0));
    // This will fail because proof is invalid
}
```

**Error**: "Invalid input proof" or "Proof verification failed"

### The Solution

```solidity
// ✅ SAFE: Always use provided proof
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external {
    // ✅ Use the proof provided by client
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);

    require(inputProof.length > 0, "Proof is required");
    // Continue with safe encrypted value
}
```

**Best practice**: Make proof validation explicit
```solidity
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external {
    // Validate proof upfront
    require(inputProof.length >= 32, "Invalid proof format");

    // Use proof
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
}
```

---

## 3. Decrypting in View Functions

### The Problem

```solidity
// ❌ UNSAFE: Trying to decrypt in view function
function getResults(uint256 _resolutionId)
    external
    view
    returns (uint256, uint256)
{
    euint32 yesVotes = resolutions[_resolutionId].yesVotes;
    euint32 noVotes = resolutions[_resolutionId].noVotes;

    // ❌ CANNOT DECRYPT HERE
    return (
        FHE.decrypt(yesVotes),  // COMPILE ERROR or RUNTIME FAILURE
        FHE.decrypt(noVotes)
    );
}
```

**Error**: "Cannot decrypt in view function" or compilation fails

### The Solution

```solidity
// ✅ SAFE: Request decryption asynchronously
function closeResolution(uint256 _resolutionId) external {
    require(block.timestamp > resolutions[_resolutionId].endTime);

    euint32 yesVotes = resolutions[_resolutionId].yesVotes;
    euint32 noVotes = resolutions[_resolutionId].noVotes;

    // Request decryption
    uint256[] memory cts = new uint256[](2);
    cts[0] = FHE.toUint256(yesVotes);
    cts[1] = FHE.toUint256(noVotes);

    Gateway.requestDecryption(
        cts,
        this.resolveResolution.selector,  // Callback function
        _resolutionId,
        block.timestamp + 100,
        false
    );
}

// ✅ SAFE: Process decrypted data in callback
function resolveResolution(
    uint256 _resolutionId,
    uint256[] memory decryptedVotes
) public onlyGateway {
    uint256 yesVotes = decryptedVotes[0];
    uint256 noVotes = decryptedVotes[1];

    bool passed = yesVotes > noVotes;
    resolutions[_resolutionId].finalResult = passed;

    emit ResolutionClosed(_resolutionId, passed);
}
```

**Pattern**: Always use Gateway for decryption
1. Call `Gateway.requestDecryption()` from state-changing function
2. Process in callback function with `onlyGateway` modifier
3. Results only available after decryption completes

---

## 4. Mismatched Encryption Binding

### The Problem

```typescript
// ❌ UNSAFE: Alice encrypts, Bob submits
const aliceEncrypted = await fhevm.createEncryptedInput(
    contractAddr,
    alice.address  // ✅ Encrypted by Alice
).add1(1).encrypt();

// ❌ Bob tries to use Alice's encryption
const tx = await contract
    .connect(bob)  // Bob submitting
    .castVote(0, aliceEncrypted.handles[0], aliceEncrypted.inputProof);
    // Proof says alice, but bob signing - FAILS!
```

**Error**: "Invalid proof" or "Signature verification failed"

### The Solution

```typescript
// ✅ SAFE: Each user encrypts their own vote
async function castVote(contractAddr, userAddr, vote) {
    // User encrypts their own vote
    const encrypted = await fhevm.createEncryptedInput(
        contractAddr,
        userAddr  // ✅ User who is voting
    );

    if (vote === 'yes') {
        encrypted.add1(1);
    } else {
        encrypted.add1(0);
    }

    const proof = encrypted.encrypt();

    // User submits their own encryption
    const contract = await ethers.getContractAt(
        "CorporateGovernance",
        contractAddr,
        userSigner  // ✅ Signing as same user
    );

    return contract.castVote(
        resolutionId,
        proof.handles[0],
        proof.inputProof
    );
}
```

**Pattern**: `[contract, user]` binding
- User `X` encrypts data with key for `[contract, X]`
- Only user `X` can submit this encrypted data
- Contract verifies binding through proof

---

## 5. Exposing Encrypted Values Publicly

### The Problem

```solidity
// ❌ UNSAFE: Encrypted value visible on-chain
euint32 public yesVotes;  // Anyone can read the encrypted value
euint32 public noVotes;

function showVotes() external view returns (euint32, euint32) {
    return (yesVotes, noVotes);  // Encrypted but publicly available
}
```

**Issue**: While encrypted, the values are visible and might be decryptable with future quantum computers

### The Solution

```solidity
// ✅ SAFE: Encrypted values private, only revealed at end
euint32 private yesVotes;  // Hidden from external access
euint32 private noVotes;

function closeResolution(uint256 _resolutionId) external {
    // Only reveal tallies when voting ends
    uint256[] memory cts = new uint256[](2);
    cts[0] = FHE.toUint256(yesVotes);
    cts[1] = FHE.toUint256(noVotes);

    Gateway.requestDecryption(
        cts,
        this.resolveResolution.selector,
        _resolutionId,
        block.timestamp + 100,
        false
    );
}
```

**Pattern**: Minimize exposure window
1. Keep encrypted values private
2. Only decrypt when necessary
3. Reveal plaintext only when voting complete

---

## 6. Forgetting Access Control

### The Problem

```solidity
// ❌ UNSAFE: Anyone can vote
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external {  // No modifier!
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);

    // Non-members can vote too!
    euint32 votingPower = FHE.asEuint32(boardMembers[msg.sender].votingPower);
    // This will be 0 for non-members, but they can still vote
}
```

**Issue**: Non-board-members can vote with 0 power (ineffective but still bad)

### The Solution

```solidity
// ✅ SAFE: Access control enforced
modifier onlyBoardMember() {
    require(boardMembers[msg.sender].isActive, "Not a board member");
    _;
}

function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external onlyBoardMember {  // ✅ Require board member
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);

    // Only board members reach here
    euint32 votingPower = FHE.asEuint32(boardMembers[msg.sender].votingPower);

    // ... rest of function
}
```

**Pattern**: Always combine with encryption
```solidity
function castVote(...)
    external
    onlyBoardMember  // Access control first
{
    ebool vote = TFHE.asEbool(_vote, inputProof);  // Then validate proof
    FHE.allowThis(vote);  // Then grant permissions
    // ... operations
}
```

---

## 7. Not Validating Input

### The Problem

```solidity
// ❌ UNSAFE: No input validation
function createResolution(
    string memory _title,
    string memory _description,
    uint256 _requiredQuorum
) external {
    // What if _requiredQuorum is 0? Or larger than total voting power?
    resolutions[resolutionCounter] = Resolution({
        title: _title,
        description: _description,
        requiredQuorum: _requiredQuorum,  // No validation!
        // ...
    });

    resolutionCounter++;
}
```

**Issues**:
- Quorum could be 0 (always passes)
- Quorum could exceed total voting power (never passes)
- Strings could be empty

### The Solution

```solidity
// ✅ SAFE: Input validation
function createResolution(
    string memory _title,
    string memory _description,
    uint256 _requiredQuorum
) external onlyBoardMember {
    // Validate all inputs
    require(bytes(_title).length > 0, "Title required");
    require(bytes(_description).length > 0, "Description required");
    require(_requiredQuorum > 0, "Quorum must be positive");
    require(
        _requiredQuorum <= totalVotingPower,
        "Quorum exceeds voting power"
    );

    // Only now safe to proceed
    uint256 resId = resolutionCounter++;
    resolutions[resId] = Resolution({
        id: resId,
        title: _title,
        description: _description,
        requiredQuorum: _requiredQuorum,
        startTime: block.timestamp,
        endTime: block.timestamp + VOTING_DURATION,
        active: true,
        creator: msg.sender,
        yesVotes: FHE.asEuint32(0),
        noVotes: FHE.asEuint32(0)
    });

    emit ResolutionCreated(resId, _title, msg.sender);
}
```

---

## 8. Incorrect Gas Limits

### The Problem

```typescript
// ❌ UNSAFE: Default gas limit
const tx = await contract.castVote(
    resolutionId,
    encryptedVote,
    inputProof
    // Gas limit auto-estimated, might fail
);
```

**Issue**: FHE operations are expensive; default estimation fails

### The Solution

```typescript
// ✅ SAFE: Explicit gas limit for FHE operations
const tx = await contract.castVote(
    resolutionId,
    encryptedVote,
    inputProof,
    {
        gasLimit: 500000  // Sufficient for FHE vote casting
    }
);

await tx.wait();
```

**Gas estimates for governance**:
- Create resolution: ~200,000 gas
- Cast vote: ~400,000 gas
- Close resolution: ~150,000 gas
- Decrypt callback: ~100,000 gas

---

## Checklist Before Deploying

- ✅ All encrypted values have permission grants (`allowThis` + `allow`)
- ✅ All input proofs are validated
- ✅ Decryption only happens in gateway callbacks
- ✅ Encrypted inputs properly bound to users
- ✅ Access control modifiers on all sensitive functions
- ✅ All user inputs validated
- ✅ Gas limits set appropriately for FHE operations
- ✅ Encrypted values marked as private
- ✅ No decryption attempts in view functions
- ✅ Gateway callback function properly registered

---

## Testing for Pitfalls

```typescript
describe("Pitfall Prevention", function () {
    it("should fail without permissions", async () => {
        // Test missing permissions
        await expect(badVoteWithoutPermissions())
            .to.be.reverted;
    });

    it("should fail with invalid proof", async () => {
        const badProof = new Uint8Array(64);
        await expect(
            contract.castVote(0, vote, badProof)
        ).to.be.reverted;
    });

    it("should reject non-board-members", async () => {
        const encrypted = await createEncrypted(bob);
        await expect(
            contract.connect(bob).castVote(0, encrypted.handle, encrypted.proof)
        ).to.be.revertedWith("Not a board member");
    });

    it("should validate quorum on creation", async () => {
        await expect(
            contract.createResolution("Test", "Desc", 0)  // Quorum 0
        ).to.be.revertedWith("Quorum must be positive");

        await expect(
            contract.createResolution("Test", "Desc", 99999)  // Too high
        ).to.be.revertedWith("Quorum exceeds voting power");
    });
});
```

---

## Summary

The most critical rules:
1. **Always grant permissions** - Both `allowThis` and `allow`
2. **Always validate proofs** - No empty or fake proofs
3. **Never decrypt in views** - Only in gateway callbacks
4. **Verify bindings** - Ensure user owns encrypted data
5. **Use access control** - Restrict who can call functions
6. **Validate inputs** - Check all user-provided data
7. **Set appropriate gas** - FHE operations are expensive
8. **Keep encrypted values hidden** - Use `private` visibility

Follow these patterns and your FHEVM application will be secure!
