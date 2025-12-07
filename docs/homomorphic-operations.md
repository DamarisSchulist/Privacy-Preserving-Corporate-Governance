# Homomorphic Operations - Corporate Governance Example

## Overview

Homomorphic operations allow computations on encrypted data without decryption. This example demonstrates key FHE operations in the context of confidential voting.

## Core FHE Operations Used

### 1. **TFHE.add() - Encrypted Addition**

```solidity
// Add encrypted values together
resolution.yesVotes = TFHE.add(
    resolution.yesVotes,
    encryptedContribution
);
```

**Use Case**: Tallying encrypted votes from multiple board members

**Properties**:
- Both operands remain encrypted
- Result is encrypted
- No intermediate decryption needed
- Maintains privacy throughout

### 2. **TFHE.select() - Encrypted Conditional**

```solidity
// IF-THEN-ELSE operation on encrypted values
euint32 contribution = TFHE.select(
    encryptedVote,      // condition (encrypted bool)
    votingPower,        // if true
    TFHE.asEuint32(0)   // if false
);
```

**Use Case**: Add voting power to appropriate counter based on vote choice

**Properties**:
- No branches in plaintext
- Condition can be encrypted
- Both branches encrypted
- No information leakage

### 3. **TFHE.asEuint32() - Type Conversion**

```solidity
// Convert plaintext to encrypted form
euint32 encryptedPower = TFHE.asEuint32(
    boardMembers[msg.sender].votingPower
);
```

**Use Case**: Convert plaintext voting power to encrypted form for use in operations

### 4. **TFHE.asEbool() - Input Validation**

```solidity
// Convert validated encrypted input to encrypted bool
ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
```

**Use Case**: Validate and convert user's encrypted vote choice

## Complete Voting Operation Flow

### Step-by-Step Breakdown

```solidity
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,      // User's encrypted vote (encrypted)
    bytes calldata inputProof
) external onlyBoardMember {

    // Step 1: Validate and convert input (encrypted)
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
    // vote is now: encrypted boolean (true = yes, false = no)

    // Step 2: Get voter's power (still plaintext at this point)
    uint256 plainVotingPower = boardMembers[msg.sender].votingPower;

    // Step 3: Convert voting power to encrypted form (encrypted)
    euint32 encryptedPower = TFHE.asEuint32(plainVotingPower);
    // encryptedPower is now: encrypted uint32

    // Step 4: Select appropriate vote counter using encrypted conditional
    euint32 contribution = TFHE.select(
        vote,                      // encrypted condition
        encryptedPower,           // add to yesVotes if true
        TFHE.asEuint32(0)        // add 0 if false
    );
    // contribution is now: encrypted value (either power or 0, encrypted)

    // Step 5: Add contribution to appropriate counter (homomorphic addition)
    resolutions[_resolutionId].yesVotes = TFHE.add(
        resolutions[_resolutionId].yesVotes,
        contribution
    );
    // yesVotes is now: updated encrypted total

    // Step 6: Grant permissions for further operations
    FHE.allowThis(contribution);
    FHE.allow(contribution, msg.sender);
}
```

## Operation Chain Example

Voting scenario with 3 board members:

```
Initial State:
yesVotes = enc(0)
noVotes = enc(0)

Alice votes YES (power: 5):
  1. vote = enc(true)
  2. power = enc(5)
  3. contribution = select(enc(true), enc(5), enc(0)) = enc(5)
  4. yesVotes = add(enc(0), enc(5)) = enc(5)

Bob votes YES (power: 3):
  1. vote = enc(true)
  2. power = enc(3)
  3. contribution = select(enc(true), enc(3), enc(0)) = enc(3)
  4. yesVotes = add(enc(5), enc(3)) = enc(8)

Charlie votes NO (power: 2):
  1. vote = enc(false)
  2. power = enc(2)
  3. contribution = select(enc(false), enc(2), enc(0)) = enc(0)
  4. noVotes = add(enc(0), enc(0)) = enc(0)

Final State (encrypted):
yesVotes = enc(8)  // Never decrypted until voting ends!
noVotes = enc(0)   // Never decrypted until voting ends!
```

## Why These Operations?

### Privacy Preservation

```
Traditional Smart Contract:
yesVotes[0] = 0
yesVotes[0] += alice_choice == true ? 5 : 0    // Visible on-chain
yesVotes[0] = 5  // Everyone knows alice voted yes

FHEVM Smart Contract:
yesVotes = enc(0)
yesVotes = add(yesVotes, select(enc(alice_choice), enc(5), enc(0)))
// No one knows alice's vote - it's encrypted!
```

### Computational Security

All operations maintain encryption:

```solidity
// ✅ SECURE: All intermediate values encrypted
euint32 temp1 = TFHE.add(enc(a), enc(b));
euint32 temp2 = TFHE.select(enc_cond, temp1, enc(c));
euint32 result = TFHE.add(temp2, enc(d));
// result is encrypted, no information leaked

// ❌ INSECURE: Decryption breaks privacy
euint32 temp = TFHE.add(enc(a), enc(b));
uint32 plain = TFHE.decrypt(temp);  // NEVER DO THIS!
euint32 result = TFHE.add(TFHE.asEuint32(plain), enc(d));
// plain value is exposed!
```

## Advanced Patterns

### Pattern 1: Weighted Conditional Addition

```solidity
// Only add if condition is true AND weight is high
euint32 result = TFHE.select(
    TFHE.gt(encryptedVote, TFHE.asEuint32(threshold)),
    encryptedWeight,
    TFHE.asEuint32(0)
);
```

### Pattern 2: Accumulation with Multiple Conditions

```solidity
// Add different amounts based on vote type
euint32 contribution = TFHE.select(
    voteType,  // encrypted type indicator
    TFHE.asEuint32(majorityPower),    // if major vote
    TFHE.asEuint32(minorityPower)     // if minor vote
);

total = TFHE.add(total, contribution);
```

### Pattern 3: Safe Counter Updates

```solidity
// Multiple counters updated atomically
euint32 yesContribution = TFHE.select(vote, power, TFHE.asEuint32(0));
euint32 noContribution = TFHE.select(vote, TFHE.asEuint32(0), power);

yesVotes = TFHE.add(yesVotes, yesContribution);
noVotes = TFHE.add(noVotes, noContribution);

// Both updates happen with encrypted logic
// No plaintext branching!
```

## Performance Considerations

### Gas Costs by Operation

```
TFHE.add():       ~20,000 gas
TFHE.select():    ~30,000 gas
TFHE.asEuint32(): ~15,000 gas
TFHE.asEbool():   ~25,000 gas (with proof verification)

Total vote operation: ~90,000 gas
Plus base transaction: ~100,000 gas
Total: ~190,000 gas per encrypted operation
```

### Optimization Strategies

```solidity
// ✅ GOOD: Minimize operations
euint32 vote = TFHE.asEbool(input, proof);
euint32 contribution = TFHE.select(vote, power, TFHE.asEuint32(0));
total = TFHE.add(total, contribution);  // Single chain

// ❌ BAD: Unnecessary operations
euint32 vote = TFHE.asEbool(input, proof);
euint32 yes = TFHE.select(vote, power, TFHE.asEuint32(0));
euint32 no = TFHE.select(vote, TFHE.asEuint32(0), power);
euint32 neutral = TFHE.select(vote, TFHE.asEuint32(0), TFHE.asEuint32(0));
// Too many operations!
```

## Testing Homomorphic Operations

```typescript
it("✅ should correctly add encrypted votes", async function () {
    const encrypted1 = await createEncrypted(5);
    const encrypted2 = await createEncrypted(3);

    // Encrypt and add
    const tx = await contract.addEncrypted(encrypted1, encrypted2);
    await tx.wait();

    // Verify result through decryption
    const result = await contract.getResult();
    // Result should be enc(8) which decrypts to 8

    const decrypted = await fhevm.decrypt(result);
    expect(decrypted).to.equal(8);
});

it("✅ should correctly select based on encrypted condition", async function () {
    const condition = true;  // encrypted
    const ifTrue = 100;      // encrypted
    const ifFalse = 200;     // encrypted

    const tx = await contract.selectEncrypted(
        encryptCondition(condition),
        encryptValue(ifTrue),
        encryptValue(ifFalse)
    );
    await tx.wait();

    const result = await contract.getResult();
    const decrypted = await fhevm.decrypt(result);
    expect(decrypted).to.equal(100);  // condition was true
});
```

## Common Mistakes with Homomorphic Operations

### ❌ Mistake 1: Mixing Encrypted and Plaintext

```solidity
// DON'T: Mix encrypted and plaintext in operations
uint256 plain = 5;
euint32 encrypted = TFHE.asEuint32(10);
euint32 result = TFHE.add(encrypted, plain);  // Type error!

// DO: Convert everything to encrypted
euint32 result = TFHE.add(
    encrypted,
    TFHE.asEuint32(plain)
);
```

### ❌ Mistake 2: Decrypting Too Early

```solidity
// DON'T: Decrypt intermediate values
euint32 temp = TFHE.add(a, b);
uint256 plain = TFHE.decrypt(temp);  // Breaks privacy!
euint32 next = TFHE.add(TFHE.asEuint32(plain), c);

// DO: Keep encrypted throughout
euint32 temp = TFHE.add(a, b);
euint32 next = TFHE.add(temp, c);
```

### ❌ Mistake 3: Unnecessary Conversions

```solidity
// DON'T: Convert back and forth
euint32 a = TFHE.asEuint32(someValue);
uint32 plain = ???;  // Can't decrypt
euint32 b = TFHE.asEuint32(plain);

// DO: Keep encrypted form
euint32 a = TFHE.asEuint32(someValue);
euint32 b = TFHE.asEuint32(otherValue);
// Use directly in operations
```

## Summary

Homomorphic operations are the foundation of FHEVM privacy:

1. **TFHE.add()** - Combine encrypted values
2. **TFHE.select()** - Conditional logic on encrypted data
3. **TFHE.asEuint32()** - Convert to encrypted form
4. **TFHE.asEbool()** - Validate and convert encrypted inputs

Together, these operations enable complex computations while maintaining complete encryption throughout, preserving privacy end-to-end.

---

**Chapter**: homomorphic-operations
**Difficulty**: Intermediate-Advanced
**Prerequisites**: Understanding of FHEVM basics and encrypted types
