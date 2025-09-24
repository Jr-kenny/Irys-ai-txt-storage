# Irys-ai-txt-storage
## Just an easy fun way of testing out irys storage 

## 1️⃣ Minimal Requirements
RAM	2 GB is enough; 4 GB recommended
Storage	~200 MB for Node.js, packages, and files
OS	Ubuntu 20.04+, any Linux distro, macOS, or Windows with Node.js
Network	Stable internet connection (uploads require network to Irys testnet)
TESTNET TOKENS
ETH Sepolia faucet (get free tokens from [Alchemy](https://www.alchemy.com/faucets/ethereum-sepolia) Faucet)
IRYS faucet ([here]([https://irys.xyz/faucet])
PLEASE DON'T USE YOUR MAIN WALLET

# STEP 1. Installations:
## install dependencies
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make protobuf-compiler cmake gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev screen ufw -y
```
## Install nodejs & npm :
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt install -y nodejs
```
for mac

      brew install node

## Check version :
```bash
node -v
npm -v
```
## Install PM2 Process Manager
PM2 is a powerful tool that will run our script in the background and ensure it restarts automatically if it crashes or the server reboots.
```bash
sudo npm install -g pm2
```

# STEP 2. CREATE FOLDER
```bash
mkdir ~/irys-txt-cli
cd ~/irys-txt-cli
```
## Initialize Node project :
```bash
npm init -y
```
## Install Irys SDK packages:
```bash
npm install @irys/upload @irys/upload-ethereum
npm install @irys/sdk dotenv
```
## Create the Directory Structure
This script needs folders to watch for new files and to store processed files.
```bash
mkdir prompts_to_upload
mkdir prompts_processed
mkdir prompts_failed
```
## Create the .env Configuration File 
```bash
nano .env
```
Inside the editor, add the following line, replacing the placeholder with your Ethereum private key.
PLEASE IN THE NAME OF WHAT YOU SERVE USE A BURNER 
```bash
PRIVATE_KEY=YOUR_ETHEREUM_PRIVATE_KEY
```
Press Ctrl+X, then Y, then Enter to save and exit.

## Create the Uploader Script

```bash
cat > looper.js << 'EOL'
const Irys = require("@irys/sdk");
const fs = require("fs/promises");
const path = require("path");
require("dotenv").config();

const POLLING_INTERVAL_MS = 10 * 1000; // Check for new files every 10 seconds
const PROMPTS_DIR = "./prompts_to_upload";
const PROCESSED_DIR = "./prompts_processed";
const FAILED_DIR = "./prompts_failed";
const PRIVATE_KEY = process.env.PRIVATE_KEY;

let sequenceId = 0;

const getIrys = async () => {
    // --- TESTNET CONFIGURATION ---
    const network = "devnet";
    const providerUrl = "https://sepolia.base.org";
    const token = "ethereum";
    return new Irys({ network, token, key: PRIVATE_KEY, config: { providerUrl } });
};

const ensureDir = async (dirPath) => {
    try { await fs.access(dirPath); } catch (error) { await fs.mkdir(dirPath, { recursive: true }); }
};

const main = async () => {
    if (!PRIVATE_KEY) {
        console.error("ERROR: PRIVATE_KEY is not defined in your .env file.");
        return;
    }
    await ensureDir(PROCESSED_DIR);
    await ensureDir(FAILED_DIR);
    await ensureDir(PROMPTS_DIR);
    console.log("Starting Irys Directory Watcher on TESTNET...");

    while (true) {
        try {
            const files = await fs.readdir(PROMPTS_DIR);
            if (files.length > 0) {
                sequenceId++;
                console.log(`\n--- Sequence ${sequenceId} ---`);
                const irys = await getIrys();
                for (const file of files) {
                    const filePath = path.join(PROMPTS_DIR, file);
                    try {
                        const tags = [{ name: "Content-Type", value: "application/octet-stream" }];
                        const response = await irys.uploadFile(filePath, { tags });
                        console.log(`[Seq ${sequenceId}] ${file} -> https://gateway.irys.xyz/${response.id}`);
                        await fs.rename(filePath, path.join(PROCESSED_DIR, file));
                    } catch (e) {
                        console.log(`[Seq ${sequenceId}] ${file} Failed: ${e.message}`);
                        await fs.rename(filePath, path.join(FAILED_DIR, file));
                    }
                }
            }
        } catch (e) {
            console.error("A critical error occurred: ", e.message);
        }
        await new Promise(resolve => setTimeout(resolve, POLLING_INTERVAL_MS));
    }
};

main();
EOL
```

## Start and Save the Script
Start the script with PM2 and then save it to run on reboot.

```bash
pm2 start looper.js --name "irys-watcher"
pm2 startup
```
Copy the command that pm2 startup gives you and run it. Then, save the process list.
After you run the command above, your terminal will show you a new, long command as its output. It will look similar to this example:
`sudo env PATH=$PATH:/root/.nvm/versions/node/v20.11.1/bin /root/.nvm/versions/node/v20.11.1/lib/node_modules/pm2/bin/pm2 startup systemd -u root --hp /root`


