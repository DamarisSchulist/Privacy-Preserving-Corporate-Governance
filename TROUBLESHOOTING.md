# Troubleshooting Guide

## 🔴 Common Issues and Solutions

### Installation Issues

#### Issue: `npm install` fails

**Error:**
```
npm ERR! ERR! code ERESOLVE
npm ERR! ERESOLVE unable to resolve dependency tree
```

**Solution:**
```bash
# Use legacy peer dependencies
npm install --legacy-peer-deps

# Or use npm 7+ with force
npm install --force
```

---

#### Issue: Missing `@fhevm/solidity`

**Error:**
```
Cannot find module '@fhevm/solidity'
```

**Solution:**
```bash
# Reinstall all dependencies
rm -rf node_modules package-lock.json
npm install
```

---

### Compilation Issues

#### Issue: Solidity version mismatch

**Error:**
```
Compiler version 0.8.24 not found
```

**Solution:**
```bash
# Update hardhat
npm install --save-dev hardhat@latest

# Or use compatible version in hardhat.config.ts
solidity: {
    version: "0.8.24"  // Change to available version
}
```

---

#### Issue: FHEVM library not found

**Error:**
```
"fhevm/lib/TFHE.sol" not found
```

**Solution:**
```bash
# Verify FHEVM imports
npm list @fhevm/solidity

# If missing, install
npm install --save-dev @fhevm/solidity
```

---

### Testing Issues

#### Issue: Tests fail with "FHEVM mock not available"

**Error:**
```
Cannot read property 'isMock' of undefined
```

**Solution:**
```bash
# Run tests with Hardhat (provides FHEVM mock)
npx hardhat test

# NOT with plain mocha
# ❌ mocha test/*.ts  # Wrong!
# ✅ npx hardhat test  # Correct!
```

---

#### Issue: Gas limit too low for tests

**Error:**
```
Transaction reverted: out of gas
```

**Solution:**
```typescript
// Increase gas limit in test
const tx = await contract.castVote(
    0,
    encrypted.handle,
    encrypted.proof,
    { gasLimit: 500000 }  // Increase this!
);
```

---

#### Issue: Invalid proof in tests

**Error:**
```
Invalid input proof
```

**Solution:**
```typescript
// Ensure proof matches user
const encrypted = await fhevm
    .createEncryptedInput(contractAddr, signer.address)  // Correct user
    .add1(1)
    .encrypt();

// Sign as same user
await contract.connect(signer).castVote(
    0,
    encrypted.handles[0],
    encrypted.inputProof
);
```

---

### Deployment Issues

#### Issue: "Insufficient funds"

**Error:**
```
insufficient funds for gas * price + value
```

**Solution:**
```bash
# Get testnet ETH from faucet
# Sepolia: https://sepoliafaucet.com

# Or use different account
# Update PRIVATE_KEY in .env

# Check balance
npx hardhat accounts --network sepolia
```

---

#### Issue: "Invalid private key"

**Error:**
```
invalid private key format
```

**Solution:**
```bash
# Private key should NOT have 0x prefix in .env
# ❌ PRIVATE_KEY=0x1234...  # Wrong!
# ✅ PRIVATE_KEY=1234...   # Correct!
```

---

#### Issue: Network timeout

**Error:**
```
Timeout waiting for transaction
```

**Solution:**
```bash
# Check RPC endpoint
# Verify INFURA_API_KEY in .env
# Try different RPC provider

# In hardhat.config.ts:
sepolia: {
    url: `https://sepolia.infura.io/v3/${process.env.INFURA_API_KEY}`,
    // Try alternative:
    // url: "https://public-rpc.sepolia.org"
}
```

---

#### Issue: Contract verification fails

**Error:**
```
Etherscan verification failed
```

**Solution:**
```bash
# Wait for 6 confirmations
# Verify with exact source code
# Check constructor arguments

npx hardhat verify --network sepolia \
    0x13116d08546b78F5fDB7fA4544f778885B19A441
```

---

### Runtime Issues

#### Issue: "Transaction reverted: Unknown reason"

**Solution:**
```javascript
// Enable detailed revert reasons
try {
    const tx = await contract.castVote(...);
    await tx.wait();
} catch (error) {
    // Check error reason
    if (error.reason) {
        console.error("Revert:", error.reason);
    } else if (error.message) {
        console.error("Error:", error.message);
    }
}
```

---

#### Issue: Encrypted value not working

**Error:**
```
Missing allowThis permission
```

**Solution:**
```solidity
// Always grant BOTH permissions
FHE.allowThis(encryptedValue);      // Contract permission
FHE.allow(encryptedValue, msg.sender); // User permission
```

---

#### Issue: Can't decrypt in view function

**Error:**
```
Cannot call non-static method
```

**Solution:**
```solidity
// ❌ WRONG: View function
function getResults() external view returns (uint256) {
    return TFHE.decrypt(yesVotes);  // Error!
}

// ✅ RIGHT: Use gateway for decryption
function closeResolution() external {
    Gateway.requestDecryption(cts, this.resolveResolution.selector, ...);
}

function resolveResolution(uint256, uint256[] memory results) public onlyGateway {
    // Process decrypted results here
}
```

---

### Frontend Issues

#### Issue: Contract address not found

**Error:**
```
Cannot read property 'address' of undefined
```

**Solution:**
```javascript
// Ensure contract deployed first
const address = "0x13116d08546b78F5fDB7fA4544f778885B19A441";
const contract = new ethers.Contract(address, ABI, signer);

// Verify address exists
console.log(address);  // Should print valid address
```

---

#### Issue: MetaMask network mismatch

**Error:**
```
chainId mismatch
```

**Solution:**
```javascript
// Ensure MetaMask on Sepolia
// ChainId: 11155111

// Or add network to MetaMask:
window.ethereum.request({
    method: "wallet_addEthereumChain",
    params: [{
        chainId: "0xaa36a7",  // 11155111 in hex
        chainName: "Sepolia",
        rpcUrls: ["https://sepolia.infura.io/v3/YOUR_KEY"],
        nativeCurrency: { name: "ETH", symbol: "ETH", decimals: 18 }
    }]
});
```

---

## 📊 Debug Mode

### Verbose Output

```bash
# Run tests with detailed logging
DEBUG=* npx hardhat test

# Or in code
console.log("Debug:", await contract.getTotalVotingPower());
```

---

### Check Contract State

```javascript
// Verify contract state after operations
const totalPower = await contract.getTotalVotingPower();
const memberCount = await contract.getResolutionCount();
const member = await contract.getBoardMember(address);

console.log({
    totalPower,
    memberCount,
    member
});
```

---

## 🔍 Debugging Encrypted Operations

### Verify Proof Generation

```typescript
// Ensure proof is valid before submission
const encrypted = await fhevm
    .createEncryptedInput(contractAddr, userAddr)
    .add1(1)
    .encrypt();

console.log({
    handle: encrypted.handles[0],
    proof: encrypted.inputProof,
    address: userAddr
});

// Verify proof is not empty
if (!encrypted.inputProof || encrypted.inputProof === "0x") {
    console.error("Invalid proof!");
}
```

---

### Check Permissions

```solidity
// Verify permissions are set
require(FHE.isCiphertext(someValue), "Not encrypted");

// Always set both permissions
FHE.allowThis(value);
FHE.allow(value, msg.sender);
```

---

## 💡 Best Practices for Troubleshooting

1. **Always check logs first**
   ```bash
   npm run test 2>&1 | head -50
   ```

2. **Isolate the issue**
   - Test in isolation
   - Check one component at a time
   - Use simplified examples

3. **Verify requirements**
   ```bash
   node --version   # >= 18
   npm --version    # >= 8
   npx hardhat --version
   ```

4. **Clear cache if stuck**
   ```bash
   npx hardhat clean
   rm -rf node_modules
   npm install
   ```

5. **Check official docs**
   - FHEVM: https://docs.zama.ai/fhevm
   - Hardhat: https://hardhat.org
   - Ethers.js: https://docs.ethers.org

---

## 📞 Getting Help

### Check These First

1. **README.md** - General overview
2. **TEST_GUIDE.md** - Testing help
3. **DEPLOYMENT.md** - Deployment help
4. **API_REFERENCE.md** - Function reference

### Ask for Help

- **GitHub Issues**: Report bugs
- **Zama Discord**: Community support
- **Stack Overflow**: General Web3 questions

---

## 🎯 Checklist Before Reporting Issue

- ✅ Ran `npm install` successfully
- ✅ Using Node.js >= 18
- ✅ Checked `.env` configuration
- ✅ Read relevant documentation
- ✅ Tried the suggested solutions above
- ✅ Have minimal reproducible example
- ✅ Clear error message saved

---

**Last Updated**: December 2025
**Status**: Common issues documented
