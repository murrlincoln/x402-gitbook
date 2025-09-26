# Troubleshooting Guide

This guide helps you diagnose and resolve common issues when implementing or using x402 payment systems.

## Payment Verification Issues

### Problem: "Payment verification failed"

**Symptoms:**
- 402 response received but payment doesn't verify
- Facilitator returns verification errors
- Transactions appear on-chain but server rejects them

**Common Causes & Solutions:**

1. **Incorrect Payment Payload Format**
   ```javascript
   // ❌ Incorrect - missing required fields
   const payload = {
     amount: "1000000", // Missing token, recipient, etc.
   };

   // ✅ Correct - complete payload structure
   const payload = {
     amount: "1000000",
     token: "0xA0b86a33E6427B0655DEAD94DD2584768DED86b",
     recipient: "0x742d35Cc6554C6FaA94f0678DD7C5A4B8A6E3A1",
     nonce: "12345",
     signature: "0x...",
     // ... other required fields
   };
   ```

2. **Network Mismatch**
   ```javascript
   // Ensure client and server use same network
   const config = {
     network: "base-sepolia", // Must match on both sides
     facilitator: "https://x402.org/facilitator"
   };
   ```

3. **Expired Payment**
   - Check timestamp in payment payload
   - Verify payment was submitted within timeout window
   - Adjust `maxTimeoutSeconds` if needed

### Problem: "Insufficient funds" during payment

**Diagnosis:**
```javascript
// Check wallet balance
const balance = await wallet.getBalance(tokenAddress);
console.log(`Wallet balance: ${balance}`);
console.log(`Required amount: ${paymentAmount}`);
console.log(`Gas estimate: ${gasEstimate}`);
```

**Solutions:**
- Fund wallet with required tokens
- Account for gas fees in balance calculations
- Use Layer 2 networks for lower costs

## Network and Connection Issues

### Problem: Facilitator connection timeouts

**Symptoms:**
- Network timeout errors
- HTTP 5xx responses from facilitator
- Slow payment processing

**Troubleshooting Steps:**

1. **Check Facilitator Status**
   ```bash
   curl -I https://x402.org/facilitator/health
   ```

2. **Test Network Connectivity**
   ```javascript
   // Add timeout and retry logic
   const fetchWithTimeout = async (url, options, timeout = 5000) => {
     const controller = new AbortController();
     const timeoutId = setTimeout(() => controller.abort(), timeout);

     try {
       const response = await fetch(url, {
         ...options,
         signal: controller.signal
       });
       return response;
     } finally {
       clearTimeout(timeoutId);
     }
   };
   ```

3. **Implement Circuit Breaker**
   ```javascript
   class CircuitBreaker {
     constructor(failureThreshold = 5, resetTimeout = 60000) {
       this.failureThreshold = failureThreshold;
       this.resetTimeout = resetTimeout;
       this.failures = 0;
       this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
       this.nextAttempt = 0;
     }

     async call(fn) {
       if (this.state === 'OPEN') {
         if (Date.now() < this.nextAttempt) {
           throw new Error('Circuit breaker is OPEN');
         }
         this.state = 'HALF_OPEN';
       }

       try {
         const result = await fn();
         this.onSuccess();
         return result;
       } catch (error) {
         this.onFailure();
         throw error;
       }
     }

     onSuccess() {
       this.failures = 0;
       this.state = 'CLOSED';
     }

     onFailure() {
       this.failures++;
       if (this.failures >= this.failureThreshold) {
         this.state = 'OPEN';
         this.nextAttempt = Date.now() + this.resetTimeout;
       }
     }
   }
   ```

## Smart Contract Issues

### Problem: Transaction reverts or fails

**Common Revert Reasons:**

1. **ERC20 Approval Issues**
   ```javascript
   // Check and set approval before payment
   const allowance = await token.allowance(wallet.address, spenderAddress);
   if (allowance < amount) {
     await token.approve(spenderAddress, amount);
   }
   ```

2. **Gas Estimation Errors**
   ```javascript
   // Add buffer to gas estimates
   const gasEstimate = await contract.estimateGas.method(...args);
   const gasLimit = gasEstimate.mul(120).div(100); // Add 20% buffer
   ```

3. **Slippage Issues**
   ```javascript
   // Account for price movements
   const slippageTolerance = 0.005; // 0.5%
   const minAmountOut = expectedAmount * (1 - slippageTolerance);
   ```

### Problem: "Nonce too low" or "Nonce too high" errors

**Solutions:**
```javascript
// Get current nonce and queue transactions properly
const nonce = await wallet.getTransactionCount('pending');

// For multiple transactions, increment nonce manually
const tx1 = await wallet.sendTransaction({ ...txParams, nonce });
const tx2 = await wallet.sendTransaction({ ...txParams, nonce: nonce + 1 });
```

## Integration Issues

### Problem: Middleware not triggering

**Express.js Debugging:**
```javascript
// Add debugging middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.path}`);
  console.log('Headers:', req.headers);
  next();
});

// Ensure middleware order is correct
app.use(paymentMiddleware); // Must be before route handlers
app.get('/api/endpoint', handler);
```

**Next.js Debugging:**
```javascript
// Check middleware.ts configuration
export const config = {
  matcher: [
    '/api/protected/:path*',
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ]
};

// Verify middleware runs
export function middleware(request) {
  console.log('Middleware triggered for:', request.nextUrl.pathname);
  return paymentMiddleware(request);
}
```

### Problem: CORS errors in browser

**Server Configuration:**
```javascript
// Add CORS headers for x402 endpoints
app.use(cors({
  origin: ['http://localhost:3000', 'https://yourdomain.com'],
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-PAYMENT'],
  credentials: true
}));
```

## Development Environment Issues

### Problem: "Module not found" errors

**Common Solutions:**

1. **TypeScript Configuration**
   ```json
   // tsconfig.json
   {
     "compilerOptions": {
       "moduleResolution": "node",
       "esModuleInterop": true,
       "allowSyntheticDefaultImports": true,
       "skipLibCheck": true
     }
   }
   ```

2. **Package Installation**
   ```bash
   # Install all required dependencies
   npm install x402-express @coinbase/x402
   # or
   npm install x402-next @coinbase/x402
   ```

3. **Environment Variables**
   ```bash
   # .env.local
   NEXT_PUBLIC_RPC_URL=https://base-sepolia.g.alchemy.com/v2/your-key
   WALLET_PRIVATE_KEY=your-private-key-for-testing
   FACILITATOR_URL=https://x402.org/facilitator
   ```

### Problem: Rate limiting during development

**Solutions:**
```javascript
// Implement development-friendly rate limiting
const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: process.env.NODE_ENV === 'development' ? 1000 : 100,
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});
```

## Production Issues

### Problem: High gas costs

**Optimization Strategies:**

1. **Batch Transactions**
   ```javascript
   // Bundle multiple payments
   const batchPayment = await contract.batchTransfer(
     recipients,
     amounts,
     { gasLimit: estimatedGas }
   );
   ```

2. **Use Layer 2 Networks**
   ```javascript
   // Switch to more cost-effective networks
   const networks = {
     'ethereum': { chainId: 1, gasMultiplier: 1.0 },
     'base': { chainId: 8453, gasMultiplier: 0.1 },
     'arbitrum': { chainId: 42161, gasMultiplier: 0.1 }
   };
   ```

3. **Implement Gas Price Oracle**
   ```javascript
   async function getOptimalGasPrice() {
     try {
       const gasPrice = await provider.getGasPrice();
       const fastGasPrice = gasPrice.mul(110).div(100); // 10% higher for faster confirmation
       return fastGasPrice;
     } catch (error) {
       return ethers.utils.parseUnits('20', 'gwei'); // Fallback
     }
   }
   ```

## Monitoring and Logging

### Setting Up Comprehensive Logging

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'x402-api' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Log payment events
function logPaymentEvent(event, data) {
  logger.info('Payment event', {
    event,
    timestamp: Date.now(),
    ...data
  });
}
```

### Health Check Endpoint

```javascript
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    await db.ping();

    // Check facilitator connectivity
    const facilitatorStatus = await fetch(`${FACILITATOR_URL}/health`);

    // Check blockchain connectivity
    const blockNumber = await provider.getBlockNumber();

    res.json({
      status: 'healthy',
      timestamp: Date.now(),
      services: {
        database: 'connected',
        facilitator: facilitatorStatus.ok ? 'connected' : 'disconnected',
        blockchain: blockNumber > 0 ? 'connected' : 'disconnected'
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});
```

## Getting Help

If you're still experiencing issues after trying these solutions:

1. **Check the logs** - Enable verbose logging to identify the exact error
2. **Test in isolation** - Create minimal reproduction cases
3. **Review documentation** - Ensure you're following the latest implementation patterns
4. **Community support** - Join the [Discord community](https://discord.gg/invite/cdp) for help
5. **GitHub issues** - Report bugs or request features on the [x402 repository](https://github.com/coinbase/x402)

Remember to include relevant error messages, code snippets, and environment details when seeking help.