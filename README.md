# Irys-ai-txt-storage
## A fun and easy way to test out Irys storage. This guide will help you set up a script on a new server that continuously generates and uploads text files to the Irys testnet.
NOT INCENTIVISED AT ALL 


## 1️⃣ Minimal Requirements
.RAM: 2 GB is enough; 4 GB recommended.
.Storage: ~200 MB for Node.js, packages, and files.
.OS: Ubuntu 20.04+ is recommended.
.Network: A stable internet connection. (uploads require network to Irys testnet)

TESTNET TOKENS
ETH Sepolia faucet (get free tokens from [Alchemy](https://www.alchemy.com/faucets/ethereum-sepolia) Faucet)
IRYS faucet ([here]([https://irys.xyz/faucet])
PLEASE DON'T USE YOUR MAIN WALLET. CREATE A NEW, DEDICATED BURNER WALLET FOR THIS.

# STEP 1. Installations:
## install dependencies
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make protobuf-compiler cmake gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev screen ufw -y
```
## Install nodejs & npm :
This script runs on Node.js. The following command will install version 20.x.
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt install -y nodejs
``` 
For macOS, you can run

      brew install node

## Check version :
```bash
node -v
npm -v
```
## Install PM2 Process Manager
PM2 will run our script in the background and ensure it restarts automatically if it crashes or the server reboots.
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
This file will securely store your private key.

```bash
nano .env
```
Inside the editor, add the following line, replacing the `YOUR_ETHEREUM_PRIVATE_KEY` with your burner wallet's Ethereum
PLEASE IN THE NAME OF WHAT YOU SERVE USE A BURNER 
```bash
PRIVATE_KEY=YOUR_ETHEREUM_PRIVATE_KEY
```
Press Ctrl+X, then Y, then Enter to save and exit.

## Create the Uploader Script

```bash
cd ~/irys-basic-watcher

cat > looper.js << 'EOL'
const Irys = require("@irys/sdk");
const fs = require("fs/promises");
const path = require("path");
require("dotenv").config();

// --- Configuration ---
const BATCH_SIZE = 100; // How many files to upload per sequence
const DELAY_BETWEEN_BATCHES_MS = 60 * 1000; // 60 seconds
const TEMP_DIR = "./temp_files_to_upload"; // Temp folder for generated files
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
    try {
        await fs.access(dirPath);
    } catch (error) {
        await fs.mkdir(dirPath, { recursive: true });
    }
};

const main = async () => {
    if (!PRIVATE_KEY) {
        console.error("ERROR: PRIVATE_KEY is not defined in your .env file.");
        return;
    }
    await ensureDir(TEMP_DIR);
    console.log("Starting Irys Infinite Uploader on TESTNET...");
    
    const irys = await getIrys();

    while (true) {
        sequenceId++;
        console.log(`\n--- Starting Sequence ${sequenceId} ---`);

        for (let i = 1; i <= BATCH_SIZE; i++) {
            const fileName = `sequence_${sequenceId}_file_${i}.txt`;
            const filePath = path.join(TEMP_DIR, fileName);
            const fileContent = `This is test file #${i} from sequence #${sequenceId}. Timestamp: ${Date.now()}`;

            try {
                await fs.writeFile(filePath, fileContent);
                
                const tags = [{ name: "Content-Type", value: "text/plain" }];
                const response = await irys.uploadFile(filePath, { tags });
                console.log(`[Seq ${sequenceId}] ${fileName} -> https://gateway.irys.xyz/${response.id}`);

            } catch (e) {
                if (e.message && e.message.includes("402")) {
                     console.log(`[Seq ${sequenceId}] ${fileName} Failed: 402 Not enough balance`);
                } else {
                     console.log(`[Seq ${sequenceId}] ${fileName} Failed: ${e.message}`);
                }
            } finally {
                try {
                    await fs.unlink(filePath);
                } catch (unlinkError) {}
            }
        }
        
        console.log(`--- Sequence ${sequenceId} complete. Waiting ${DELAY_BETWEEN_BATCHES_MS / 1000} seconds... ---`);
        await new Promise(resolve => setTimeout(resolve, DELAY_BETWEEN_BATCHES_MS));
    }
};

main();
EOL
```

## Start and Save the Script
Start the script with PM2 and then save it to run on reboot.
This is a multi-step process to ensure your script always runs.

```bash
pm2 start looper.js --name "irys-watcher"
pm2 startup
```
Simply type this command and press Enter:
```bash
pm2 save
```
## CHECK LOGS 
```bash
pm2 logs irys-watcher
```

DONE 
 by [Jrkenny](https://x.com/Jrken_ny)
