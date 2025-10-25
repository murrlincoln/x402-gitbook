# Implementation Patterns

This guide showcases common implementation patterns for x402 services, from simple pay-per-request APIs to complex subscription models.

## Pattern 1: Simple Pay-Per-Request API

The most straightforward x402 implementation - charge users for individual API calls.

### Use Cases
- Data lookup services
- Image/text processing APIs
- One-off computational tasks

### Implementation

```javascript
import express from 'express';
import { paymentMiddleware } from 'x402-express';

const app = express();

app.use(paymentMiddleware(
  process.env.RECIPIENT_ADDRESS,
  {
    "GET /weather/:city": {
      price: "$0.01",
      network: "base-sepolia",
      config: {
        description: "Weather data for any city",
        maxTimeoutSeconds: 60
      }
    }
  }
));

app.get('/weather/:city', async (req, res) => {
  const { city } = req.params;

  // Fetch weather data
  const weatherData = await fetchWeatherData(city);

  res.json({
    city,
    ...weatherData,
    timestamp: new Date().toISOString()
  });
});
```

### Benefits
- Simple to implement and understand
- Predictable costs for users
- No state management required

### Considerations
- Can be expensive for high-volume users
- No bulk discounts or loyalty incentives

## Pattern 2: Tiered Pricing by Complexity

Different endpoints with different pricing based on computational complexity or value provided.

### Use Cases
- AI/ML services with varying model sizes
- Image processing with different quality levels
- Data analysis with different detail levels

### Implementation

```javascript
const pricingTiers = {
  "POST /ai/analyze/basic": {
    price: "$0.02",
    description: "Basic sentiment analysis"
  },
  "POST /ai/analyze/advanced": {
    price: "$0.10",
    description: "Advanced NLP with entity extraction"
  },
  "POST /ai/analyze/premium": {
    price: "$0.50",
    description: "Full analysis with custom model training"
  }
};

app.use(paymentMiddleware(
  process.env.RECIPIENT_ADDRESS,
  pricingTiers
));

app.post('/ai/analyze/:tier', async (req, res) => {
  const { tier } = req.params;
  const { text } = req.body;

  let result;
  switch (tier) {
    case 'basic':
      result = await basicAnalysis(text);
      break;
    case 'advanced':
      result = await advancedAnalysis(text);
      break;
    case 'premium':
      result = await premiumAnalysis(text);
      break;
    default:
      return res.status(400).json({ error: 'Invalid tier' });
  }

  res.json({
    tier,
    analysis: result,
    processingTime: result.processingTime
  });
});
```

## Pattern 3: Credit-Based System

Users pre-purchase credits and consume them over multiple API calls.

### Use Cases
- High-volume API usage
- Variable pricing per operation
- Subscription-like experience without time limits

### Implementation

```javascript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// Credit purchase endpoints
const creditPackages = {
  "POST /credits/small": { credits: 100, price: "$5.00" },
  "POST /credits/medium": { credits: 500, price: "$20.00" },
  "POST /credits/large": { credits: 2000, price: "$60.00" }
};

app.use(paymentMiddleware(
  process.env.RECIPIENT_ADDRESS,
  creditPackages
));

// Credit purchase handlers
Object.entries(creditPackages).forEach(([endpoint, config]) => {
  const route = endpoint.split(' ')[1];
  app.post(route, async (req, res) => {
    const { walletAddress } = req.body;

    await redis.incrby(`credits:${walletAddress}`, config.credits);
    const newBalance = await redis.get(`credits:${walletAddress}`);

    res.json({
      creditsAdded: config.credits,
      newBalance: parseInt(newBalance),
      cost: config.price
    });
  });
});

// Credit-consuming middleware
async function consumeCredits(cost) {
  return async (req, res, next) => {
    const walletAddress = req.headers['x-wallet-address'];

    if (!walletAddress) {
      return res.status(400).json({ error: 'Wallet address required' });
    }

    const credits = await redis.get(`credits:${walletAddress}`);
    const balance = parseInt(credits) || 0;

    if (balance < cost) {
      return res.status(402).json({
        error: 'Insufficient credits',
        required: cost,
        balance,
        purchaseOptions: creditPackages
      });
    }

    // Consume credits
    await redis.decrby(`credits:${walletAddress}`, cost);
    req.creditsUsed = cost;
    next();
  };
}

// API endpoints that consume credits
app.post('/api/process', consumeCredits(1), async (req, res) => {
  // Process the request
  const result = await processData(req.body);

  const remainingCredits = await redis.get(`credits:${req.headers['x-wallet-address']}`);

  res.json({
    result,
    creditsUsed: req.creditsUsed,
    creditsRemaining: parseInt(remainingCredits)
  });
});
```

## Pattern 4: Time-Based Subscriptions

Users purchase access for a specific time period.

### Use Cases
- Real-time data feeds
- Streaming services
- Premium feature access

### Implementation

```javascript
const subscriptions = new Map(); // In production, use a database

app.use(paymentMiddleware(
  process.env.RECIPIENT_ADDRESS,
  {
    "POST /subscribe/hour": {
      price: "$1.00",
      description: "1-hour premium access"
    },
    "POST /subscribe/day": {
      price: "$5.00",
      description: "24-hour premium access"
    },
    "POST /subscribe/month": {
      price: "$20.00",
      description: "30-day premium access"
    }
  }
));

app.post('/subscribe/:duration', (req, res) => {
  const { duration } = req.params;
  const { walletAddress } = req.body;

  const durations = {
    hour: 60 * 60 * 1000,
    day: 24 * 60 * 60 * 1000,
    month: 30 * 24 * 60 * 60 * 1000
  };

  const expiresAt = Date.now() + durations[duration];
  subscriptions.set(walletAddress, { expiresAt, plan: duration });

  res.json({
    subscribed: true,
    plan: duration,
    expiresAt: new Date(expiresAt).toISOString()
  });
});

// Subscription validation middleware
function requireSubscription(req, res, next) {
  const walletAddress = req.headers['x-wallet-address'];
  const subscription = subscriptions.get(walletAddress);

  if (!subscription || subscription.expiresAt < Date.now()) {
    return res.status(402).json({
      error: 'Active subscription required',
      subscriptionOptions: ['/subscribe/hour', '/subscribe/day', '/subscribe/month']
    });
  }

  req.subscription = subscription;
  next();
}

// Protected endpoints
app.get('/premium/data', requireSubscription, (req, res) => {
  res.json({
    data: getPremiumData(),
    subscription: req.subscription
  });
});
```

## Pattern 5: Usage-Based Billing with Rate Limits

Combine payments with rate limiting to create fair usage policies.

### Implementation

```javascript
import rateLimit from 'express-rate-limit';

// Different rate limits for different payment tiers
const createRateLimiter = (windowMs, max, price) => {
  return rateLimit({
    windowMs,
    max,
    keyGenerator: (req) => req.headers['x-wallet-address'] || req.ip,
    handler: (req, res) => {
      res.status(402).json({
        error: 'Rate limit exceeded',
        resetTime: new Date(Date.now() + windowMs),
        upgradeOption: {
          endpoint: '/upgrade',
          price: price,
          description: 'Pay to increase rate limit'
        }
      });
    }
  });
};

// Free tier: 10 requests per hour
app.use('/api/free', createRateLimiter(60 * 60 * 1000, 10, '$1.00'));

// Paid tier upgrade
app.use(paymentMiddleware(
  process.env.RECIPIENT_ADDRESS,
  {
    "POST /upgrade": {
      price: "$1.00",
      description: "Upgrade to 100 requests per hour"
    }
  }
));

const upgradeTracker = new Map();

app.post('/upgrade', (req, res) => {
  const { walletAddress } = req.body;
  const expiresAt = Date.now() + (60 * 60 * 1000); // 1 hour

  upgradeTracker.set(walletAddress, expiresAt);

  res.json({
    upgraded: true,
    expiresAt: new Date(expiresAt).toISOString(),
    newLimit: 100
  });
});

// Premium tier: 100 requests per hour for paying users
app.use('/api/premium', (req, res, next) => {
  const walletAddress = req.headers['x-wallet-address'];
  const upgrade = upgradeTracker.get(walletAddress);

  if (upgrade && upgrade > Date.now()) {
    // Use higher rate limit
    return createRateLimiter(60 * 60 * 1000, 100, '$5.00')(req, res, next);
  } else {
    // Use standard rate limit
    return createRateLimiter(60 * 60 * 1000, 10, '$1.00')(req, res, next);
  }
});
```

## Pattern 6: Dynamic Pricing

Adjust prices based on demand, computational cost, or other factors.

### Implementation

```javascript
class DynamicPricer {
  constructor() {
    this.basePrice = 0.01; // $0.01
    this.demandMultiplier = 1.0;
    this.requestCount = 0;
    this.resetInterval = 60000; // 1 minute

    // Reset demand multiplier periodically
    setInterval(() => {
      this.updateDemandMultiplier();
    }, this.resetInterval);
  }

  updateDemandMultiplier() {
    // Increase price during high demand
    if (this.requestCount > 100) {
      this.demandMultiplier = Math.min(3.0, this.demandMultiplier * 1.1);
    } else if (this.requestCount < 10) {
      this.demandMultiplier = Math.max(0.5, this.demandMultiplier * 0.9);
    }

    this.requestCount = 0;
  }

  getCurrentPrice(complexity = 1) {
    this.requestCount++;
    return (this.basePrice * complexity * this.demandMultiplier).toFixed(3);
  }
}

const pricer = new DynamicPricer();

// Dynamic pricing middleware
app.use('/api/dynamic', (req, res, next) => {
  const complexity = req.headers['x-complexity'] || 1;
  const currentPrice = pricer.getCurrentPrice(complexity);

  // Check if user has made payment for current price
  const paymentHeader = req.headers['x-payment'];
  if (!paymentHeader) {
    return res.status(402).json({
      error: 'Payment required',
      currentPrice: `$${currentPrice}`,
      demandLevel: pricer.demandMultiplier.toFixed(2),
      paymentRequired: {
        amount: currentPrice,
        currency: 'USD'
      }
    });
  }

  // Verify payment amount matches current price
  // (simplified - in practice, verify with facilitator)
  next();
});
```

## Pattern 7: Bulk Operations with Discount

Encourage larger purchases with volume discounts.

### Implementation

```javascript
function calculateBulkPrice(quantity) {
  const basePrice = 0.10;
  let discount = 0;

  if (quantity >= 100) discount = 0.20;      // 20% off for 100+
  else if (quantity >= 50) discount = 0.15;  // 15% off for 50+
  else if (quantity >= 10) discount = 0.10;  // 10% off for 10+

  const unitPrice = basePrice * (1 - discount);
  return {
    unitPrice: unitPrice.toFixed(3),
    totalPrice: (unitPrice * quantity).toFixed(2),
    discount: (discount * 100).toFixed(0) + '%',
    savings: ((basePrice - unitPrice) * quantity).toFixed(2)
  };
}

app.post('/api/bulk-quote', (req, res) => {
  const { quantity } = req.body;
  const pricing = calculateBulkPrice(quantity);

  res.json({
    quantity,
    ...pricing,
    validFor: '10 minutes'
  });
});

app.use(paymentMiddleware(
  process.env.RECIPIENT_ADDRESS,
  {
    "POST /api/bulk-process": {
      // Price calculated dynamically based on quantity
      price: (req) => {
        const quantity = req.body.quantity || 1;
        return `$${calculateBulkPrice(quantity).totalPrice}`;
      },
      config: {
        description: "Bulk processing with volume discounts"
      }
    }
  }
));
```

## Choosing the Right Pattern

### Consider These Factors:

**User Behavior**
- High-volume users → Credit system or subscriptions
- Sporadic usage → Pay-per-request
- Predictable usage → Time-based subscriptions

**Service Characteristics**
- Variable computational cost → Tiered pricing
- Real-time data → Subscriptions
- Batch processing → Bulk operations

**Business Goals**
- Revenue predictability → Subscriptions
- Usage growth → Credit systems
- Market penetration → Dynamic pricing with discounts

**Technical Complexity**
- Simple APIs → Pay-per-request
- Complex services → Multiple patterns combined

### Implementation Tips

1. **Start Simple**: Begin with pay-per-request, add complexity as needed
2. **Monitor Usage**: Track metrics to optimize pricing and patterns
3. **User Experience**: Provide clear pricing information and options
4. **Flexibility**: Allow users to switch between different payment models
5. **Testing**: A/B test different pricing strategies and patterns

These patterns can be combined and customized based on your specific use case and user needs. The key is to align your payment model with how users want to consume your service.