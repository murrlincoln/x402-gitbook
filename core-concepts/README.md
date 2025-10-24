# Core Concepts

x402 relies on a few foundational ideas that work together to enable permissionless, usage-based payments over HTTP. This section introduces each component at a high level and links to the detailed guides.

## Key Building Blocks

- **[HTTP 402](http-402.md):** How x402 uses the `402 Payment Required` status code to negotiate payment before a response is returned.
- **[Client / Server](client-server.md):** The buyer and seller responsibilities during the payment handshake.
- **[Facilitator](facilitator.md):** The service that verifies and settles payments on behalf of sellers.
- **[Wallet](wallet.md):** How buyers manage the credentials and funds needed to send payments.
- **[Bazaar (Discovery Layer)](bazaar-discovery-layer.md):** A marketplace pattern for publishing and discovering x402-compatible services.
- **[Network & Token Support](network-and-token-support.md):** Supported chains, assets, and settlement schemes today.

Use these chapters to understand how x402 fits together before diving into implementation specifics.
