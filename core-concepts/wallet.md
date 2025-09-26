# Wallet

This page explains the role of the **wallet** in the x402 protocol and provides practical examples for integration.

In x402, a wallet is both a payment mechanism and a form of unique identity for buyers and sellers. Wallet addresses are used to send, receive, and verify payments, while also serving as identifiers within the protocol.

## Role of the Wallet

### For Buyers

Buyers use wallets to:

* Store USDC/crypto for payments
* Sign payment payloads cryptographically
* Authorize onchain payments programmatically
* Maintain payment history and transaction records

Wallets enable buyers, including AI agents, to transact without account creation or credential management.

### For Sellers

Sellers use wallets to:

* Receive USDC/crypto payments from buyers
* Define their payment destination within server configurations
* Track incoming payments and revenue
* Manage multi-signature setups for enhanced security

A seller's wallet address is included in the payment requirements provided to buyers.

## Wallet Integration Examples

### Setting Up a Wallet for Buyers

```javascript
import { Wallet } from 'ethers';
import { createX402Client } from '@coinbase/x402';

// Create or import a wallet
const wallet = new Wallet(process.env.PRIVATE_KEY, provider);

// Initialize x402 client with wallet
const client = createX402Client({
  wallet: wallet,
  facilitator: "https://x402.org/facilitator"
});

// Check wallet balance before making payments
async function checkBalance() {
  const balance = await wallet.getBalance();
  const usdcBalance = await usdcContract.balanceOf(wallet.address);

  console.log(`ETH Balance: ${ethers.utils.formatEther(balance)}`);
  console.log(`USDC Balance: ${ethers.utils.formatUnits(usdcBalance, 6)}`);
}
```

### Configuring a Wallet for Sellers

```javascript
// Server configuration
const RECIPIENT_WALLET = process.env.RECIPIENT_ADDRESS;

app.use(paymentMiddleware(
  RECIPIENT_WALLET, // Your receiving wallet address
  {
    "GET /premium-content": {
      price: "$0.05",
      network: "base-sepolia",
    }
  }
));

// Monitor incoming payments
async function monitorPayments() {
  const filter = usdcContract.filters.Transfer(null, RECIPIENT_WALLET);

  usdcContract.on(filter, (from, to, amount, event) => {
    console.log(`Payment received: ${ethers.utils.formatUnits(amount, 6)} USDC from ${from}`);
    // Update your records, trigger fulfillment, etc.
  });
}
```

### Wallet Security Best Practices

#### For Production Use

```javascript
// Use environment variables for private keys (development only)
const wallet = new Wallet(process.env.PRIVATE_KEY, provider);

// Better: Use hardware wallets or key management services
import { LedgerSigner } from '@ethersproject/hardware-wallets';

const ledger = new LedgerSigner(provider, "hid", "m/44'/60'/0'/0/0");

// Or use cloud key management
import { KmsSigner } from '@aws-sdk/kms-signer';

const kmsSigner = new KmsSigner({
  keyId: process.env.KMS_KEY_ID,
  region: 'us-east-1'
});
```

#### Multi-signature Wallets

```javascript
// For high-value seller wallets, consider multi-sig
const MULTISIG_WALLET = "0x742d35Cc6554C6FaA94f0678DD7C5A4B8A6E3A1";

// Configure multiple signers
const signers = [
  new Wallet(process.env.SIGNER_1_KEY, provider),
  new Wallet(process.env.SIGNER_2_KEY, provider),
  new Wallet(process.env.SIGNER_3_KEY, provider)
];

// Require 2 of 3 signatures for withdrawals
const multiSigContract = new ethers.Contract(
  MULTISIG_WALLET,
  multiSigABI,
  provider
);
```

## Wallet Types and Recommendations

### Development and Testing

**MetaMask/Browser Wallets**
- Good for: Development and testing
- Pros: Easy setup, familiar interface
- Cons: Manual transaction approval required

**Programmatic Wallets**
- Good for: Automated testing, CI/CD
- Pros: Fully automated, scriptable
- Cons: Private key management required

### Production Environments

**Hardware Wallets (Ledger/Trezor)**
- Good for: High-security seller wallets
- Pros: Private keys never leave device
- Cons: Requires physical device access

**Cloud Key Management (AWS KMS, Azure Key Vault)**
- Good for: Scalable production deployments
- Pros: Enterprise security, compliance
- Cons: Cloud dependency, complexity

**Multi-signature Wallets**
- Good for: High-value treasury management
- Pros: Distributed security, governance
- Cons: Multiple signatures required for transactions

## Wallet Management Utilities

### Balance Monitoring

```javascript
class WalletMonitor {
  constructor(wallet, tokens = []) {
    this.wallet = wallet;
    this.tokens = tokens;
  }

  async getBalances() {
    const balances = {
      eth: await this.wallet.getBalance(),
      tokens: {}
    };

    for (const token of this.tokens) {
      const contract = new ethers.Contract(token.address, ERC20_ABI, this.wallet);
      balances.tokens[token.symbol] = await contract.balanceOf(this.wallet.address);
    }

    return balances;
  }

  async monitorLowBalance(threshold = ethers.utils.parseEther("0.01")) {
    const balance = await this.wallet.getBalance();

    if (balance.lt(threshold)) {
      console.warn(`Low balance warning: ${ethers.utils.formatEther(balance)} ETH`);
      // Send alert, auto-refill, etc.
    }
  }
}
```

### Transaction History

```javascript
async function getPaymentHistory(walletAddress, startBlock = 0) {
  const filter = {
    address: USDC_CONTRACT_ADDRESS,
    topics: [
      ethers.utils.id("Transfer(address,address,uint256)"),
      null, // from (any address)
      ethers.utils.hexZeroPad(walletAddress, 32) // to our wallet
    ],
    fromBlock: startBlock
  };

  const logs = await provider.getLogs(filter);

  return logs.map(log => {
    const decoded = usdcContract.interface.parseLog(log);
    return {
      from: decoded.args.from,
      amount: ethers.utils.formatUnits(decoded.args.value, 6),
      txHash: log.transactionHash,
      blockNumber: log.blockNumber
    };
  });
}
```

## Recommended Wallet Solutions

### For Developers

**[CDP Wallet API](https://docs.cdp.coinbase.com/wallet-api-v2/docs/welcome)** - Our recommended option for programmatic payments and secure key management.

**[Ethers.js Wallet](https://docs.ethers.io/v5/api/signer/#Wallet)** - Simple, reliable wallet implementation for Node.js applications.

**[Web3.js](https://web3js.readthedocs.io/)** - Alternative JavaScript wallet library with broad ecosystem support.

### For Production

**[Fireblocks](https://www.fireblocks.com/)** - Enterprise-grade wallet infrastructure with institutional security.

**[AWS KMS](https://aws.amazon.com/kms/)** - Cloud-native key management for scalable applications.

**[Gnosis Safe](https://safe.global/)** - Multi-signature wallet for shared custody and governance.

## Summary

* Wallets enable programmatic, permissionless payments in x402
* Buyers use wallets to authorize and send payments automatically
* Sellers use wallets to receive payments and manage treasury
* Wallet addresses serve as unique identifiers within the protocol
* Security considerations vary significantly between development and production use
* Choose wallet solutions based on your security requirements and operational needs

The wallet is the foundation of trust and identity in the x402 ecosystem, making secure wallet management critical for successful implementations.
