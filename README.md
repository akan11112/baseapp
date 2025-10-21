#!/usr/bin/env bash
set -e

ROOT="base-sepolia-message-dapp"
mkdir -p "$ROOT"
cd "$ROOT"

# package.json
cat > package.json <<'EOF'
{
  "name": "base-sepolia-message-dapp",
  "version": "1.0.0",
  "description": "Simple Message dApp for Base Sepolia",
  "scripts": {
    "compile": "npx hardhat compile",
    "deploy:sepolia": "npx hardhat run scripts/deploy.js --network baseSepolia",
    "create-wallet": "node scripts/createWallet.js",
    "start-frontend": "npx http-server -c-1 -p 8080"
  },
  "devDependencies": {
    "@nomicfoundation/hardhat-toolbox": "^3.0.0",
    "dotenv": "^16.0.0",
    "hardhat": "^2.16.0"
  },
  "dependencies": {
    "http-server": "^14.1.1"
  }
}
EOF

# .gitignore
cat > .gitignore <<'EOF'
node_modules
.env
dist
.cache
.DS_Store
EOF

# .env.example
cat > .env.example <<'EOF'
# COPY THIS to .env and fill values (do NOT commit .env)
PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE
BASE_SEPOLIA_RPC=https://sepolia.base.org
# (Optional) ETHERSCAN_API_KEY=your_key_here
EOF

# hardhat.config.js
cat > hardhat.config.js <<'EOF'
require("dotenv").config();
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.20",
  networks: {
    baseSepolia: {
      url: process.env.BASE_SEPOLIA_RPC || "https://sepolia.base.org",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : []
    }
  }
};
EOF

# contracts
mkdir -p contracts
cat > contracts/MessageStorage.sol <<'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MessageStorage {
    string private message;

    event MessageSet(address indexed setter, string message);

    function setMessage(string calldata _msg) external {
        message = _msg;
        emit MessageSet(msg.sender, _msg);
    }

    function getMessage() external view returns (string memory) {
        return message;
    }
}
EOF

# scripts
mkdir -p scripts
cat > scripts/deploy.js <<'EOF'
async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying from:", deployer.address);

  const Factory = await ethers.getContractFactory("MessageStorage");
  const contract = await Factory.deploy();
  await contract.deployed();

  console.log("Deployed MessageStorage at:", contract.address);
}

main().catch((err) => {
  console.error(err);
  process.exitCode = 1;
});
EOF

cat > scripts/createWallet.js <<'EOF'
const { ethers } = require("ethers");

function main(){
  const wallet = ethers.Wallet.createRandom();
  console.log("Address:", wallet.address);
  console.log("Private Key:", wallet.privateKey);
  console.log("Mnemonic:", wallet.mnemonic.phrase);
  console.log("\n*** Bu private key'i yalnızca test için kullan; asla paylaşma! ***");
}

main();
EOF

# frontend
cat > index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Base Sepolia Message dApp</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 720px; margin: 40px auto; }
    input[type="text"] { width: 100%; padding: 8px; margin: 8px 0; }
    button { padding: 8px 12px; margin-right: 8px; }
    p { background:#f7f7f7; padding:12px; border-radius:6px; }
  </style>
</head>
<body>
  <h1>Message dApp (Base Sepolia)</h1>
  <p>MetaMask ile Base Sepolia ağına bağlanın.</p>

  <div>
    <input id="messageInput" type="text" placeholder="Mesajınızı yazın..." />
    <div style="margin-top:8px;">
      <button id="btnSet">Mesajı Kaydet</button>
      <button id="btnGet">Mesajı Görüntüle</button>
    </div>
  </div>

  <h3>Mevcut Mesaj</h3>
  <p id="currentMessage">--</p>

  <script src="https://cdn.jsdelivr.net/npm/ethers@5.7.2/dist/ethers.umd.min.js"></script>
  <script>
    // DEPLOY sonrası buraya kontrat adresini koy
    const CONTRACT_ADDRESS = "0xPUT_YOUR_CONTRACT_ADDRESS_HERE";
    const ABI = [
      "function setMessage(string _msg) external",
      "function getMessage() external view returns (string)"
    ];

    async function ensureMetaMask() {
      if (!window.ethereum) {
        alert("MetaMask yok. Tarayıcıda MetaMask yüklü olmalı.");
        throw new Error("MetaMask not found");
      }
    }

    async function setMessage() {
      await ensureMetaMask();
      const msg = document.getElementById("messageInput").value;
      if (!msg) return alert("Mesaj boş olamaz");

      const provider = new ethers.providers.Web3Provider(window.ethereum);
      await provider.send("eth_requestAccounts", []);
      const signer = provider.getSigner();
      const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);

      try {
        const tx = await contract.setMessage(msg);
        document.getElementById("btnSet").disabled = true;
        await tx.wait();
        alert("Mesaj blockchain'e kaydedildi");
      } catch (e) {
        console.error(e);
        alert("İşlem hatası: " + (e.message || e));
      } finally {
        document.getElementById("btnSet").disabled = false;
      }
    }

    async function getMessage() {
      await ensureMetaMask();
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, provider);
      try {
        const msg = await contract.getMessage();
        document.getElementById("currentMessage").innerText = msg || "(boş)";
      } catch (e) {
        console.error(e);
        alert("Oku hatası: " + (e.message || e));
      }
    }

    document.getElementById("btnSet").addEventListener("click", setMessage);
    document.getElementById("btnGet").addEventListener("click", getMessage);
  </script>
</body>
</html>
EOF

# README
cat > README.md <<'EOF'
# Base Sepolia Message dApp

Basit bir dApp: kullanıcılar mesaj yazıp blockchain'e kaydedebilir ve mevcut mesajı okuyabilir.

## Dosyalar
- `contracts/MessageStorage.sol` — Solidity kontratı
- `scripts/deploy.js` — Hardhat deploy script
- `scripts/createWallet.js` — Test throwaway cüzdan oluşturur
- `index.html` — Frontend (MetaMask ile etkileşim)
- `.env.example` — .env örneği (PRIVATE_KEY, BASE_SEPOLIA_RPC)
- `.github/workflows/deploy.yml` — (opsiyonel) GitHub Actions deploy workflow

## Kurulum
1. `npm install`
2. `.env` oluştur: `.env.example` kopyala ve `PRIVATE_KEY` ile doldur
3. `npm run compile`
4. `npm run deploy:sepolia`
5. Deploy sonrası `index.html` içindeki CONTRACT_ADDRESS'e kontrat adresini koy
6. `npm run start-frontend` -> `http://localhost:8080`

## Güvenlik
- `.env` dosyasını asla repoya koyma.
- Deploy için test cüzdan (throwaway) kullan.

EOF

# GitHub Actions workflow (optional)
mkdir -p .github/workflows
cat > .github/workflows/deploy.yml <<'EOF'
name: Deploy to Base Sepolia

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18
      - name: Install
        run: npm ci
      - name: Compile
        run: npx hardhat compile
      - name: Deploy
        env:
          PRIVATE_KEY: ${{ secrets.BASE_PRIVATE_KEY }}
          BASE_SEPOLIA_RPC: ${{ secrets.BASE_SEPOLIA_RPC }}
        run: npx hardhat run scripts/deploy.js --network baseSepolia
EOF

echo "Bootstrap complete in $(pwd)"
echo "Next steps:"
echo "  1) cd $ROOT"
echo "  2) npm install"
echo "  3) create .env from .env.example and fill PRIVATE_KEY (use test wallet)"
echo "  4) npm run compile"
echo "  5) npm run deploy:sepolia"
echo "  6) put deployed contract address into index.html"

