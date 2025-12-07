# Privacy-Preserving Corporate Governance - FHEVM Example

A fully homomorphic encryption (FHE) powered board resolution and voting system demonstrating advanced FHEVM concepts including encrypted state management, input proofs, homomorphic operations, access control, and gateway-based decryption.

## 📋 Overview

This FHEVM example demonstrates a **confidential corporate governance system** where:
- Board members cast **completely private votes** using FHE encryption
- Individual votes remain encrypted throughout the process and are never revealed
- Vote tallies are computed **homomorphically** without ever decrypting intermediate values
- Only **final results are revealed** when voting concludes
- **Complete audit trail** maintained on-chain for transparency

**Live Demo**: https://fhe-corporate-governance.vercel.app/

**Contract Address (Sepolia)**: `0x13116d08546b78F5fDB7fA4544f778885B19A441`

## ✨ Key FHEVM Concepts Demonstrated

### 1. **Encrypted State Management** (chapter: encryption)
```solidity
// Private vote counters remain encrypted throughout voting period
euint32 yesVotes;   // Encrypted yes vote tally
euint32 noVotes;    // Encrypted no vote tally
```
**Concept**: Sensitive data stored as encrypted types and never decrypted until final results

---

### 2. **Input Encryption & Proof Validation** (chapter: input-proofs)
```solidity
function castVote(
    uint256 _resolutionId,
    einput _encryptedVote,      // User-encrypted vote
    bytes calldata inputProof    // Zero-knowledge proof
) external {
    // Verify and convert encrypted input
    ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
    // ...
}
```
**Concept**: Secure input encryption with cryptographic proof validation

**Common Pitfall**: ❌ Missing or invalid input proof validation
```solidity
// ❌ DON'T: No proof validation
ebool vote = TFHE.asEbool(_encryptedVote, new bytes(0));

// ✅ DO: Always validate proof
ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
```

---

### 3. **Homomorphic Operations** (chapter: fhe-operations)
```solidity
// Add voting power to appropriate counter (homomorphically)
euint32 votingPower = TFHE.asEuint32(boardMembers[msg.sender].votingPower);

// Encrypted conditional addition - core FHE operation
resolution.yesVotes = TFHE.add(
    resolution.yesVotes,
    TFHE.select(vote, votingPower, TFHE.asEuint32(0))  // IF vote THEN add power ELSE 0
);
```
**Concept**: Perform operations on encrypted data without decryption
- `TFHE.add()` - Homomorphic addition
- `TFHE.select()` - Encrypted conditional (if-else)
- All operations preserve encryption

---

### 4. **Access Control with FHE** (chapter: access-control)
```solidity
modifier onlyBoardMember() {
    require(boardMembers[msg.sender].isActive, "Only active board members");
    _;
}

// Only authorized members can vote
function castVote(...) external onlyBoardMember {
    // Encrypted operations within access-controlled functions
    FHE.allowThis(encryptedValue);        // Grant contract permission
    FHE.allow(encryptedValue, msg.sender); // Grant user permission
}
```
**Concept**: Combine traditional access control with FHE permission system
- Contract permissions: `FHE.allowThis()`
- User permissions: `FHE.allow(address, user)`

**Common Pitfall**: ❌ Missing permission grants
```solidity
// ❌ DON'T: Only grant one permission
FHE.allow(encryptedValue, msg.sender);  // Missing allowThis()

// ✅ DO: Grant both permissions
FHE.allowThis(encryptedValue);
FHE.allow(encryptedValue, msg.sender);
```

---

### 5. **Gateway-Based Decryption** (chapter: decryption)
```solidity
function closeResolution(uint256 _resolutionId) external {
    // Request async decryption of final results
    uint256[] memory cts = new uint256[](2);
    cts[0] = Gateway.toUint256(resolution.yesVotes);
    cts[1] = Gateway.toUint256(resolution.noVotes);

    Gateway.requestDecryption(
        cts,
        this.resolveResolution.selector,  // Callback function
        0,
        block.timestamp + 100,
        false
    );
}

// Gateway calls callback with decrypted results
function resolveResolution(uint256, uint256[] memory decryptedVotes)
    public onlyGateway {
    uint256 yesVotes = decryptedVotes[0];
    uint256 noVotes = decryptedVotes[1];
    bool passed = yesVotes > noVotes;
    emit ResolutionClosed(0, passed);
}
```
**Concept**: Asynchronous decryption pattern
1. Request decryption of final results
2. Gateway decrypts and calls callback
3. Process results with full transparency

---

## 🚀 Quick Start

### Prerequisites
```bash
Node.js >= 18.0.0
npm >= 8.0.0
Git
```

### Installation & Setup

```bash
# Clone the repository
git clone https://github.com/DamarisSchulist/CorporateGovernance.git
cd CorporateGovernance

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Add your PRIVATE_KEY and INFURA_PROJECT_ID
```

### Compile & Test

```bash
# Compile Solidity contracts
npx hardhat compile

# Run test suite
npx hardhat test

# Run tests with gas reporting
REPORT_GAS=true npx hardhat test

# Generate coverage report
npx hardhat coverage
```

### Deploy

```bash
# Deploy to Sepolia testnet
npx hardhat run scripts/deploy-corporate-governance.js --network sepolia

# Verify contract on Etherscan
npx hardhat verify --network sepolia <CONTRACT_ADDRESS>
```

### Run Locally

```bash
# Start local FHEVM-ready node
npx hardhat node

# Deploy to local network (in another terminal)
npx hardhat run scripts/deploy-corporate-governance.js --network localhost
```

## 📁 Project Structure

```
corporate-governance/
├── contracts/                              # Smart contracts
│   ├── CorporateGovernance.sol            # Main FHE implementation
│   ├── SimpleCorporateGovernance.sol      # Non-FHE version for comparison
│   ├── SimpleBoardResolution.sol          # Simplified example
│   └── UltraSimpleVoting.sol             # Minimal voting demo
├── test/                                  # Test suite
│   └── CorporateGovernance.ts            # Comprehensive tests
├── scripts/
│   └── deploy-corporate-governance.js     # Deployment script
├── frontend/                              # Web interface
│   ├── index.html
│   ├── package.json
│   └── server.js
├── docs/                                  # Documentation
│   ├── fhe-counter.md                    # FHE concepts
│   └── SUMMARY.md                        # GitBook index
├── hardhat.config.ts                     # Hardhat configuration
├── package.json
└── README.md
```

## 🔍 Example Walkthrough

### Creating a Resolution

```javascript
// Resolution creator specifies voting parameters
const tx = await contract.createResolution(
    "Approve Q4 Budget",
    "Resolution to approve quarterly budget allocation",
    5  // Required quorum (sum of voting power needed to pass)
);
await tx.wait();
```

### Casting an Encrypted Vote

```typescript
// 1. User encrypts their vote client-side
const encryptedInput = await fhevm
    .createEncryptedInput(contractAddress, userAddress)
    .add1(1)  // 1 = yes vote, 0 = no vote
    .encrypt();

// 2. Submit encrypted vote to contract
const tx = await contract.castVote(
    resolutionId,
    encryptedInput.handles[0],
    encryptedInput.inputProof,
    { gasLimit: 500000 }
);
await tx.wait();

// 3. Individual vote remains encrypted on-chain
// Other board members cannot see how you voted
```

### Closing Resolution & Revealing Results

```typescript
// After voting period ends, close the resolution
const tx = await contract.closeResolution(resolutionId);
await tx.wait();

// Gateway decrypts final tallies and calls resolveResolution()
// Only final yes/no counts are revealed, never individual votes
contract.on('ResolutionClosed', (resolutionId, passed) => {
    console.log(`Resolution ${resolutionId} ${passed ? 'PASSED' : 'FAILED'}`);
});
```

## 📚 Core Concepts Explanation

### FHE Encryption Binding

FHEVM uses **encryption binding** where values are bound to `[contract, user]` pairs:

```
Before submitting to contract:
┌─────────────────────────────┐
│ User's Local Environment    │
│ • User has private key      │
│ • FHE encrypts vote         │
│ • Creates proof of binding  │
└──────────┬──────────────────┘
           │ Submit (encrypted)
           ▼
┌─────────────────────────────┐
│ Smart Contract              │
│ • Verify binding proof      │
│ • Process encrypted vote    │
│ • Add to encrypted tally    │
└─────────────────────────────┘
```

### Vote Processing Flow

```
1. User selects vote (YES/NO)
   ↓
2. Client encrypts vote with FHE
   ↓
3. Creates zero-knowledge proof
   ↓
4. Submits encrypted vote + proof to contract
   ↓
5. Contract verifies proof signature
   ↓
6. Contract converts to encrypted boolean: ebool vote
   ↓
7. Homomorphic addition: yesVotes += vote ? votingPower : 0
   ↓
8. Vote stored encrypted on-chain
   ↓
9. After voting period:
   - Request decryption of final tallies
   - Gateway decrypts results
   - Contract processes final outcome
   - Individual votes never decrypted!
```

## ⚙️ Configuration & Gas Costs

### Gas Optimization

FHE operations are computationally expensive. This contract optimizes by:

```solidity
// ✅ GOOD: Single conditional FHE operation
resolution.yesVotes = TFHE.add(
    resolution.yesVotes,
    TFHE.select(vote, votingPower, TFHE.asEuint32(0))
);

// ❌ BAD: Multiple separate FHE operations
if (TFHE.decrypt(vote)) {
    resolution.yesVotes = TFHE.add(resolution.yesVotes, votingPower);
} else {
    resolution.noVotes = TFHE.add(resolution.noVotes, votingPower);
}
```

### Typical Gas Costs
- Create resolution: ~200,000 gas
- Cast vote: ~400,000 gas
- Close resolution: ~150,000 gas
- Decrypt callback: ~100,000 gas

## 🔒 Security Analysis

### Privacy Guarantees

✅ **What's Private:**
- Individual votes encrypted at submission
- Never decrypted during voting period
- Even contract creator cannot see individual votes
- FHE operations preserve encryption throughout

✅ **What's Public:**
- Resolution details (title, description, timing)
- Board member identities and voting power
- Final yes/no vote counts (only after closure)
- All transactions recorded on-chain

### Common Pitfalls & Solutions

**❌ Pitfall 1: Missing Input Proof Validation**
```solidity
// DON'T do this:
ebool vote = TFHE.asEbool(_encryptedVote, new bytes(0));

// DO this instead:
ebool vote = TFHE.asEbool(_encryptedVote, inputProof);
```

**❌ Pitfall 2: Decrypting in View Functions**
```solidity
// DON'T expose encrypted data in view functions:
function getEncryptedVotes() external view returns (euint32) {
    return resolution.yesVotes;  // ❌ Never decrypt here
}

// DO only decrypt in gateway callbacks:
function resolveResolution(uint256, uint256[] memory decryptedVotes)
    public onlyGateway {
    // ✅ Safe to process decrypted results here
}
```

**❌ Pitfall 3: Forgetting Permission Grants**
```solidity
// DON'T: Only grant user permission
FHE.allow(encryptedValue, msg.sender);

// DO: Grant both contract and user permissions
FHE.allowThis(encryptedValue);
FHE.allow(encryptedValue, msg.sender);
```

**❌ Pitfall 4: Vote Timing Issues**
```solidity
// DON'T: Allow votes after period ends
require(block.timestamp <= resolution.endTime, "Voting period ended");

// DO: Enforce strict voting window
require(block.timestamp >= resolution.startTime, "Voting not started");
require(block.timestamp <= resolution.endTime, "Voting period ended");
```

## 🧪 Testing Strategy

### Unit Tests
- ✅ Resolution creation with valid/invalid quorum
- ✅ Board member addition and removal
- ✅ Encrypted vote submission and validation
- ✅ Vote tallying with different voting powers
- ✅ Decryption callback handling
- ✅ Access control enforcement
- ✅ Time-based voting restrictions

### Integration Tests
- ✅ Complete voting workflow end-to-end
- ✅ Multiple board members voting sequence
- ✅ Gateway interaction for decryption
- ✅ Event emission verification
- ✅ State transitions (pending → voting → closed)

### Edge Cases Covered
- ❌ Non-board-members cannot vote
- ❌ Cannot vote after period ends
- ❌ Cannot vote twice on same resolution
- ❌ Invalid quorum values rejected
- ❌ Missing access control blocks operations

## 🎯 Real-World Use Cases

### Corporate Governance
- Board of directors voting on strategic decisions
- Executive compensation approvals
- Merger and acquisition votes
- Policy change ratifications

### Organizational Decision Making
- Committee confidential voting
- Shareholder resolutions
- Multi-signature approvals with privacy
- Confidential personnel decisions

### Compliance & Regulation
- Regulatory-compliant private voting
- Audit committee decisions with confidentiality
- Whistleblower voting systems
- Confidential risk assessments

## 📊 Advanced Features

### Weighted Voting
```solidity
// Board members have different voting power based on role/shareholding
struct BoardMember {
    bool isActive;
    uint256 votingPower;  // Different power per member
    string name;
    string position;
}

// Voting power included in encrypted calculation:
euint32 votingPower = TFHE.asEuint32(boardMembers[msg.sender].votingPower);
resolution.yesVotes = TFHE.add(
    resolution.yesVotes,
    TFHE.select(vote, votingPower, TFHE.asEuint32(0))
);
```

### Flexible Quorum
```solidity
// Each resolution can have different quorum requirements
function createResolution(
    string memory _title,
    string memory _description,
    uint256 _requiredQuorum  // Custom per resolution
) external {
    require(_requiredQuorum <= totalVotingPower, "Quorum exceeds voting power");
    // ...
}
```

### Time-Bound Voting
```solidity
uint256 public constant VOTING_DURATION = 7 days;

function createResolution(...) external {
    Resolution storage resolution = resolutions[resolutionId];
    resolution.startTime = block.timestamp;
    resolution.endTime = block.timestamp + VOTING_DURATION;
}

function castVote(...) external {
    require(block.timestamp <= resolution.endTime, "Voting period has ended");
    // ...
}
```

## 📖 Documentation

- **README.md** - This file
- **QUICK_START.md** - 5-minute setup guide
- **HELLO_FHEVM_TUTORIAL.md** - Complete beginner tutorial
- **VIDEO_SCRIPT.md** - 60-second demo video script
- **VIDEO_DIALOGUE.md** - Voiceover transcript
- **COMPETITION_SUBMISSION.md** - Bounty submission summary

## 🔗 Links & Resources

### FHEVM Documentation
- **Official Docs**: https://docs.zama.ai/fhevm
- **API Reference**: https://docs.zama.ai/fhevm/fundamentals/types
- **GitHub**: https://github.com/zama-ai/fhevm

### This Project
- **Live Demo**: https://fhe-corporate-governance.vercel.app/
- **GitHub**: https://github.com/DamarisSchulist/CorporateGovernance
- **Contract (Sepolia)**: https://sepolia.etherscan.io/address/0x13116d08546b78F5fDB7fA4544f778885B19A441

### Development Tools
- **Hardhat**: https://hardhat.org
- **Ethers.js**: https://docs.ethers.org
- **FHEVM Hardhat Plugin**: https://www.npmjs.com/package/@fhevm/hardhat-plugin

## 🤝 Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Add tests for new functionality
4. Ensure all tests pass (`npm run test`)
5. Submit a pull request

## 📝 License

BSD-3-Clause-Clear License - See LICENSE file for details

## 🙏 Acknowledgments

Built with [FHEVM](https://www.zama.ai/fhevm) by Zama - bringing fully homomorphic encryption to smart contracts.

---

## 📊 Example Tags (for GitBook)

- **chapter: access-control** - Access control implementation
- **chapter: encryption** - Encrypted state management
- **chapter: fhe-operations** - Homomorphic operations
- **chapter: input-proofs** - Input proof validation
- **chapter: decryption** - Gateway decryption patterns
- **chapter: governance** - Governance use cases
- **chapter: privacy** - Privacy guarantees

**Difficulty Level**: Intermediate-Advanced
**FHEVM Version**: 0.4.x compatible
**Solidity Version**: ^0.8.24

---

**Built with ❤️ using [FHEVM](https://www.zama.ai/fhevm) by Zama**
