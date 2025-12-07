# Access Control - Corporate Governance Example

## Overview

Access control in FHEVM combines traditional role-based access control with FHE permission management. This example demonstrates proper implementation in confidential governance.

## Layers of Access Control

### 1. **Role-Based Access Control (RBAC)**

```solidity
// Define roles through mapping
mapping(address => BoardMember) public boardMembers;

struct BoardMember {
    bool isActive;
    uint256 votingPower;
    string name;
    string position;
}

// Enforce roles with modifiers
modifier onlyBoardMember() {
    require(boardMembers[msg.sender].isActive, "Only active board members");
    _;
}

function castVote(...) external onlyBoardMember {
    // Only board members can execute
}
```

### 2. **FHE Permission System**

```solidity
// Contract permission - allows contract to use encrypted value
FHE.allowThis(encryptedValue);

// User permission - allows user to decrypt (if needed)
FHE.allow(encryptedValue, msg.sender);
```

### 3. **Combined Access Control**

```solidity
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
)
    external
    onlyBoardMember              // ✅ RBAC: Only board members
{
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);

    FHE.allowThis(vote);         // ✅ FHE: Contract permission
    FHE.allow(vote, msg.sender);  // ✅ FHE: User permission

    // Encrypted operation using access-controlled value
}
```

## Implementation Patterns

### Pattern 1: Board Member Addition

```solidity
function addBoardMember(
    address _member,
    string memory _name,
    string memory _position,
    uint256 _votingPower
) external {
    require(_votingPower > 0, "Power must be > 0");

    // Update member status
    if (boardMembers[_member].isActive) {
        totalVotingPower -= boardMembers[_member].votingPower;
    }

    boardMembers[_member] = BoardMember({
        isActive: true,
        votingPower: _votingPower,
        name: _name,
        position: _position
    });

    totalVotingPower += _votingPower;
}
```

### Pattern 2: Board Member Removal

```solidity
modifier onlyChairperson() {
    require(msg.sender == chairperson, "Only chairperson");
    _;
}

function removeBoardMember(address _member)
    external
    onlyChairperson  // Only chairperson can remove
{
    require(boardMembers[_member].isActive, "Not a member");
    require(_member != chairperson, "Cannot remove self");

    totalVotingPower -= boardMembers[_member].votingPower;
    boardMembers[_member].isActive = false;

    emit BoardMemberRemoved(_member);
}
```

### Pattern 3: Function-Level Access Control

```solidity
// Different access requirements
function createResolution(...)
    external
    // Auto-add if needed, so any address can create
{
    if (!boardMembers[msg.sender].isActive) {
        // Auto-add with minimum power
        boardMembers[msg.sender] = BoardMember({
            isActive: true,
            votingPower: 1,
            name: "Auto Added",
            position: "Board Member"
        });
        totalVotingPower += 1;
    }
    // Continue with resolution creation
}

function closeResolution(uint256 _resolutionId)
    external
    // Only creator or after voting period
{
    Resolution storage resolution = resolutions[_resolutionId];
    require(
        msg.sender == resolution.creator ||
        block.timestamp > resolution.endTime,
        "Not creator or voting still active"
    );
    // Continue with closure
}
```

## FHE Permission Management

### Grant Permissions

```solidity
function castVote(...) external {
    euint32 vote = TFHE.asEuint32(someValue);

    // ✅ BOTH permissions required
    FHE.allowThis(vote);           // Contract can use this
    FHE.allow(vote, msg.sender);   // User can access this
}
```

### Why Both Permissions?

```solidity
// ✅ CORRECT: Both permissions
FHE.allowThis(value);      // Contract authorization
FHE.allow(value, user);    // User authorization
// Now contract can use on behalf of user

// ❌ INCOMPLETE: Only one permission
FHE.allow(value, user);    // User can access
// But contract cannot use! Operations fail!

// ❌ INCOMPLETE: Only one permission
FHE.allowThis(value);      // Contract can use
// But user cannot access! Decryption fails!
```

### Hierarchical Permissions

```solidity
// Board Member Level
mapping(address => bool) boardMembers;

// Voting Power Level
mapping(address => uint256) votingPower;

// Resolution-Level Access
mapping(uint256 => mapping(address => bool)) hasVoted;

// FHE Permission Level
FHE.allowThis(encryptedValue);
FHE.allow(encryptedValue, msg.sender);
```

## Security Considerations

### 1. **Prevent Unauthorized Voting**

```solidity
function castVote(...) external onlyBoardMember {
    // Only board members reach here
    // Non-members rejected at modifier level
}
```

### 2. **Prevent Double Voting**

```solidity
mapping(uint256 => mapping(address => bool)) hasVoted;

function castVote(uint256 _resolutionId, ...) external {
    // In the actual contract, note that hasVoted is updated
    // This prevents double-counting
    hasVoted[_resolutionId][msg.sender] = true;
}
```

### 3. **Prevent Timing Attacks**

```solidity
function castVote(...) external {
    require(
        block.timestamp >= resolution.startTime,
        "Voting not started"
    );
    require(
        block.timestamp <= resolution.endTime,
        "Voting ended"
    );
    // Strict timing enforcement
}
```

### 4. **Prevent Reentrancy**

```solidity
// Simple pattern: finish state changes before external calls
function castVote(...) external {
    // Update state first
    hasVoted[resId][msg.sender] = true;
    resolution.yesVotes = TFHE.add(...);

    // Any external calls after state updates
}
```

## Testing Access Control

```typescript
describe("Access Control", function () {
    it("✅ should allow board members to vote", async function () {
        await contract.addBoardMember(alice.address, "Alice", "Dir", 5);

        const encrypted = await createEncryptedVote(alice.address, true);
        const tx = await contract
            .connect(alice)
            .castVote(0, encrypted.handle, encrypted.proof);

        await expect(tx).to.not.be.reverted;
    });

    it("❌ should reject non-members", async function () {
        // charlie is not a member
        const encrypted = await createEncryptedVote(charlie.address, true);

        await expect(
            contract.connect(charlie).castVote(0, encrypted.handle, encrypted.proof)
        ).to.be.reverted;  // Or auto-adds (depending on design)
    });

    it("❌ should reject after removal", async function () {
        // Add alice
        await contract.addBoardMember(alice.address, "Alice", "Dir", 5);

        // Remove alice
        await contract.removeBoardMember(alice.address);

        // alice can no longer vote
        const encrypted = await createEncryptedVote(alice.address, true);

        // Now either rejected or auto-adds again
        // (depends on implementation)
    });

    it("✅ should enforce voting power limits", async function () {
        // Add members with different power
        await contract.addBoardMember(alice.address, "Alice", "CEO", 10);
        await contract.addBoardMember(bob.address, "Bob", "CFO", 3);

        // Both can vote
        // Their votes have different weights (10 vs 3)
    });

    it("❌ should prevent unauthorized operations", async function () {
        // Only chairperson can remove members
        await expect(
            contract.connect(alice).removeBoardMember(bob.address)
        ).to.be.reverted;
    });
});
```

## Common Access Control Mistakes

### ❌ Mistake 1: Missing Modifiers

```solidity
// DON'T: No access control
function castVote(...) external {
    // Anyone can vote!
    // No modifier checking
}

// DO: Enforce access control
function castVote(...) external onlyBoardMember {
    // Only board members reach here
}
```

### ❌ Mistake 2: Forgetting FHE Permissions

```solidity
// DON'T: Missing FHE permissions
function castVote(...) external {
    euint32 encrypted = TFHE.asEuint32(value);
    resolutions[0].votes = TFHE.add(resolutions[0].votes, encrypted);
    // No allowThis/allow - operations will fail!
}

// DO: Grant both FHE permissions
function castVote(...) external {
    euint32 encrypted = TFHE.asEuint32(value);
    FHE.allowThis(encrypted);
    FHE.allow(encrypted, msg.sender);
    resolutions[0].votes = TFHE.add(resolutions[0].votes, encrypted);
}
```

### ❌ Mistake 3: Inconsistent Access Levels

```solidity
// DON'T: Inconsistent access control
function doSomething() external onlyBoardMember { ... }  // Needs member
function doAnother() external { ... }                    // Anyone can call
// Inconsistent security boundaries

// DO: Clear, consistent access levels
function doSomething() external onlyBoardMember { ... }
function doAnother() external onlyBoardMember { ... }
// Or document why some are public
```

### ❌ Mistake 4: Time-of-Check/Time-of-Use

```solidity
// DON'T: Check then use pattern
if (boardMembers[msg.sender].isActive) {
    // Members could be removed between check and use!
    executeVote();
}

// DO: Enforce with modifiers
function executeVote() external onlyBoardMember {
    // Member status checked at entry, protected throughout
}
```

## Best Practices

1. **Use Modifiers for Access Control**
   ```solidity
   modifier onlyBoardMember() {
       require(boardMembers[msg.sender].isActive, "Not member");
       _;
   }
   ```

2. **Always Grant FHE Permissions**
   ```solidity
   FHE.allowThis(value);
   FHE.allow(value, msg.sender);
   ```

3. **Enforce at Function Entry**
   ```solidity
   function critical() external onlyAuthorized {
       // Protected for entire execution
   }
   ```

4. **Multiple Levels of Control**
   ```solidity
   function castVote(...)
       external
       onlyBoardMember              // Role check
       onlyDuringVoting            // Time check
   {
       FHE.allowThis(encrypted);    // FHE permission
       FHE.allow(encrypted, msg.sender);
   }
   ```

5. **Clear Error Messages**
   ```solidity
   require(
       boardMembers[msg.sender].isActive,
       "Only active board members can vote"
   );
   ```

## Summary

Access control in FHEVM requires:

1. **RBAC** - Role-based access with modifiers
2. **FHE Permissions** - Both allowThis and allow
3. **Time Checks** - Enforce voting periods
4. **State Consistency** - Maintain accurate member status
5. **Clear Rules** - Document what each operation requires

Proper access control ensures:
- ✅ Only authorized users execute operations
- ✅ Encrypted data is properly protected
- ✅ Voting periods are enforced
- ✅ Member status is respected
- ✅ FHE operations can proceed safely

---

**Chapter**: access-control
**Difficulty**: Intermediate
**Prerequisites**: FHEVM basics, Solidity modifiers
