# Input Proofs & Validation - Corporate Governance Example

## What Are Input Proofs?

Input proofs are **zero-knowledge proofs** that cryptographically attest:
1. The input was encrypted with the correct public key
2. The input is bound to a specific `[contract, user]` pair
3. The encryption was performed correctly

Without input proofs, anyone could submit arbitrary encrypted data.

## Why They Matter

### Without Input Proofs

```solidity
// ❌ UNSAFE: No validation
function castVote(einput _vote, bytes calldata) external {
    ebool vote = TFHE.asEbool(_vote, new bytes(0));  // No proof!
    // Attacker could submit garbage data and claim it's a vote
}
```

**Problems:**
- No guarantee input is actually encrypted
- No binding to user identity
- Contract cannot verify legitimacy
- Security completely compromised

### With Input Proofs

```solidity
// ✅ SECURE: Validated input
function castVote(einput _vote, bytes calldata inputProof) external {
    ebool vote = TFHE.asEbool(_vote, inputProof);  // Proof verified!
    // FHEVM verifies proof before processing
}
```

**Benefits:**
- Cryptographic guarantee of proper encryption
- Binding to user and contract
- FHEVM validates proof automatically
- Only legitimate encrypted inputs accepted

## Implementation in Corporate Governance

### Vote Submission with Proof

```solidity
struct Resolution {
    // ... other fields
    euint32 yesVotes;   // Encrypted tallies
    euint32 noVotes;
}

function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,      // Encrypted input from user
    bytes calldata inputProof    // Zero-knowledge proof
) external onlyBoardMember {

    // ✅ FHEVM verifies proof automatically
    // This line validates that _encryptedVote is properly encrypted
    // and bound to [this contract, msg.sender]
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);

    euint32 votingPower = FHE.asEuint32(boardMembers[msg.sender].votingPower);

    // Safe to use encrypted vote - it's been validated
    resolutions[_resolutionId].yesVotes = FHE.add(
        resolutions[_resolutionId].yesVotes,
        FHE.select(vote, votingPower, FHE.asEuint32(0))
    );

    // Grant permissions
    FHE.allowThis(resolutions[_resolutionId].yesVotes);
    FHE.allow(resolutions[_resolutionId].yesVotes, msg.sender);
}
```

## Client-Side: Creating Encrypted Input with Proof

### Using FHEVM Hardhat Plugin

```typescript
import { ethers, fhevm } from "hardhat";
import { FhevmType } from "@fhevm/hardhat-plugin";

async function submitVote(contractAddress, userAddress, vote) {
    // 1. Create encrypted input bound to [contract, user]
    const encrypted = await fhevm.createEncryptedInput(
        contractAddress,
        userAddress
    );

    // 2. Add the vote (1 = yes, 0 = no)
    if (vote === "yes") {
        encrypted.add1(1);
    } else {
        encrypted.add1(0);
    }

    // 3. Encrypt and get proof
    const encryptedData = encrypted.encrypt();
    // encryptedData now contains:
    // - handles[0]: encrypted vote handle
    // - inputProof: zero-knowledge proof

    // 4. Submit to contract
    const contract = new ethers.Contract(
        contractAddress,
        CONTRACT_ABI,
        signer
    );

    const tx = await contract.castVote(
        resolutionId,
        encryptedData.handles[0],  // Encrypted value
        encryptedData.inputProof,  // Proof
        { gasLimit: 500000 }
    );

    await tx.wait();
}
```

### What the Proof Contains

The input proof is a cryptographic attestation that:

```
InputProof {
    signer: bob.address,           // User who encrypted this
    contractAddress: governance,   // Target contract
    encryptedValue: 0x1a2b3c...,  // Encrypted data
    timestamp: 1234567890,         // When encrypted
    signature: 0x...              // Proof signature
}
```

FHEVM verifies:
1. ✅ Signature is valid (from user's key)
2. ✅ Contract address matches
3. ✅ Encrypted value is not replayed
4. ✅ Timestamp is reasonable
5. ✅ User hasn't been revoked

## Common Pitfalls

### ❌ Pitfall 1: Invalid or Missing Proof

```solidity
// DON'T do this:
function castVote(
    uint256 resId,
    einput vote,
    bytes calldata inputProof
) external {
    // ❌ Empty proof - will fail!
    ebool encrypted = TFHE.asEbool(vote, new bytes(0));
}
```

**Why it fails**: FHEVM cannot verify the binding

**Solution**: Always use provided proof
```solidity
// ✅ DO this:
ebool encrypted = TFHE.asEbool(vote, inputProof);
```

### ❌ Pitfall 2: Wrong User

```typescript
// Client-side example
const encrypted = await fhevm.createEncryptedInput(
    contractAddr,
    alice.address  // ✅ Alice encrypts
).add1(1).encrypt();

// ❌ DON'T: Bob submits Alice's vote
const tx = await contract
    .connect(bob)  // Bob signing
    .castVote(0, encrypted.handles[0], encrypted.inputProof);
    // But proof says alice encrypted this - FAILS!
```

**Why it fails**: Proof is bound to alice, but bob is submitting

**Solution**: Each user must encrypt their own vote
```typescript
// ✅ DO this:
const bobEncrypted = await fhevm.createEncryptedInput(
    contractAddr,
    bob.address  // Bob encrypts
).add1(1).encrypt();

const tx = await contract
    .connect(bob)  // Bob submitting own encryption
    .castVote(0, bobEncrypted.handles[0], bobEncrypted.inputProof);
```

### ❌ Pitfall 3: Stale Proof

```typescript
// ❌ DON'T: Reuse old proof
const proof1 = await createVoteProof(1);
await sleep(1000);
// Proof 1 now potentially expired
await contract.castVote(resId, proof1.handle, proof1.proof);
```

**Why it fails**: Proofs include timestamp and cannot be replayed

**Solution**: Create fresh proof for each submission
```typescript
// ✅ DO this:
const proof = await createVoteProof(1);
const tx = await contract.castVote(resId, proof.handle, proof.proof);
await tx.wait();

// If retry needed, create new proof
const newProof = await createVoteProof(1);
const retryTx = await contract.castVote(resId, newProof.handle, newProof.proof);
```

### ❌ Pitfall 4: Incorrect Contract Address

```typescript
// DON'T: Encrypt for wrong contract
const wrong = await fhevm.createEncryptedInput(
    "0x1234567890..."  // ❌ Wrong address
    alice.address
).add1(1).encrypt();

// Submit to actual contract - FAILS!
await realContract.castVote(0, wrong.handles[0], wrong.inputProof);
```

**Why it fails**: Proof bound to different contract

**Solution**: Always use actual contract address
```typescript
// ✅ DO this:
const proof = await fhevm.createEncryptedInput(
    realContractAddress,  // ✅ Correct address
    alice.address
).add1(1).encrypt();

await realContract.castVote(0, proof.handles[0], proof.inputProof);
```

## Best Practices

### 1. Always Validate Proofs

```solidity
// ✅ GOOD: Proof is mandatory parameter
function castVote(
    uint256 _resId,
    einput _vote,
    bytes calldata _inputProof  // Not optional!
) external {
    require(_inputProof.length > 0, "Proof required");
    ebool vote = TFHE.asEbool(_vote, _inputProof);
}
```

### 2. Validate User Identity

```solidity
// ✅ GOOD: Combine access control with proof validation
function castVote(
    uint256 _resId,
    einput _vote,
    bytes calldata _inputProof
) external onlyBoardMember {  // Access control first
    ebool vote = TFHE.asEbool(_vote, _inputProof);  // Then validate proof
}
```

### 3. Include Reasonable Limits

```typescript
// ✅ GOOD: Client validates before submitting
async function castVote(vote) {
    // Validate user address
    if (!userAddress || userAddress === ethers.ZeroAddress) {
        throw new Error("Invalid user address");
    }

    // Validate contract address
    if (!contractAddress || contractAddress === ethers.ZeroAddress) {
        throw new Error("Invalid contract address");
    }

    // Create proof with validation
    const encrypted = await fhevm.createEncryptedInput(
        contractAddress,
        userAddress
    );

    if (!encrypted || !encrypted.inputProof) {
        throw new Error("Failed to generate proof");
    }

    // Submit
    const tx = await contract.castVote(
        resId,
        encrypted.handles[0],
        encrypted.inputProof,
        { gasLimit: 500000 }
    );

    return tx.wait();
}
```

## Testing Proofs

```typescript
describe("Input Proof Validation", function () {
    it("should accept valid proof", async () => {
        const encrypted = await fhevm
            .createEncryptedInput(contractAddr, alice.address)
            .add1(1)
            .encrypt();

        const tx = await contract.connect(alice).castVote(
            0,
            encrypted.handles[0],
            encrypted.inputProof
        );

        expect(tx).to.emit(contract, "VoteCast");
    });

    it("should reject invalid proof", async () => {
        const encrypted = await fhevm
            .createEncryptedInput(contractAddr, alice.address)
            .add1(1)
            .encrypt();

        const badProof = new Uint8Array(64);  // Wrong proof format

        await expect(
            contract.connect(alice).castVote(
                0,
                encrypted.handles[0],
                badProof
            )
        ).to.be.revertedWith("Invalid proof");
    });

    it("should reject proof from wrong user", async () => {
        const encrypted = await fhevm
            .createEncryptedInput(contractAddr, alice.address)
            .add1(1)
            .encrypt();

        // Bob tries to use Alice's proof
        await expect(
            contract.connect(bob).castVote(
                0,
                encrypted.handles[0],
                encrypted.inputProof
            )
        ).to.be.revertedWith("Invalid proof");
    });
});
```

## Summary

Input proofs are critical security components that:
- **Verify encryption**: Proof of correct encryption
- **Bind identities**: Bind to specific user and contract
- **Prevent replay**: Cannot be reused across contexts
- **Ensure integrity**: Detect tampering or forgery

Always include input proof validation in FHEVM contracts!
