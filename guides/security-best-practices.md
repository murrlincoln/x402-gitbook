# Security Best Practices

When implementing x402 payment systems, security should be your top priority. This guide covers essential security considerations for both sellers and buyers.

## For Sellers

### Wallet Security

**Private Key Management**
- Never store private keys in your application code or environment variables in production
- Use hardware security modules (HSMs) or secure key management services
- Implement key rotation policies
- Consider using multi-signature wallets for high-value transactions

**Receiving Address Validation**
- Verify that payment destinations match your expected receiving addresses
- Implement address whitelisting for automated systems
- Monitor all incoming transactions for anomalies

### Payment Verification

**Always Verify Before Serving**
- Never trust client-provided payment proofs without verification
- Use the facilitator's `/verify` endpoint for all payment validations
- Implement proper error handling for failed verifications
- Cache verification results appropriately to avoid replay attacks

**Rate Limiting**
```javascript
// Example: Implement rate limiting per wallet address
const rateLimit = new Map();

function checkRateLimit(walletAddress) {
  const now = Date.now();
  const requests = rateLimit.get(walletAddress) || [];

  // Remove old requests (older than 1 minute)
  const recentRequests = requests.filter(time => now - time < 60000);

  if (recentRequests.length >= 10) {
    throw new Error('Rate limit exceeded');
  }

  recentRequests.push(now);
  rateLimit.set(walletAddress, recentRequests);
}
```

### Network Security

**HTTPS Everywhere**
- Always use HTTPS for all x402 endpoints
- Implement proper TLS certificate validation
- Use HTTP Strict Transport Security (HSTS) headers

**Input Validation**
- Validate all incoming payment payloads
- Sanitize user inputs before processing
- Implement proper schema validation for payment requests

### Monitoring and Logging

**Transaction Monitoring**
- Log all payment attempts (successful and failed)
- Monitor for unusual payment patterns
- Set up alerts for failed verification attempts
- Track payment amounts and frequencies

**Security Logging**
```javascript
// Example: Security-focused logging
function logPaymentAttempt(request, result) {
  console.log({
    timestamp: new Date().toISOString(),
    clientIP: request.ip,
    userAgent: request.headers['user-agent'],
    paymentAmount: result.amount,
    walletAddress: result.fromAddress,
    verified: result.success,
    facilitator: result.facilitatorUsed
  });
}
```

## For Buyers

### Wallet Protection

**Secure Key Storage**
- Use encrypted storage for private keys
- Implement proper key derivation functions
- Consider using hardware wallets for high-value operations
- Never share or expose private keys

**Transaction Signing**
- Verify transaction details before signing
- Implement transaction preview mechanisms
- Use deterministic signing to prevent replay attacks
- Validate recipient addresses

### Payment Verification

**Facilitator Trust**
- Only use trusted facilitators
- Verify facilitator certificates and reputation
- Implement failover mechanisms for facilitator unavailability
- Monitor facilitator behavior for anomalies

**Amount Validation**
```javascript
// Example: Verify payment amounts match expectations
function validatePaymentAmount(requested, actual, tolerance = 0.001) {
  const difference = Math.abs(requested - actual);
  if (difference > tolerance) {
    throw new Error(`Payment amount mismatch: expected ${requested}, got ${actual}`);
  }
}
```

## Network-Level Security

### Smart Contract Interactions

**Contract Verification**
- Always verify smart contract addresses
- Check contract source code when possible
- Monitor for contract upgrades or changes
- Implement multi-signature requirements for critical operations

**Gas Management**
- Set appropriate gas limits and prices
- Monitor gas usage patterns
- Implement gas price oracles for dynamic pricing
- Consider Layer 2 solutions for cost optimization

### Privacy Considerations

**Transaction Privacy**
- Be aware of on-chain transaction visibility
- Consider using privacy-preserving payment methods when needed
- Implement proper data handling for user information
- Follow applicable privacy regulations (GDPR, CCPA, etc.)

## Incident Response

### Security Incident Handling

**Preparation**
- Establish incident response procedures
- Identify key personnel and contact information
- Prepare communication templates
- Set up monitoring and alerting systems

**Response Steps**
1. **Immediate Actions**
   - Isolate affected systems
   - Stop processing payments if necessary
   - Preserve evidence and logs
   - Notify relevant stakeholders

2. **Investigation**
   - Analyze transaction logs
   - Identify the scope of the incident
   - Determine root cause
   - Document findings

3. **Recovery**
   - Implement fixes and patches
   - Resume operations safely
   - Monitor for recurring issues
   - Update security measures

## Compliance and Legal

### Regulatory Considerations

**Know Your Customer (KYC)**
- Understand applicable KYC requirements
- Implement appropriate identity verification
- Maintain proper record-keeping
- Consider jurisdiction-specific regulations

**Anti-Money Laundering (AML)**
- Implement transaction monitoring
- Report suspicious activities as required
- Maintain transaction records
- Follow sanctions and prohibited persons lists

### Audit and Compliance

**Regular Security Audits**
- Conduct periodic security assessments
- Test incident response procedures
- Review access controls and permissions
- Update security policies and procedures

**Documentation**
- Maintain security policies and procedures
- Document system architecture and data flows
- Keep audit trails for all transactions
- Prepare for regulatory examinations

## Tools and Resources

### Security Tools
- [Slither](https://github.com/crytic/slither) - Smart contract static analyzer
- [MythX](https://mythx.io/) - Smart contract security analysis
- [OpenZeppelin](https://openzeppelin.com/) - Secure smart contract libraries
- [Consensys Diligence](https://consensys.net/diligence/) - Security auditing services

### Monitoring Solutions
- [Forta](https://forta.org/) - Real-time threat detection
- [OpenZeppelin Defender](https://defender.openzeppelin.com/) - Security operations
- [Chainlink VRF](https://chain.link/vrf) - Verifiable randomness
- [Tenderly](https://tenderly.co/) - Smart contract monitoring

Remember: Security is an ongoing process, not a one-time implementation. Stay updated with the latest security practices and regularly review your implementation as the x402 ecosystem evolves.