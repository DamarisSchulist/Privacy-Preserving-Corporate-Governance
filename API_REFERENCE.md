# API Reference - Corporate Governance Smart Contract

## Contract: CorporateGovernance

Location: `contracts/CorporateGovernance.sol`
Network: Ethereum Sepolia Testnet
Address: `0x13116d08546b78F5fDB7fA4544f778885B19A441`

---

## Data Types

### struct BoardMember

```solidity
struct BoardMember {
    bool isActive;           // Is member currently active
    uint256 votingPower;     // Member's voting weight
    string name;            // Member's display name
    string position;        // Member's board position
}
```

### struct Resolution

```solidity
struct Resolution {
    uint256 id;              // Unique resolution ID
    string title;            // Resolution title
    string description;      // Detailed description
    uint256 startTime;       // Voting start timestamp
    uint256 endTime;         // Voting end timestamp
    bool active;             // Is voting currently active
    euint32 yesVotes;       // Encrypted yes vote count
    euint32 noVotes;        // Encrypted no vote count
    address creator;         // Resolution creator
    uint256 requiredQuorum;  // Required voting power to pass
}
```

---

## State Variables

### Public Variables

```solidity
address public chairperson
```
The contract's chairperson (deployer). Only chairperson can remove board members.

```solidity
uint256 public totalVotingPower
```
Sum of all active board members' voting power.

```solidity
uint256 public resolutionCounter
```
Current number of resolutions created.

```solidity
uint256 public constant VOTING_DURATION = 7 days
```
Default voting period duration (7 days = 604,800 seconds).

### Public Mappings

```solidity
mapping(address => BoardMember) public boardMembers
```
Access: `boardMembers[address]`

```solidity
mapping(uint256 => Resolution) public resolutions
```
Access: `resolutions[resolutionId]`

```solidity
mapping(uint256 => mapping(address => bool)) public hasVoted
```
Access: `hasVoted[resolutionId][address]`

---

## Events

### BoardMemberAdded
```solidity
event BoardMemberAdded(
    address indexed member,
    string name,
    uint256 votingPower
)
```
Emitted when a board member is added or updated.

### BoardMemberRemoved
```solidity
event BoardMemberRemoved(address indexed member)
```
Emitted when a board member is removed.

### ResolutionCreated
```solidity
event ResolutionCreated(
    uint256 indexed resolutionId,
    string title,
    address indexed creator
)
```
Emitted when a resolution is created.

### VoteCast
```solidity
event VoteCast(
    uint256 indexed resolutionId,
    address indexed voter
)
```
Emitted when a vote is cast.

### ResolutionClosed
```solidity
event ResolutionClosed(
    uint256 indexed resolutionId,
    bool passed
)
```
Emitted when resolution voting is closed and results determined.

---

## Function Reference

### Board Member Management

#### addBoardMember()
```solidity
function addBoardMember(
    address _member,
    string memory _name,
    string memory _position,
    uint256 _votingPower
) external
```

**Parameters:**
- `_member`: Address of board member
- `_name`: Display name
- `_position`: Board position title
- `_votingPower`: Voting weight (must be > 0)

**Returns:** Nothing

**Requirements:**
- `_votingPower > 0`

**Events:** `BoardMemberAdded`

**Example:**
```javascript
const tx = await contract.addBoardMember(
    "0xAliceAddress",
    "Alice",
    "Chief Executive Officer",
    5
);
await tx.wait();
```

#### removeBoardMember()
```solidity
function removeBoardMember(address _member) external
```

**Parameters:**
- `_member`: Address to remove

**Requirements:**
- Only chairperson can call
- Member must be active
- Cannot remove chairperson

**Events:** `BoardMemberRemoved`

**Example:**
```javascript
const tx = await contract.removeBoardMember("0xAliceAddress");
await tx.wait();
```

#### getBoardMember()
```solidity
function getBoardMember(address _member)
    external view
    returns (
        bool isActive,
        uint256 votingPower,
        string memory name,
        string memory position
    )
```

**Parameters:**
- `_member`: Member address

**Returns:**
- `isActive`: Is member currently active
- `votingPower`: Member's voting weight
- `name`: Display name
- `position`: Board position

**Example:**
```javascript
const member = await contract.getBoardMember("0xAliceAddress");
console.log(member.votingPower);  // 5
```

### Resolution Management

#### createResolution()
```solidity
function createResolution(
    string memory _title,
    string memory _description,
    uint256 _requiredQuorum
) external
```

**Parameters:**
- `_title`: Resolution title
- `_description`: Detailed description
- `_requiredQuorum`: Required voting power to pass

**Requirements:**
- `_requiredQuorum > 0`
- `_requiredQuorum <= totalVotingPower`

**Events:** `ResolutionCreated`

**Behavior:**
- Auto-adds caller as board member if not already
- Sets voting period to 7 days
- Creates new resolution with ID = resolutionCounter

**Example:**
```javascript
const tx = await contract.createResolution(
    "Approve Q4 Budget",
    "Annual budget allocation for Q4",
    5  // quorum
);
await tx.wait();
const resId = await contract.getResolutionCount() - 1;
```

#### getResolution()
```solidity
function getResolution(uint256 _resolutionId)
    external view
    returns (
        uint256 id,
        string memory title,
        string memory description,
        uint256 startTime,
        uint256 endTime,
        bool active,
        address creator,
        uint256 requiredQuorum
    )
```

**Parameters:**
- `_resolutionId`: Resolution ID

**Returns:**
- Resolution metadata (not encrypted vote counts)

**Note:** Encrypted vote counts cannot be returned from view functions

**Example:**
```javascript
const res = await contract.getResolution(0);
console.log(res.title);      // "Approve Q4 Budget"
console.log(res.active);     // true or false
console.log(res.endTime);    // timestamp
```

### Voting

#### castVote()
```solidity
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,
    bytes calldata inputProof
) external
```

**Parameters:**
- `_resolutionId`: Which resolution to vote on
- `_encryptedVote`: Encrypted vote choice (true = yes, false = no)
- `inputProof`: Zero-knowledge proof validating encryption

**Requirements:**
- Caller must be board member (or auto-added)
- Resolution must be active
- Voting period must not have ended
- Valid input proof required

**Events:** `VoteCast`

**Gas:** ~400,000 gas (FHE operations expensive)

**Example:**
```typescript
// Create encrypted vote
const encrypted = await fhevm
    .createEncryptedInput(contractAddress, voterAddress)
    .add1(1)  // 1 = yes vote
    .encrypt();

// Submit vote
const tx = await contract.castVote(
    0,                           // resolution ID
    encrypted.handles[0],        // encrypted vote
    encrypted.inputProof,        // proof
    { gasLimit: 500000 }        // adequate gas
);
await tx.wait();
```

#### closeResolution()
```solidity
function closeResolution(uint256 _resolutionId) external
```

**Parameters:**
- `_resolutionId`: Which resolution to close

**Requirements:**
- Resolution must be active
- Either voting period ended OR caller is creator

**Events:** `ResolutionClosed` (via gateway callback)

**Behavior:**
- Marks resolution as inactive
- Requests decryption from gateway
- Gateway will call `resolveResolution()` callback

**Example:**
```javascript
// Wait for voting period to end
await ethers.provider.send("hardhat_mine", ["0x" + (7*24*3600).toString(16)]);

// Close resolution
const tx = await contract.closeResolution(0);
await tx.wait();
```

#### resolveResolution()
```solidity
function resolveResolution(
    uint256 _resolutionId,
    uint256[] memory decryptedVotes
) public onlyGateway
```

**Parameters:**
- `_resolutionId`: Which resolution
- `decryptedVotes`: Decrypted [yesCount, noCount]

**Requirements:**
- Only gateway can call (onlyGateway modifier)

**Events:** `ResolutionClosed` (emitted by this function)

**Note:** Called automatically by gateway after decryption

### Utilities

#### getTotalVotingPower()
```solidity
function getTotalVotingPower() external view returns (uint256)
```

**Returns:** Sum of all active board members' voting power

**Example:**
```javascript
const power = await contract.getTotalVotingPower();
console.log(`Total voting power: ${power}`);
```

#### getResolutionCount()
```solidity
function getResolutionCount() external view returns (uint256)
```

**Returns:** Total number of resolutions created

**Example:**
```javascript
const count = await contract.getResolutionCount();
console.log(`Total resolutions: ${count}`);
```

---

## Modifiers

### onlyChairperson()
```solidity
modifier onlyChairperson()
```
Only chairperson can call this function.

### onlyBoardMember()
```solidity
modifier onlyBoardMember()
```
Only active board members can call. However, non-members can be auto-added in some functions.

### resolutionExists()
```solidity
modifier resolutionExists(uint256 _resolutionId)
```
Ensures resolution with given ID exists.

---

## Error Messages

### Access Control
- `"Only chairperson can perform this action"`
- `"Only active board members can perform this action"`
- `"Not a board member"`

### Board Member Management
- `"Voting power must be greater than 0"`
- `"Member is not active"`
- `"Cannot remove chairperson"`

### Resolution Management
- `"Quorum cannot exceed total voting power"`
- `"Quorum must be greater than 0"`
- `"Resolution does not exist"`

### Voting
- `"Resolution is not active"`
- `"Voting period has ended"`

---

## Usage Examples

### Complete Voting Workflow

```typescript
const hre = require("hardhat");
const { ethers } = hre;

async function main() {
    const [deployer] = await ethers.getSigners();

    // Get contract
    const contract = await ethers.getContractAt(
        "CorporateGovernance",
        "0x13116d08546b78F5fDB7fA4544f778885B19A441"
    );

    // Step 1: Add board members
    await contract.addBoardMember(deployer.address, "CEO", "Chief", 10);
    console.log("✅ Added board member");

    // Step 2: Create resolution
    const tx1 = await contract.createResolution(
        "Approve Budget",
        "Q4 budget allocation",
        5
    );
    await tx1.wait();
    console.log("✅ Created resolution");

    // Step 3: Cast encrypted vote
    const encrypted = await hre.fhevm
        .createEncryptedInput("0x...", deployer.address)
        .add1(1)
        .encrypt();

    const tx2 = await contract.castVote(
        0,
        encrypted.handles[0],
        encrypted.inputProof,
        { gasLimit: 500000 }
    );
    await tx2.wait();
    console.log("✅ Cast encrypted vote");

    // Step 4: Close resolution
    const tx3 = await contract.closeResolution(0);
    await tx3.wait();
    console.log("✅ Closed resolution");
}

main().catch(console.error);
```

---

## Contract Constants

```
VOTING_DURATION = 7 days (604,800 seconds)
```

---

**Last Updated**: December 2025
**Contract Version**: 1.0.0
**FHEVM Compatible**: Yes (0.4.x+)
