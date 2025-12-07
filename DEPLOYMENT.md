# Deployment Guide - Corporate Governance FHEVM Example

## 📋 Prerequisites

```bash
Node.js >= 18.0.0
npm >= 8.0.0
Git
Hardhat knowledge
```

## 🛠️ Setup

### 1. Clone Repository

```bash
git clone https://github.com/DamarisSchulist/CorporateGovernance.git
cd CorporateGovernance
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Configure Environment

```bash
cp .env.example .env
```

Edit `.env`:
```bash
PRIVATE_KEY=your_private_key_without_0x
INFURA_API_KEY=your_infura_project_id
ETHERSCAN_API_KEY=your_etherscan_api_key (optional)
```

⚠️ **SECURITY WARNING**: Never commit `.env` with real keys!

## 🏗️ Local Development

### Start Local Node

```bash
npx hardhat node
```

This starts a local Hardhat node with FHEVM mock environment.

### Deploy to Localhost

In another terminal:

```bash
npx hardhat run scripts/deploy-corporate-governance.js --network localhost
```

Output:
```
✅ CorporateGovernance deployed to: 0x...
```

### Run Tests Locally

```bash
npm run test

# With gas reporting
REPORT_GAS=true npm run test

# Coverage
npm run coverage
```

## 🌐 Testnet Deployment (Sepolia)

### 1. Get Sepolia ETH

Visit faucet:
- https://sepoliafaucet.com
- https://www.infura.io/faucet/sepolia
- https://faucets.chain.link

### 2. Deploy to Sepolia

```bash
npx hardhat run scripts/deploy-corporate-governance.js --network sepolia
```

Expected output:
```
Deploying CorporateGovernance to Sepolia...
✅ Contract deployed to: 0x13116d08546b78F5fDB7fA4544f778885B19A441
Transaction hash: 0x...
```

### 3. Verify on Etherscan

```bash
npx hardhat verify --network sepolia 0x13116d08546b78F5fDB7fA4544f778885B19A441
```

Verify successful:
```
✅ Contract verified on Etherscan
View: https://sepolia.etherscan.io/address/0x13116d08546b78F5fDB7fA4544f778885B19A441
```

## 📝 Deployment Script Breakdown

File: `scripts/deploy-corporate-governance.js`

```javascript
async function main() {
    // Get deployer account
    const [deployer] = await ethers.getSigners();
    console.log(`Deploying with account: ${deployer.address}`);

    // Deploy contract
    const CorporateGovernance = await ethers.getContractFactory("CorporateGovernance");
    const contract = await CorporateGovernance.deploy();

    // Wait for deployment
    await contract.waitForDeployment();
    const address = await contract.getAddress();

    console.log(`✅ CorporateGovernance deployed to: ${address}`);

    // Optional: Verify on Etherscan
    if (network.name === "sepolia") {
        console.log("Waiting for block confirmations...");
        await contract.deploymentTransaction().wait(6);

        console.log("Verifying contract...");
        await hre.run("verify:verify", {
            address: address,
            constructorArguments: [],
        });
    }
}

main()
    .then(() => process.exit(0))
    .catch((error) => {
        console.error(error);
        process.exit(1);
    });
```

## ⛽ Gas Estimation

### Deployment Gas

```
Contract deployment: ~250,000 gas
Total cost (at 20 gwei): ~0.005 ETH
```

### Operation Gas Costs

| Operation | Gas | Cost (20 gwei) |
|-----------|-----|----------------|
| Add board member | ~120,000 | 0.0024 ETH |
| Create resolution | ~200,000 | 0.004 ETH |
| Cast vote | ~400,000 | 0.008 ETH |
| Close resolution | ~150,000 | 0.003 ETH |

## 🔒 Post-Deployment Steps

### 1. Save Contract Address

```bash
# Add to .env
CORPORATE_GOVERNANCE_ADDRESS=0x13116d08546b78F5fDB7fA4544f778885B19A441
```

### 2. Initialize Board Members

```javascript
const contract = await ethers.getContractAt("CorporateGovernance", deployedAddress);

// Add board members
await contract.addBoardMember(
    "0xAliceAddress",
    "Alice",
    "Chief Executive Officer",
    5  // voting power
);

await contract.addBoardMember(
    "0xBobAddress",
    "Bob",
    "Chief Financial Officer",
    3  // voting power
);
```

### 3. Test Basic Operations

```javascript
// Create resolution
await contract.createResolution(
    "Test Resolution",
    "Testing the deployment",
    5  // quorum
);

// Verify creation
const count = await contract.getResolutionCount();
console.log(`Resolutions: ${count}`);
```

## 🚨 Troubleshooting Deployment

### Error: "Insufficient funds"

```bash
# Ensure you have Sepolia ETH
# Get from faucet: https://sepoliafaucet.com
# Check balance:
npx hardhat accounts --network sepolia
```

### Error: "Invalid private key"

```bash
# Verify .env
cat .env
# Private key should be without 0x prefix
```

### Error: "Network timeout"

```bash
# Check RPC endpoint in hardhat.config.ts
# Ensure INFURA_API_KEY is valid
# Try different RPC provider if needed
```

### Contract doesn't verify

```bash
# Ensure 6+ block confirmations before verifying
# Check contract is on Etherscan
# Verify with exact same source code
```

## 📊 Deployment Checklist

- ✅ Environment configured (.env created)
- ✅ Dependencies installed
- ✅ Tests passing locally
- ✅ Contract compiles without warnings
- ✅ Sepolia ETH obtained
- ✅ Deployment script runs successfully
- ✅ Contract appears on Etherscan
- ✅ Contract verified on Etherscan
- ✅ Address saved safely
- ✅ Initial board members added
- ✅ Basic operations tested

## 📚 Next Steps

1. **Deploy to Zama Devnet** (if available)
   ```bash
   npx hardhat run scripts/deploy-corporate-governance.js --network zama
   ```

2. **Connect Frontend**
   - Update contract address in frontend
   - Update ABI in frontend
   - Test voting operations

3. **Monitor Contract**
   - Watch for governance events
   - Monitor voting activity
   - Track resolution outcomes

4. **Upgrade if Needed**
   - Deploy new version
   - Update frontend to new address
   - Migrate any state if necessary

## 🔐 Security Checklist

- ✅ Never commit private keys
- ✅ Use separate accounts for testnet/mainnet
- ✅ Verify contracts before mainnet
- ✅ Test thoroughly on testnet first
- ✅ Set up monitoring/alerts
- ✅ Have upgrade plan ready
- ✅ Document all addresses
- ✅ Back up configuration

## 📞 Support

- **Deployment Issues**: Check Hardhat docs
- **FHEVM Questions**: See FHEVM docs
- **Contract Issues**: Review smart contract code
- **RPC Problems**: Check Infura status

---

**Last Updated**: December 2025
**Network**: Ethereum Sepolia Testnet
**Status**: Production Ready
