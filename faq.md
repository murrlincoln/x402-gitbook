# FAQ

### General

#### What _is_ x402 in a single sentence?

x402 is an open‑source protocol that turns the dormant HTTP `402 Payment Required` status code into a fully‑featured, on‑chain payment layer for APIs, websites, and autonomous agents.

**Is x402 a CDP Product?**

_No._ While Coinbase Developer Platform provides tooling and are the creators of the standard, it is an open protocol (Apache-2.0 license) and you don't need any Coinbase products to use it. We look forward to further clarifying this distinction and making x402 a credibly neutral payment standard.&#x20;

#### Why not use traditional payment rails or API keys?

Traditional rails require credit‑card networks, user accounts, and multi‑step UI flows.\
x402 removes those dependencies, enabling programmatic, HTTP-native payments (perfect for AI agents) while dropping fees to near‑zero and settling in \~1 second.

#### Is x402 only for crypto‑native projects?

No. Any web API or content provider—crypto or web2—can integrate x402 if it wants a lower‑cost, friction‑free payment path for small or usage‑based transactions.

### Language & Framework Support

#### What languages and frameworks are supported?

TypeScript and Python are **the reference implementations**, but x402 is an **open protocol**.

Nothing prevents you from implementing the spec in Go, Rust, etc. If you're interested in building support for your favorite language, please [open an issue](https://github.com/coinbase/x402/issues) and let us know, we'd be happy to help!

### Facilitators

#### Who runs facilitators today?

Coinbase Developer Platform operates the first production facilitator. The protocol, however, is **permissionless**—anyone can run a facilitator. Expect:

* Community‑run facilitators for other networks or assets.
* Private facilitators for enterprises that need custom KYT / KYC flows.

#### What stops a malicious facilitator from stealing funds or lying about settlement?

Every `x402PaymentPayload` is **signed by the buyer** and settles **directly on‑chain**.\
A facilitator that tampers with the transaction will fail signature checks.

### Pricing & Schemes

#### How should I price my endpoint?

There is no single answer, but common patterns are:

* **Flat per‑call** (e.g., `$0.001` per request)
* **Tiered** (`/basic` vs `/pro` endpoints with different prices)
* **Up‑to** (work in progress): "pay‑up‑to" where the final cost equals usage (tokens, MB, etc.)

#### Can I integrate x402 with a usage / plan manager like Metronome?

Yes. x402 handles the _payment execution_. You can still meter usage, aggregate calls, or issue prepaid credits in Metronome and only charge when limits are exceeded. Example glue code is coming soon.

### Assets, Networks & Fees

#### Which assets and networks are supported today?

| Network        | Asset | Fees\*   | Status      |
| ------------   | ----- | -------- | ----------- |
| Base           | USDC  | fee-free | **Mainnet** |
| Base Sepolia   | USDC  | fee-free | **Testnet** |
| Solana         | SPL Tokens  | fee-free | **Mainnet** |
| Solana Devnet | SPL Tokens  | fee-free | **Testnet** |

* Gas paid on chain; Coinbase's x402 facilitator adds **zero** facilitator fee.

_Support for additional chains and assets is on the roadmap and community‑driven._

#### Does x402 support fiat off‑ramps or credit‑card deposits?

Not natively. However, facilitators or third‑party gateways can wrap x402 flows with on‑ and off‑ramps.&#x20;

### Security

#### Do I have to expose my private key to my backend?

No. The recommended pattern is:

1. **Buyers (clients/agents)** sign locally in their runtime (browser, serverless, agent VM). You can use CDP Wallet API to create a programmatic wallet.
2. **Sellers** never hold the buyer's key; they only verify signatures.

#### How do refunds work?

The current `exact` scheme is a _push payment_—irreversible once executed. Two options:

1. **Business‑logic refunds:** Seller sends a new USDC transfer back to the buyer.
2. **Escrow schemes:** Future spec could add conditional transfers (e.g., HTLCs or hold invoices).

### Usage by AI Agents

#### How does an agent know what to pay?

Agents follow the same flow as humans:

1. Make a request.
2. Parse the `402` JSON (`accepts` array).
3. Choose a suitable requirement and sign a payload via the x402 client SDKs.
4. Retry with `X‑PAYMENT`.

#### Do agents need wallets?

Yes. Programmatic wallets (e.g., **CDP Wallet API**, **viem**, **ethers‑v6** HD wallets) let agents sign `EIP‑712` payloads without exposing seed phrases.

### Governance & Roadmap

#### Is there a formal spec or whitepaper?

* **Spec:** [GitHub Specification](https://github.com/coinbase/x402/tree/main/specs)
* [**Whitepaper**](https://www.x402.org/x402-whitepaper.pdf)

#### How will x402 evolve?

Tracked in public GitHub issues + community RFCs. Major themes:

* Multi‑asset support
* Additional schemes (`upto`, `stream`, `permit2`)
* Discovery layer for service search & reputation

**Why is x402 hosted in the Coinbase GitHub?**

We acknowledge that the repo is primarily under Coinbase ownership today. This is primarily to leverage our best-in-house security and auditing team to ensure the spec is safe and nobody accidentally creates legally ambiguous payment flows. We intend to eventually transfer ownership of the repo to a steering group or open-source committee.

### Troubleshooting

#### I keep getting `402 Payment Required`, even after attaching `X‑PAYMENT`. Why?

1. Signature is invalid (wrong chain ID or payload fields).
2. Payment amount < `maxAmountRequired`.
3. Address has insufficient USDC or was flagged by KYT.\
   Check the `error` field in the server's JSON response for details.

#### My test works on Base Sepolia but fails on Base mainnet—what changed?

* Ensure you set `network: "base"` (not `"base‑sepolia"`).
* Confirm your wallet has _mainnet_ USDC.
* Gas fees are higher on mainnet; fund the wallet with a small amount of ETH for gas.

### Development and Integration

#### What programming languages can I use with x402?

The x402 protocol itself is language-agnostic since it uses standard HTTP. Official SDKs are available for:
- **TypeScript/JavaScript**: Express, Next.js, Hono middleware
- **Python**: FastAPI and Flask middleware

Community implementations exist for other languages, and you can implement the protocol directly using any HTTP client library.

#### How do I handle failed payments or network issues?

Implement retry logic with exponential backoff:
```javascript
async function retryPayment(paymentFn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await paymentFn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, attempt)));
    }
  }
}
```

Also implement circuit breakers for facilitator connectivity issues and proper error handling for different failure modes.

#### Can I use x402 with existing authentication systems?

Yes! x402 payments can work alongside traditional auth:
```javascript
app.use('/api/premium', authenticateUser, paymentMiddleware, handler);
```

This allows you to have both authenticated users and anonymous payments, or require both authentication and payment for premium features.

#### How do I test x402 integration locally?

Use test networks like Base Sepolia with the test facilitator:
```javascript
const testConfig = {
  network: "base-sepolia",
  facilitator: "https://x402.org/facilitator",
  recipient: "0x742d35Cc6554C6FaA94f0678DD7C5A4B8A6E3A1"
};
```

Create test wallets with testnet tokens from faucets, and use them to simulate real payment flows.

#### What happens if a user pays but my server is down?

Payment verification happens on-chain, so payments are preserved even if your server is down. When your server comes back online:
1. The user can retry the request with the same payment proof
2. Your server validates the payment against the blockchain
3. The service is provided as normal

Implement idempotency to handle duplicate requests safely.

### Business and Pricing

#### How should I price my API endpoints?

Consider these factors:
- **Computational cost**: More expensive operations should cost more
- **Value provided**: Charge based on the value users receive
- **Market rates**: Research what similar services charge
- **User behavior**: High-volume users may need different pricing

Start with simple per-request pricing and iterate based on usage patterns.

#### Can I implement subscription-like models with x402?

Yes, several approaches work:
- **Time-based**: Users pay for access periods (hour/day/month)
- **Credit-based**: Users pre-purchase credits to spend over time
- **Usage-based**: Combine payments with rate limiting for fair usage

See the [Implementation Patterns](guides/implementation-patterns.md) guide for detailed examples.

#### How do I handle refunds or disputes?

x402 payments are on-chain transactions that cannot be automatically reversed. For refunds:
1. Implement a separate refund system in your application
2. Send refunds to the original payment address
3. Keep detailed transaction logs for dispute resolution
4. Consider escrow systems for high-value services

#### Can I do partial payments or installments?

Yes, you can implement various payment structures:
- **Staged payments**: Charge different amounts for different service phases
- **Micro-subscriptions**: Very small recurring payments
- **Usage tracking**: Accumulate usage and charge periodically

The key is designing your application logic to handle these patterns.

### Technical Operations

#### How do I monitor payment success rates?

Implement comprehensive logging and metrics:
```javascript
// Track payment metrics
const paymentMetrics = {
  attempts: 0,
  successes: 0,
  failures: 0,
  averageTime: 0
};

function logPaymentAttempt(success, duration) {
  paymentMetrics.attempts++;
  if (success) paymentMetrics.successes++;
  else paymentMetrics.failures++;

  // Update rolling average
  paymentMetrics.averageTime =
    (paymentMetrics.averageTime + duration) / 2;
}
```

Monitor facilitator response times, payment verification success rates, and user completion funnels.

#### What's the best way to handle high-volume payments?

For high traffic:
1. **Load balancing**: Distribute payment processing across multiple servers
2. **Caching**: Cache payment verifications temporarily to reduce facilitator calls
3. **Async processing**: Queue payment verifications for non-critical paths
4. **Rate limiting**: Implement smart rate limiting to prevent abuse
5. **Database optimization**: Index payment-related data properly

#### How do I handle different blockchain networks?

Configure network-specific settings:
```javascript
const networkConfigs = {
  'ethereum': { chainId: 1, facilitator: 'https://mainnet-facilitator.x402.org' },
  'base': { chainId: 8453, facilitator: 'https://base-facilitator.x402.org' },
  'arbitrum': { chainId: 42161, facilitator: 'https://arbitrum-facilitator.x402.org' }
};

function getNetworkConfig(network) {
  return networkConfigs[network] || networkConfigs['base'];
}
```

Consider gas costs, confirmation times, and user preferences when choosing networks.

#### How secure is x402?

x402 inherits the security properties of the underlying blockchain:
- **Payment proofs** are cryptographically signed and verifiable
- **Facilitators** cannot steal funds or forge payments
- **Double-spending** is prevented by blockchain consensus
- **Privacy** depends on the chosen blockchain's privacy features

Follow security best practices in the [Security Guide](guides/security-best-practices.md) for additional protection.

### Troubleshooting Common Issues

#### "Payment verification failed" - what does this mean?

Common causes and solutions:
1. **Network mismatch**: Ensure client and server use the same network
2. **Expired payment**: Check if payment was submitted within the timeout window
3. **Invalid signature**: Verify the wallet is signing transactions correctly
4. **Facilitator issues**: Check facilitator status and connectivity

See the [Troubleshooting Guide](guides/troubleshooting.md) for detailed debugging steps.

#### My middleware isn't triggering payments

Check these common issues:
1. **Middleware order**: Payment middleware must be registered before route handlers
2. **Route matching**: Ensure your route patterns match exactly
3. **HTTP methods**: Verify GET/POST methods are configured correctly
4. **Network configuration**: Confirm you're using the right network and facilitator URL

#### Payments are slow - how can I optimize?

Performance optimization strategies:
1. **Choose faster networks**: Base and Arbitrum have lower latency than Ethereum
2. **Optimize gas prices**: Use dynamic gas pricing for faster confirmations
3. **Cache verifications**: Temporarily cache payment proofs to avoid re-verification
4. **Async processing**: Don't block user experience waiting for blockchain confirmation

### Still have questions?

• Reach out in the [Discord channel](https://discord.gg/invite/cdp)\
• Open a GitHub Discussion or Issue in the [x402 repo](https://github.com/coinbase/x402)\
• Check the [Troubleshooting Guide](guides/troubleshooting.md) for technical issues\
• Review [Implementation Patterns](guides/implementation-patterns.md) for design guidance
