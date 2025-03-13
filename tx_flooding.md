# TX FLOODING 
This has 2 components:
* javascript to create wallets and submit transactions in parallel
* javascript to recover funds distributed to the other wallets
<br>

## Tx FLOODING - Submit
Configure:
* number of wallets
* range for batch size
* amount of WMTX to send to each subwallet
* how fast to scale gas price up
* max tx to submit

Will write file out containing wallet recovery details for stage2 - get any WMTx back
<br>

```javascript
// TRANSACTION FLOODING ON TESTNET - Parallel version
// Creates subwallets for parallel processing
import { ethers } from "ethers";
import crypto from "crypto";
import fs from 'fs';
import path from 'path';

// Configuration
const PRIVATE_KEY = 'INSERT_PRIVATE_KEY_HERE';
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const CONFIG_PATH = path.join(process.cwd(), 'config');
const TOTAL_TRANSACTIONS = 10000; // Stop when the total passes this
const WALLET_COUNT = 4; // Number of wallets to create eg 20
const FUNDING_AMOUNT_WMTX = 0.1; // Amount of WMTx to send to each subwallet
const ENABLE_GAS_ADJUSTMENT = true;  // Adapt gas amount according to current requirements
const GAS_PRICE_BUFFER_MULTIPLIER = 1.5; // The multiplier as a decimal
const BATCH_SIZE_MIN = 250;  // Random batch size greater than this and..
const BATCH_SIZE_MAX = 300; // Random batch size less than this

// Derived configuration
const BATCH_SIZE = Math.floor(Math.random() * (BATCH_SIZE_MAX - BATCH_SIZE_MIN + 1)) + BATCH_SIZE_MIN; // Random batch size
const FUNDING_AMOUNT = ethers.parseEther(FUNDING_AMOUNT_WMTX.toString()); // Amount to send to each subwallet

// Provider with optimized settings
const provider = new ethers.JsonRpcProvider(RPC_URL, undefined, {
    staticNetwork: true,
    batchMaxCount: 500,
    polling: false,
});


// Create wallet instances
async function createWallets() {
    const wallets = [];
    const walletInfo = [];
    
    // Main wallet
    const mainWallet = new ethers.Wallet(PRIVATE_KEY, provider);
    wallets.push(mainWallet);
    walletInfo.push({
        address: mainWallet.address,
        privateKey: mainWallet.privateKey,
        isMain: true,
        emptied: true
    });
    
    console.log(`Main wallet address: ${mainWallet.address}`);

    // Can main wallet fund the subwallets - is there enough WMTX?
    const mainWalletBalance = await provider.getBalance(mainWallet.address);
    const totalFundingNeeded = ethers.parseEther((FUNDING_AMOUNT_WMTX * (WALLET_COUNT - 1)).toString());

    if (mainWalletBalance < totalFundingNeeded) {
        console.error(`Insufficient funds: need ${ethers.formatEther(totalFundingNeeded)} WMTX but have ${ethers.formatEther(mainWalletBalance)} WMTX`);
        process.exit(1);
    }
    
    // Create additional wallets and fund them
    if (WALLET_COUNT > 1) {
        console.log(`Creating ${WALLET_COUNT - 1} additional wallets and funding them...`);
        
        for (let i = 1; i < WALLET_COUNT; i++) {
            const newWallet = ethers.Wallet.createRandom().connect(provider);
            wallets.push(newWallet);
            
            // Fund the new wallet
            console.log(`Funding wallet ${i}: ${newWallet.address}`);
            // Get current gas price from network, increase by multiplier
            const feeData = await provider.getFeeData();
            const boostedGasPrice = BigInt(Math.floor(Number(feeData.gasPrice) * GAS_PRICE_BUFFER_MULTIPLIER));
            const tx = await mainWallet.sendTransaction({
                to: newWallet.address,
                value: FUNDING_AMOUNT,
                gasPrice: boostedGasPrice // Use the boosted current gas price
            });
            await tx.wait();
            console.log(`Funded wallet ${i} with ${ethers.formatEther(FUNDING_AMOUNT)} WMTX. Tx: ${tx.hash}`);
            
            walletInfo.push({
                address: newWallet.address,
                privateKey: newWallet.privateKey,
                isMain: false,
                emptied: false
            });
        }
    }
    
    // Save wallet information for recovery of funds
    const timestamp = new Date();
    const formattedTimestamp = timestamp.getFullYear().toString() +
                               (timestamp.getMonth() + 1).toString().padStart(2, '0') +
                               timestamp.getDate().toString().padStart(2, '0') + '.' +
                               timestamp.getHours().toString().padStart(2, '0') +
                               timestamp.getMinutes().toString().padStart(2, '0') +
                               timestamp.getSeconds().toString().padStart(2, '0');
    
    const walletFilename = `generated-wallets.${formattedTimestamp}.json`;
    
    fs.writeFileSync(
        walletFilename,
        JSON.stringify(walletInfo, null, 2)
    );
    console.log(`Saved wallet information to ${walletFilename}`);

    return wallets;
}

// Pre-generate addresses to reduce overhead
function generateRandomAddresses(count) {
    return Array.from({ length: count }, () => {
        const randomBytes = crypto.randomBytes(20);
        return ethers.getAddress("0x" + randomBytes.toString("hex"));
    });
}

const preGeneratedAddresses = generateRandomAddresses(BATCH_SIZE);

// Track current batch gas price
let currentGasPrice = BigInt(1000000000); // Start at 1 Gwei

// Updates gas price by 20% when transactions fail
async function updateGasPrice() {
    if (!ENABLE_GAS_ADJUSTMENT) return;
    
    try {
        const feeData = await provider.getFeeData();
        currentGasPrice = BigInt(feeData.gasPrice) * BigInt(150) / BigInt(100);
    } catch (error) {
        currentGasPrice = currentGasPrice * BigInt(150) / BigInt(100);
    }
}

// Sends a batch of zero-value transactions to random addresses
async function sendBatch(wallet, startNonce, count) {
    const txPromises = [];
    
    for (let i = 0; i < count; i++) {
        const randomAddress = preGeneratedAddresses[i % BATCH_SIZE];
        
        const txPromise = wallet.sendTransaction({
            to: randomAddress,
            value: 0,
            gasLimit: 21000,
            nonce: startNonce + i,
            gasPrice: currentGasPrice
        }).then(tx => {
            console.log(`[${wallet.address.slice(0, 6)}...] Transaction sent: ${tx.hash} gas: ${currentGasPrice} wei`);
            return tx;
        }).catch(err => {
            if (err.code === 'NONCE_EXPIRED' || err.message?.includes('nonce too low')) {
                console.log(`[${wallet.address.slice(0, 6)}...] Nonce mismatch detected: ${startNonce + i}`);
                throw err;
            }
            if (err.code === 'REPLACEMENT_UNDERPRICED' || 
                (err.code === -32000 && err.message?.includes('replacement transaction underpriced'))) {
                console.log(`[${wallet.address.slice(0, 6)}...] Gas price too low for transaction ${startNonce + i}`);
            } else {
                console.error(`[${wallet.address.slice(0, 6)}...] Error in transaction ${startNonce + i}:`, err);
            }
            return null;
        });
        
        txPromises.push(txPromise);
    }
    
    return Promise.all(txPromises);
}

// Run transaction flooding for a single wallet
async function runWalletFlooding(wallet, transactionsPerWallet) {
    let currentNonce = await provider.getTransactionCount(wallet.address, "latest");
    console.log(`[${wallet.address.slice(0, 6)}...] Starting nonce: ${currentNonce}`);
    
    let processedTx = 0;
    
    while (processedTx < transactionsPerWallet) {
        const remainingTx = transactionsPerWallet - processedTx;
        const batchSize = Math.min(BATCH_SIZE, remainingTx);
        
        console.log(`[${wallet.address.slice(0, 6)}...] Processing batch of ${batchSize} transactions starting at nonce ${currentNonce}`);
        
        try {
            const results = await sendBatch(wallet, currentNonce, batchSize);
            const successfulTxs = results.filter(tx => tx !== null);
            processedTx += successfulTxs.length;
            currentNonce += batchSize;
            
            // If we had any failures in the batch, update gas price for next batch
            if (successfulTxs.length < batchSize) {
                await updateGasPrice();
                console.log(`[${wallet.address.slice(0, 6)}...] Updated gas price to ${currentGasPrice} wei for next batch`);
            }
            
            console.log(`[${wallet.address.slice(0, 6)}...] Progress: ${processedTx}/${transactionsPerWallet} transactions processed`);
            await new Promise(resolve => setTimeout(resolve, 200));
        } catch (err) {
            console.error(`[${wallet.address.slice(0, 6)}...] Batch error:`, err);
            const newNonce = await provider.getTransactionCount(wallet.address, "latest");
            console.log(`[${wallet.address.slice(0, 6)}...] Resyncing nonce. Old: ${currentNonce}, New: ${newNonce}`);
            currentNonce = newNonce;
            await updateGasPrice();
            console.log(`[${wallet.address.slice(0, 6)}...] Updated gas price to ${currentGasPrice} wei for next batch`);
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
    }
    
    return processedTx;
}

// Main transaction sending loop with multi-wallet processing
async function sendTransactions(wallets) {
    const startTime = Date.now();
    const walletsCount = wallets.length;
    const txPerWallet = Math.ceil(TOTAL_TRANSACTIONS / walletsCount);
    
    console.log(`Distributing ${TOTAL_TRANSACTIONS} transactions across ${walletsCount} wallets (${txPerWallet} per wallet)`);
    
    // Run flooding for each wallet in parallel
    const walletPromises = wallets.map((wallet, index) => {
        return runWalletFlooding(wallet, txPerWallet)
            .then(count => {
                console.log(`Wallet ${index} completed ${count} transactions`);
                return count;
            })
            .catch(err => {
                console.error(`Error in wallet ${index}:`, err);
                return 0;
            });
    });
    
    const results = await Promise.all(walletPromises);
    const totalProcessed = results.reduce((sum, count) => sum + count, 0);
    
    const duration = (Date.now() - startTime) / 1000;
    console.log(`Process completed. ${totalProcessed} transactions processed.`);
    console.log(`Total execution time: ${duration}s`);
    console.log(`Average TPS: ${totalProcessed / duration}`);
}

// Execution wrapper
(async () => {
    try {
        console.log("Starting parallel transaction flooding test");
        console.log(`Creating and funding ${WALLET_COUNT} wallets...`);
        
        const wallets = await createWallets();
        await sendTransactions(wallets);
        
        console.log("Transaction flooding complete. Wallet information saved for recovery.");
    } catch (err) {
        console.error("Fatal error:", err);
    }
})();
```
<br>
<br>

## Tx FLOODING - Recover WMTx
Will loop:
* searching for files from previous flooding runs containing subwallet information
* fetching back funds from subwallets
* rename files to processed when completed so won't recheck
<br>

```javascript
//RECOVER ANY FUNDS LEFT IN WALLETS POST TX FLOODING RUN
// Script to recover funds from temporary wallets back to main wallet
const { ethers } = require("ethers");
const fs = require('fs');
const path = require('path');

// Configurations
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const MAIN_WALLET_PRIVATE_KEY = "INSERT_PRIVATE_KEY_HERE"; // Your main wallet private key

// Instead of a fixed buffer, we'll use a multiplier for the gas price
// Since gas on this L3 chain is extremely low, use a higher multiplier to account for potential fluctuations
const GAS_BUFFER_MULTIPLIER = 3.0; // 300% of current gas price for ultra-low-cost L3
const STANDARD_GAS_LIMIT = BigInt(21000); // Standard transfer gas limit

// Pattern to match wallet files with timestamps (YYYYMMDD.HHMMSS)
const WALLET_FILE_PATTERN = /^generated-wallets\.\d{8}\.\d{6}\.json$/;
// Pattern to match already processed files
const PROCESSED_FILE_PATTERN = /^empty-generated-wallets/;
// Fallback to the old filename if no timestamp files are found
const DEFAULT_WALLET_FILE = "generated-wallets.json";

// Function to find all unprocessed wallet files
function findUnprocessedWalletFiles() {
  const files = fs.readdirSync('./');
  const results = {
    timestampedFiles: [],
    hasDefaultFile: false
  };
  
  // Find files matching the timestamp pattern, excluding already processed files
  results.timestampedFiles = files.filter(file => 
    WALLET_FILE_PATTERN.test(file) && !PROCESSED_FILE_PATTERN.test(file)
  );
  
  // Sort files by timestamp (most recent first)
  if (results.timestampedFiles.length > 0) {
    results.timestampedFiles.sort((a, b) => {
      // Extract timestamps from filenames
      const timestampA = a.split('.').slice(1, 3).join('');
      const timestampB = b.split('.').slice(1, 3).join('');
      // Compare in reverse order to get most recent first
      return timestampB.localeCompare(timestampA);
    });
    
    console.log(`Found ${results.timestampedFiles.length} unprocessed timestamped wallet files.`);
  }
  
  // Check if default file exists and is not already processed
  if (files.includes(DEFAULT_WALLET_FILE) && !files.includes(`empty-${DEFAULT_WALLET_FILE}`)) {
    results.hasDefaultFile = true;
    console.log(`Found unprocessed default wallet file: ${DEFAULT_WALLET_FILE}`);
  }
  
  return results;
}

// Process a single wallet file
async function processWalletFile(walletFilePath, mainWallet, provider, baseGasPrice, finalGasBuffer, totalRecovered, totalWalletsProcessed, processedFiles) {
  // Load wallet data from file
  let walletData;
  try {
    const fileContent = fs.readFileSync(walletFilePath, 'utf8');
    walletData = JSON.parse(fileContent);
    console.log(`Loaded ${walletData.length} wallets from ${walletFilePath}`);
  } catch (error) {
    console.error(`Error loading wallet file: ${error.message}`);
    console.log(`Make sure ${walletFilePath} exists and is valid JSON`);
    return;
  }
  
  // Filter out the main wallet by address and by wallets already marked as emptied
  const generatedWallets = walletData.filter(wallet => 
    wallet.address.toLowerCase() !== mainWallet.address.toLowerCase() && 
    wallet.emptied === false
  );
  
  console.log(`Found ${generatedWallets.length} generated wallets to recover funds from`);
  
  // Connect each wallet and track original data index for updating
  const fundedWallets = generatedWallets.map(wallet => ({
    wallet: new ethers.Wallet(wallet.privateKey, provider),
    originalIndex: walletData.findIndex(w => w.address === wallet.address)
  }));
  
  // Process each funded wallet
  let fileRecovered = BigInt(0);
  let fileRecoveredWallets = 0;
  let walletUpdates = []; // Track which wallets need to be marked as emptied
  
  for (const { wallet, originalIndex } of fundedWallets) {
    try {
      console.log(`\nChecking wallet ${wallet.address}...`);
      
      // Get balance with explicit debugging
      const balanceHex = await provider.send("eth_getBalance", [wallet.address, "latest"]);
      console.log(`Raw balance response: ${balanceHex}`);
      const balance = BigInt(balanceHex);
      console.log(`Wallet balance: ${ethers.formatEther(balance)} WMTX`);
      
      if (balance <= finalGasBuffer) {
        console.log(`Skipping wallet ${wallet.address} - insufficient balance (${ethers.formatEther(balance)} WMTX)`);
        continue;
      }
      
      // Calculate gas cost for the transfer
      const gasLimit = STANDARD_GAS_LIMIT;
      const gasCost = baseGasPrice * gasLimit;
      
      console.log(`Estimated gas cost: ${ethers.formatEther(gasCost)} WMTX`);
      
      // Calculate amount to send (leave enough for gas plus buffer)
      const amountToSend = balance - gasCost - finalGasBuffer;
      
      if (amountToSend <= BigInt(0)) {
        console.log(`  Skipping - not enough to cover transfer costs plus buffer`);
        continue;
      }
      
      console.log(`  Recovering ${ethers.formatEther(amountToSend)} WMTX`);
      
      // Send funds back to main wallet
      const tx = await wallet.sendTransaction({
        to: mainWallet.address,
        value: amountToSend,
        gasLimit: gasLimit
      });
      
      console.log(`  Transaction submitted: ${tx.hash}`);
      await tx.wait();
      console.log(`  Transaction confirmed`);
      
      // Mark this wallet as emptied for updating the JSON file later
      walletUpdates.push(originalIndex);
      
      fileRecovered += amountToSend;
      totalRecovered += amountToSend;
      fileRecoveredWallets++;
      totalWalletsProcessed++;
      
    } catch (error) {
      console.error(`Error processing wallet ${wallet.address}:`, error.message);
      if (error.info && error.info.error) {
        console.error(`  Error details: ${JSON.stringify(error.info.error)}`);
      }
    }
  }
  
  // Always update the wallet JSON file, regardless of whether funds were recovered
  console.log(`\nProcessing wallet file ${walletFilePath}...`);
  
  // Mark updated wallets as emptied
  if (walletUpdates.length > 0) {
    console.log(`Marking ${walletUpdates.length} wallets as emptied`);
    for (const index of walletUpdates) {
      walletData[index].emptied = true;
    }
  }
  
  // Mark all remaining wallets as emptied (those that may have been skipped due to insufficient funds)
  let additionalWalletsMarked = 0;
  for (let i = 0; i < walletData.length; i++) {
    const wallet = walletData[i];
    // Skip main wallet and wallets already marked as emptied
    if (wallet.isMain === true || wallet.emptied === true) {
      continue;
    }
    wallet.emptied = true;
    additionalWalletsMarked++;
  }
  
  if (additionalWalletsMarked > 0) {
    console.log(`Marked an additional ${additionalWalletsMarked} skipped wallets as emptied`);
  }
  
  // After marking all wallets, all non-main wallets should be emptied
  const allNonMainWalletsEmptied = walletData.every(wallet => 
    wallet.isMain === true || wallet.emptied === true
  );
  
  console.log("All non-main wallets have been marked as emptied.");
  
  // Mask the private key of the main wallet (if present)
  const mainWalletIndex = walletData.findIndex(wallet => wallet.isMain === true);
  
  if (mainWalletIndex !== -1) {
    const privateKey = walletData[mainWalletIndex].privateKey;
    
    // Keep first and last 6 characters, mask the rest with 'x'
    if (privateKey.length > 12) {  // Ensure it's long enough to mask
      const firstSix = privateKey.substring(0, 6);
      const lastSix = privateKey.substring(privateKey.length - 6);
      const maskedMiddle = 'x'.repeat(privateKey.length - 12);
      walletData[mainWalletIndex].privateKey = firstSix + maskedMiddle + lastSix;
      console.log("Main wallet private key has been masked.");
    }
  } else {
    console.log("No main wallet found in the wallet file.");
  }
  
  // Write the updated data back to the file
  try {
    fs.writeFileSync(walletFilePath, JSON.stringify(walletData, null, 2), 'utf8');
    console.log(`Successfully updated wallet data file.`);
    
    // Rename the file to indicate it's been emptied, keeping the original timestamp
    const baseName = path.basename(walletFilePath);
    const dirName = path.dirname(walletFilePath);
    const emptyFileName = path.join(dirName, `empty-${baseName}`);
    
    fs.renameSync(walletFilePath, emptyFileName);
    console.log(`Renamed processed file to ${emptyFileName}`);
    
    // Add to processed files list
    processedFiles.push(emptyFileName);
    
    // File summary
    console.log(`\n----- FILE SUMMARY -----`);
    console.log(`File: ${walletFilePath} → ${emptyFileName}`);
    console.log(`Wallets processed: ${fundedWallets.length}`);
    console.log(`Wallets with funds recovered: ${fileRecoveredWallets}`);
    console.log(`WMTX recovered from this file: ${ethers.formatEther(fileRecovered)} WMTX`);
    
    return emptyFileName;
  } catch (error) {
    console.error(`Error updating/renaming wallet file: ${error.message}`);
    return null;
  }
}

async function main() {
  console.log("Starting fund recovery process...");
  
  // Connect to the network
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const network = await provider.getNetwork();
  console.log(`Connected to network: ${network.name} (${network.chainId})`);
  
  // Set up main wallet
  const mainWallet = new ethers.Wallet(MAIN_WALLET_PRIVATE_KEY, provider);
  console.log(`Main wallet: ${mainWallet.address}`);
  
  // Get main wallet balance
  const mainBalanceBefore = await provider.getBalance(mainWallet.address);
  console.log(`Main wallet starting balance: ${ethers.formatEther(mainBalanceBefore)} WMTX`);
  
  // Find all unprocessed wallet files
  const walletFiles = findUnprocessedWalletFiles();
  
  if (walletFiles.timestampedFiles.length === 0 && !walletFiles.hasDefaultFile) {
    console.error('No unprocessed wallet files found. Exiting...');
    return;
  }
  
  let totalRecovered = BigInt(0);
  let totalWalletsProcessed = 0;
  let totalFilesProcessed = 0;
  let processedFiles = [];
  
  // Get current gas price and calculate dynamic buffer
  const feeData = await provider.getFeeData();
  let baseGasPrice;
  
  if (feeData.maxFeePerGas) {
    baseGasPrice = feeData.maxFeePerGas;
    console.log(`Using maxFeePerGas: ${ethers.formatUnits(baseGasPrice, "gwei")} gwei`);
  } else if (feeData.gasPrice) {
    baseGasPrice = feeData.gasPrice;
    console.log(`Using gasPrice: ${ethers.formatUnits(baseGasPrice, "gwei")} gwei`);
  } else {
    // Fallback to a conservative gas price estimate
    baseGasPrice = BigInt(10000000000); // 10 gwei
    console.log(`Using fallback gas price: 10 gwei`);
  }
  
  // Calculate a boosted gas price as buffer (300% of current gas price)
  const boostedGasPrice = (baseGasPrice * BigInt(Math.floor(GAS_BUFFER_MULTIPLIER * 100))) / BigInt(100);
  console.log(`Boosted gas price: ${ethers.formatUnits(boostedGasPrice, "gwei")} gwei`);
  
  // Calculate dynamic gas buffer (boosted gas price × gas limit)
  const dynamicGasBuffer = boostedGasPrice * STANDARD_GAS_LIMIT;
  console.log(`Dynamic gas buffer: ${ethers.formatEther(dynamicGasBuffer)} WMTX`);
  
  // Add a minimum buffer for ultra-low-cost L3 chain to handle extreme edge cases
  // This ensures we don't leave wallets completely empty even if gas is nearly zero
  const minBuffer = ethers.parseEther("0.0000001"); // Absolute minimum buffer
  const finalGasBuffer = dynamicGasBuffer > minBuffer ? dynamicGasBuffer : minBuffer;
  console.log(`Final gas buffer (with minimum): ${ethers.formatEther(finalGasBuffer)} WMTX`);
  
  // Process each timestamped file
  for (const walletFilePath of walletFiles.timestampedFiles) {
    console.log(`\n===== Processing file: ${walletFilePath} =====`);
    await processWalletFile(walletFilePath, mainWallet, provider, baseGasPrice, finalGasBuffer, totalRecovered, totalWalletsProcessed, processedFiles);
    totalFilesProcessed++;
  }
  
  // Process default file if it exists and is unprocessed
  if (walletFiles.hasDefaultFile) {
    console.log(`\n===== Processing default file: ${DEFAULT_WALLET_FILE} =====`);
    await processWalletFile(DEFAULT_WALLET_FILE, mainWallet, provider, baseGasPrice, finalGasBuffer, totalRecovered, totalWalletsProcessed, processedFiles);
    totalFilesProcessed++;
  }
  
  // Show summary
  const mainBalanceAfter = await provider.getBalance(mainWallet.address);
  console.log("\n======= RECOVERY SUMMARY =======");
  console.log(`Files processed: ${totalFilesProcessed}`);
  console.log(`Total wallets processed: ${totalWalletsProcessed}`);
  console.log(`Files renamed:`);
  processedFiles.forEach(file => {
    console.log(`  - ${file}`);
  });
  console.log(`Total WMTX recovered: ${ethers.formatEther(totalRecovered)} WMTX`);
  console.log(`Main wallet starting balance: ${ethers.formatEther(mainBalanceBefore)} WMTX`);
  console.log(`Main wallet ending balance: ${ethers.formatEther(mainBalanceAfter)} WMTX`);
  console.log(`Net change: ${ethers.formatEther(mainBalanceAfter - mainBalanceBefore)} WMTX`);
}

main().catch(console.error);

```
