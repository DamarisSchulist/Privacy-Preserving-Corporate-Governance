# 🚀 Quick Start Guide

Get your Hello FHEVM tutorial running in 5 minutes!

## ⚡ One-Command Setup

```bash
# Clone and setup everything
git clone https://github.com/DamarisSchulist/CorporateGovernance.git
cd CorporateGovernance
npm run setup
```

## 🔧 Environment Configuration

1. **Copy environment file:**
   ```bash
   cp .env.example .env
   ```

2. **Get your MetaMask private key:**
   - Open MetaMask → Click account menu → Account Details → Export Private Key
   - ⚠️ **NEVER share this key!**

3. **Get Infura Project ID:**
   - Visit [infura.io](https://infura.io/) → Create free account → New Project → Copy Project ID

4. **Update `.env` file:**
   ```bash
   PRIVATE_KEY=your_private_key_without_0x
   INFURA_PROJECT_ID=your_infura_project_id
   ```

## 💰 Get Testnet ETH

Get free Sepolia ETH from any faucet:
- [sepoliafaucet.com](https://sepoliafaucet.com/)
- [faucets.chain.link](https://faucets.chain.link/)
- [infura.io/faucet](https://www.infura.io/faucet/sepolia)

## 🚀 Deploy & Run

```bash
# Compile contracts
npm run compile

# Deploy to Sepolia testnet
npm run deploy:sepolia

# Start the frontend
npm run dev
```

## 🌐 Access Your dApp

1. Browser opens automatically at `http://localhost:3000`
2. Connect MetaMask to Sepolia testnet
3. Click "Connect Wallet"
4. Start voting with FHE encryption! 🔐

## 📚 Next Steps

- Read the full tutorial: `HELLO_FHEVM_TUTORIAL.md`
- Explore the smart contracts in `/contracts`
- Modify the frontend in `index.html`
- Deploy to mainnet when ready!

## 🆘 Need Help?

- **Tutorial Issues**: Check `HELLO_FHEVM_TUTORIAL.md` Chapter 8 (Troubleshooting)
- **Contract Errors**: Review the deployment logs
- **MetaMask Issues**: Switch to Sepolia testnet
- **Gas Errors**: Ensure you have enough Sepolia ETH

## 🎯 What You're Building

A complete corporate governance system with:
- **🔐 Private Voting**: Votes encrypted with FHE
- **🏛️ Board Management**: Add/remove members
- **📊 Resolution Tracking**: Create and manage proposals
- **⚡ Real-time Updates**: Instant blockchain synchronization

**Live Demo**: [corporate-governance.vercel.app](https://corporate-governance.vercel.app/)

---

*🎉 You're now ready to build the future of private blockchain applications!*