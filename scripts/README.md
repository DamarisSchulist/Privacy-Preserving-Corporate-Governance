# Scripts Documentation

This directory contains deployment and utility scripts for the Corporate Governance FHEVM example.

## 📜 Available Scripts

### deploy-corporate-governance.js

**Purpose:** Deploy CorporateGovernance contract to blockchain

**Location:** `scripts/deploy-corporate-governance.js`

**Usage:**
```bash
# Deploy to localhost
npx hardhat run scripts/deploy-corporate-governance.js --network localhost

# Deploy to Sepolia testnet
npx hardhat run scripts/deploy-corporate-governance.js --network sepolia

# Deploy to Zama (if available)
npx hardhat run scripts/deploy-corporate-governance.js --network zama
```

**What it does:**
1. Gets deployer account
2. Deploys CorporateGovernance contract
3. Outputs contract address
4. (Optional) Verifies on Etherscan
5. Returns contract instance

**Output:**
```
Deploying CorporateGovernance to Sepolia...
✅ CorporateGovernance deployed to: 0x13116d08546b78F5fDB7fA4544f778885B19A441
Transaction hash: 0xabc123...
```

**Configuration:**
```javascript
const CorporateGovernance = await ethers.getContractFactory("CorporateGovernance");
const contract = await CorporateGovernance.deploy();
await contract.waitForDeployment();
```

---

## 🛠️ NPM Scripts

Defined in `package.json`:

```bash
npm run compile              # Compile all contracts
npm run test                # Run test suite
npm run test:gas            # Run tests with gas reporting
npm run coverage            # Generate coverage report
npm run deploy:localhost    # Deploy to local node
npm run deploy:sepolia      # Deploy to Sepolia testnet
npm run deploy:zama         # Deploy to Zama network (if available)
npm run verify              # Verify contract on Etherscan
npm run lint                # Run linting checks
npm run clean               # Clean build artifacts
npm run node                # Start local Hardhat node
npm run typechain           # Generate TypeChain types
npm run build               # Compile and generate types
npm run dev                 # Start local development node
```

---

## 📋 Deployment Checklist

Before deploying, verify:

```
✅ Environment configured (.env exists)
✅ Private key set (PRIVATE_KEY in .env)
✅ Testnet ETH obtained (for Sepolia)
✅ Node.js >= 18
✅ Dependencies installed (npm install)
✅ Contracts compile (npm run compile)
✅ Tests pass (npm run test)
```

---

## 🔍 Deployment Steps Explained

### Step 1: Environment Setup
```bash
cp .env.example .env
# Edit .env with your credentials
```

### Step 2: Compile Contracts
```bash
npm run compile
# Generates artifacts/ folder with ABI and bytecode
```

### Step 3: Run Local Tests
```bash
npm run test
# Ensures everything works before deployment
```

### Step 4: Deploy
```bash
npx hardhat run scripts/deploy-corporate-governance.js --network sepolia
# Contract deployed, address printed
```

### Step 5: Verify (Optional)
```bash
npx hardhat verify --network sepolia 0x13116d08546b78F5fDB7fA4544f778885B19A441
# Contract source visible on Etherscan
```

---

## 📊 Script Output Examples

### Successful Deployment

```
Deploying to Sepolia...

✅ CorporateGovernance deployed!
Contract address: 0x13116d08546b78F5fDB7fA4544f778885B19A441
Transaction hash: 0xabc123def456...
Deployment cost: ~0.005 ETH

Waiting for block confirmations...
✅ 6 confirmations received

Verifying on Etherscan...
✅ Contract verified!
View: https://sepolia.etherscan.io/address/0x13116d08546b78F5fDB7fA4544f778885B19A441
```

### Gas Report

```
REPORT_GAS=true npm run test

·──────────────────────────────────┬─────────────┐
│  CorporateGovernance Methods     │ Gas Usage   │
├──────────────────────────────────┼─────────────┤
│ addBoardMember                   │ 123,456     │
│ createResolution                 │ 187,890     │
│ castVote (with FHE)              │ 387,654     │
│ closeResolution                  │ 145,230     │
└──────────────────────────────────┴─────────────┘
```

---

## 🐛 Troubleshooting Scripts

### Issue: "Cannot find contract"

**Solution:**
```bash
# Verify compilation
npx hardhat compile

# Check artifacts folder
ls artifacts/contracts/
```

### Issue: "Invalid network"

**Solution:**
```bash
# Check hardhat.config.ts for network name
# Ensure network is defined:
networks: {
    sepolia: { ... },
    localhost: { ... }
}
```

### Issue: "Insufficient funds"

**Solution:**
```bash
# Get testnet ETH from faucet
# https://sepoliafaucet.com

# Verify balance
npx hardhat accounts --network sepolia
```

---

## 🚀 Advanced Usage

### Custom Deployment Script

Create `scripts/custom-deploy.js`:

```javascript
const hre = require("hardhat");

async function main() {
    const [deployer] = await hre.ethers.getSigners();
    console.log(`Deploying with account: ${deployer.address}`);

    // Deploy
    const CG = await hre.ethers.getContractFactory("CorporateGovernance");
    const contract = await CG.deploy();
    await contract.waitForDeployment();

    const addr = await contract.getAddress();
    console.log(`✅ Deployed to: ${addr}`);

    // Initialize
    await contract.addBoardMember(deployer.address, "CEO", "Chief", 10);
    console.log(`✅ Board initialized`);
}

main().catch(console.error);
```

**Run:**
```bash
npx hardhat run scripts/custom-deploy.js --network sepolia
```

### Deployment with Setup

```javascript
// scripts/deploy-and-setup.js
async function main() {
    // Deploy
    const contract = await deploy();

    // Setup board
    await contract.addBoardMember("0xAlice", "Alice", "CEO", 5);
    await contract.addBoardMember("0xBob", "Bob", "CFO", 3);

    // Create initial resolution
    await contract.createResolution("Setup Complete", "Initialization", 3);

    console.log("✅ Deployment and setup complete!");
}
```

---

## 📝 Script Development Guide

### Creating New Scripts

1. **Create file in `scripts/`**
   ```bash
   touch scripts/my-script.js
   ```

2. **Follow pattern:**
   ```javascript
   async function main() {
       // Get signer
       const [deployer] = await ethers.getSigners();

       // Get contract
       const contract = await ethers.getContractAt(
           "CorporateGovernance",
           "0x..."
       );

       // Execute operations
       const tx = await contract.someFunction();
       await tx.wait();

       console.log("✅ Complete");
   }

   main().catch(error => {
       console.error(error);
       process.exit(1);
   });
   ```

3. **Run:**
   ```bash
   npx hardhat run scripts/my-script.js --network sepolia
   ```

---

## 🔐 Security Best Practices

### Safe Script Patterns

```javascript
// ✅ GOOD: Check before operations
const balance = await contract.getTotalVotingPower();
if (balance === 0) {
    console.error("No voting power!");
    return;
}

// ✅ GOOD: Wait for confirmations
const tx = await contract.someFunction();
await tx.wait(6);  // 6 confirmations

// ✅ GOOD: Error handling
try {
    await contract.operation();
} catch (error) {
    console.error("Operation failed:", error.message);
}
```

### Unsafe Patterns

```javascript
// ❌ BAD: No checks
await contract.dangerousOperation();

// ❌ BAD: No await
const tx = contract.someFunction();  // Missing await!

// ❌ BAD: No error handling
await contract.operation();  // May fail silently
```

---

## 📚 Related Documentation

- **DEPLOYMENT.md** - Full deployment guide
- **hardhat.config.ts** - Network configuration
- **package.json** - npm scripts definition
- **TROUBLESHOOTING.md** - Common issues

---

## 🎯 Common Tasks

### Save Contract Address

```bash
# Add to .env
echo "CORPORATE_GOVERNANCE_ADDRESS=0x..." >> .env
```

### Initialize Board Members

```bash
# Create scripts/init-board.js
async function main() {
    const contract = await ethers.getContractAt("CorporateGovernance", ADDRESS);
    await contract.addBoardMember(...);
}
```

### Monitor Contract

```bash
# Create scripts/monitor.js
async function main() {
    const contract = await ethers.getContractAt("CorporateGovernance", ADDRESS);
    const power = await contract.getTotalVotingPower();
    console.log(`Total voting power: ${power}`);
}
```

---

## ✅ Deployment Verification

After deployment, verify:

```bash
# Check contract exists
npx hardhat run scripts/verify-deployment.js --network sepolia

# Test basic functions
npx hardhat run scripts/test-functions.js --network sepolia

# Monitor events
npx hardhat run scripts/listen-events.js --network sepolia
```

---

**Last Updated**: December 2025
**Status**: All scripts functional and tested
