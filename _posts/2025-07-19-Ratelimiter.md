# Complete Guide to Rate Limiting with Upstash Redis and Next.js

Rate limiting is a crucial technique for protecting your APIs from abuse, ensuring fair usage, and maintaining service quality. In this comprehensive guide, we'll explore how to implement robust rate limiting using Upstash Redis and their dedicated rate limiting library.

## Why Rate Limiting Matters

Before diving into implementation, let's understand why rate limiting is essential:

- **Prevent API Abuse**: Stop malicious users from overwhelming your services
- **Ensure Fair Usage**: Guarantee all users get equitable access to resources
- **Cost Control**: Manage server resources and prevent unexpected scaling costs
- **Service Quality**: Maintain consistent performance for all users
- **Business Logic**: Implement tiered pricing models with usage limits

## Getting Started

### Installation

First, install the required packages:

```bash
npm install @upstash/ratelimit @upstash/redis
```

### Environment Setup

Create your environment variables:

```env
UPSTASH_REDIS_REST_URL=your_redis_url
UPSTASH_REDIS_REST_TOKEN=your_redis_token
```

You can get these credentials from your [Upstash Console](https://console.upstash.com) after creating a Redis database.

## Basic Implementation

Let's start with a simple rate limiting setup:

```javascript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

// Initialize Redis client
const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

// Create rate limiter instance
const ratelimit = new Ratelimit({
  redis: redis,
  limiter: Ratelimit.slidingWindow(10, "10 s"), // 10 requests per 10 seconds
});

// Example usage in an API route
export async function handleRequest(request) {
  // Use IP address or user ID as identifier
  const identifier = getClientIP(request) || "anonymous";

  const { success, limit, reset, remaining } = await ratelimit.limit(
    identifier
  );

  if (!success) {
    return new Response("Too Many Requests", {
      status: 429,
      headers: {
        "X-RateLimit-Limit": limit.toString(),
        "X-RateLimit-Remaining": remaining.toString(),
        "X-RateLimit-Reset": new Date(reset).toISOString(),
      },
    });
  }

  // Process the request normally
  return new Response("Request processed successfully", {
    headers: {
      "X-RateLimit-Limit": limit.toString(),
      "X-RateLimit-Remaining": remaining.toString(),
      "X-RateLimit-Reset": new Date(reset).toISOString(),
    },
  });
}

function getClientIP(request) {
  return (
    request.headers.get("x-forwarded-for") ||
    request.headers.get("x-real-ip") ||
    "unknown"
  );
}
```

This basic setup provides sliding window rate limiting with 10 requests allowed per 10-second window.

## Understanding Rate Limiting Algorithms

Upstash provides several rate limiting algorithms, each with different characteristics:

### 1. Fixed Window

```javascript
const fixedWindow = new Ratelimit({
  redis,
  limiter: Ratelimit.fixedWindow(5, "60 s"), // 5 requests per minute
});
```

**Pros**: Simple and memory efficient
**Cons**: Can allow bursts at window boundaries (up to 2x the limit)

### 2. Sliding Window

```javascript
const slidingWindow = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, "60 s"), // 10 requests per minute
});
```

**Pros**: More accurate, prevents burst issues
**Cons**: Slightly more complex and uses more memory

### 3. Token Bucket

```javascript
const tokenBucket = new Ratelimit({
  redis,
  limiter: Ratelimit.tokenBucket(5, "10 s", 10), // 5 tokens per 10s, max 10 tokens
});
```

**Pros**: Allows controlled bursts up to bucket capacity
**Cons**: More complex to reason about

### 4. Multi-Region (Tiered Limiting)

```javascript
const tieredLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.multiRegion([
    Ratelimit.slidingWindow(10, "10 s"), // 10 per 10 seconds
    Ratelimit.slidingWindow(50, "60 s"), // 50 per minute
    Ratelimit.slidingWindow(1000, "1 h"), // 1000 per hour
  ]),
});
```

**Pros**: Multiple layers of protection
**Cons**: More complex and resource-intensive

## Next.js Integration

### App Router Implementation

For Next.js 13+ with the App Router:

```javascript
// app/api/protected-endpoint/route.js
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";
import { NextResponse } from "next/server";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

// Create different rate limiters for different user types
const rateLimiters = {
  free: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(10, "1 h"),
    prefix: "ratelimit:free",
  }),
  premium: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(100, "1 h"),
    prefix: "ratelimit:premium",
  }),
  anonymous: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(5, "1 h"),
    prefix: "ratelimit:anon",
  }),
};

export async function POST(request) {
  const identifier = getIdentifier(request);
  const userTier = getUserTier(request);

  const ratelimit = rateLimiters[userTier] || rateLimiters.anonymous;

  const { success, limit, reset, remaining } = await ratelimit.limit(
    identifier
  );

  if (!success) {
    return NextResponse.json(
      {
        error: "Too many requests",
        retryAfter: Math.round((reset - Date.now()) / 1000),
      },
      {
        status: 429,
        headers: {
          "X-RateLimit-Limit": limit.toString(),
          "X-RateLimit-Remaining": remaining.toString(),
          "X-RateLimit-Reset": reset.toString(),
          "Retry-After": Math.round((reset - Date.now()) / 1000).toString(),
        },
      }
    );
  }

  // Your API logic here
  const result = await processRequest(request);

  return NextResponse.json(result, {
    headers: {
      "X-RateLimit-Limit": limit.toString(),
      "X-RateLimit-Remaining": remaining.toString(),
      "X-RateLimit-Reset": reset.toString(),
    },
  });
}
```

### Pages Router Implementation

For Next.js 12 and below:

```javascript
// pages/api/protected-endpoint.js
export default async function handler(req, res) {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const identifier = getIdentifier(req);
  const userTier = getUserTier(req);

  const ratelimit = rateLimiters[userTier] || rateLimiters.anonymous;

  const { success, limit, reset, remaining } = await ratelimit.limit(
    identifier
  );

  // Set rate limit headers
  res.setHeader("X-RateLimit-Limit", limit);
  res.setHeader("X-RateLimit-Remaining", remaining);
  res.setHeader("X-RateLimit-Reset", reset);

  if (!success) {
    return res.status(429).json({
      error: "Too many requests",
      retryAfter: Math.round((reset - Date.now()) / 1000),
    });
  }

  // Your API logic here
  const result = await processRequest(req);
  res.status(200).json(result);
}
```

## Advanced Patterns

### Middleware-Based Rate Limiting

Implement global rate limiting using Next.js middleware:

```javascript
// middleware.js
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";
import { NextResponse } from "next/server";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

const globalLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(50, "1 m"),
  prefix: "global",
});

export async function middleware(request) {
  const { pathname } = request.nextUrl;

  // Skip rate limiting for static files
  if (
    pathname.startsWith("/_next") ||
    pathname.startsWith("/static") ||
    pathname.includes(".") ||
    pathname === "/health"
  ) {
    return NextResponse.next();
  }

  const ip = request.ip ?? request.headers.get("x-forwarded-for") ?? "unknown";
  const { success, limit, reset, remaining } = await globalLimiter.limit(
    `global:${ip}`
  );

  const response = success
    ? NextResponse.next()
    : NextResponse.json({ error: "Too Many Requests" }, { status: 429 });

  // Add rate limit headers
  response.headers.set("X-RateLimit-Limit", limit.toString());
  response.headers.set("X-RateLimit-Remaining", remaining.toString());
  response.headers.set("X-RateLimit-Reset", reset.toString());

  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### Rate Limit Manager Class

Create a reusable rate limiting utility:

```javascript
// utils/rateLimit.js
export class RateLimitManager {
  constructor(redisConfig) {
    this.redis = new Redis(redisConfig);
    this.limiters = new Map();
  }

  createLimiter(name, config) {
    const limiter = new Ratelimit({
      redis: this.redis,
      limiter: config.algorithm,
      prefix: `ratelimit:${name}`,
      analytics: config.analytics || false,
    });

    this.limiters.set(name, limiter);
    return limiter;
  }

  async checkLimit(limiterName, identifier, options = {}) {
    const limiter = this.limiters.get(limiterName);
    if (!limiter) {
      throw new Error(`Rate limiter '${limiterName}' not found`);
    }

    const result = await limiter.limit(identifier);

    if (options.throwOnLimit && !result.success) {
      const error = new Error("Rate limit exceeded");
      error.status = 429;
      error.rateLimit = result;
      throw error;
    }

    return result;
  }
}

// Usage
const rateLimitManager = new RateLimitManager({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

rateLimitManager.createLimiter("auth", {
  algorithm: Ratelimit.fixedWindow(5, "15 m"),
  analytics: true,
});
```

### Higher-Order Function for Route Protection

Create a decorator for protecting API routes:

```javascript
export function withRateLimit(limiterName, getIdentifier) {
  return function (handler) {
    return async function (req, res) {
      try {
        const identifier = await getIdentifier(req);
        const result = await rateLimitManager.checkLimit(
          limiterName,
          identifier
        );

        // Add headers
        res.setHeader("X-RateLimit-Limit", result.limit);
        res.setHeader("X-RateLimit-Remaining", result.remaining);
        res.setHeader("X-RateLimit-Reset", result.reset);

        if (!result.success) {
          return res.status(429).json({
            error: "Too many requests",
            retryAfter: Math.round((result.reset - Date.now()) / 1000),
          });
        }

        return await handler(req, res);
      } catch (error) {
        // Handle rate limit errors
        if (error.status === 429) {
          return res.status(429).json({
            error: "Rate limit exceeded",
            retryAfter: Math.round((error.rateLimit.reset - Date.now()) / 1000),
          });
        }
        throw error;
      }
    };
  };
}

// Usage
export default withRateLimit(
  "api",
  (req) => req.headers["x-user-id"] || req.ip
)(async function handler(req, res) {
  res.json({ message: "Success" });
});
```

## Best Practices

### 1. Choose the Right Identifier

Different scenarios require different identifiers:

- **IP Address**: Good for anonymous users, but problematic for shared networks
- **User ID**: Best for authenticated users
- **API Key**: Ideal for B2B APIs
- **Session ID**: Useful for temporary rate limiting

### 2. Implement Tiered Limiting

Create different limits based on user types:

```javascript
const getLimiterForUser = (userType) => {
  switch (userType) {
    case "premium":
      return new Ratelimit({
        redis,
        limiter: Ratelimit.slidingWindow(1000, "1 h"),
      });
    case "free":
      return new Ratelimit({
        redis,
        limiter: Ratelimit.slidingWindow(100, "1 h"),
      });
    default:
      return new Ratelimit({
        redis,
        limiter: Ratelimit.slidingWindow(10, "1 h"),
      });
  }
};
```

### 3. Always Include Headers

Provide clients with rate limit information:

```javascript
const headers = {
  "X-RateLimit-Limit": limit.toString(),
  "X-RateLimit-Remaining": remaining.toString(),
  "X-RateLimit-Reset": reset.toString(),
  "Retry-After": Math.round((reset - Date.now()) / 1000).toString(),
};
```

### 4. Use Prefixes for Organization

Separate different types of rate limits:

```javascript
const authLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.fixedWindow(5, "15 m"),
  prefix: "auth",
});

const apiLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, "1 h"),
  prefix: "api",
});
```

### 5. Handle Edge Cases

Consider various scenarios:

- **Network Issues**: Implement fallbacks for Redis connectivity problems
- **Shared IPs**: Use additional identifiers for users behind NAT
- **Bot Traffic**: Implement more aggressive limits for suspected bots
- **Burst Traffic**: Use token bucket for legitimate burst scenarios

## Testing Your Rate Limits

### Unit Testing

```javascript
// __tests__/ratelimit.test.js
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

describe("Rate Limiting", () => {
  let ratelimit;

  beforeEach(() => {
    const redis = new Redis({
      url: process.env.UPSTASH_REDIS_REST_URL,
      token: process.env.UPSTASH_REDIS_REST_TOKEN,
    });

    ratelimit = new Ratelimit({
      redis,
      limiter: Ratelimit.slidingWindow(5, "60 s"),
      prefix: "test",
    });
  });

  it("should allow requests within limit", async () => {
    const result = await ratelimit.limit("test-user");
    expect(result.success).toBe(true);
    expect(result.remaining).toBe(4);
  });

  it("should block requests exceeding limit", async () => {
    // Make 5 requests to reach the limit
    for (let i = 0; i < 5; i++) {
      await ratelimit.limit("test-user-blocked");
    }

    // 6th request should be blocked
    const result = await ratelimit.limit("test-user-blocked");
    expect(result.success).toBe(false);
    expect(result.remaining).toBe(0);
  });
});
```

### Load Testing

Use tools like Artillery or k6 to test your rate limits under load:

```yaml
# artillery-config.yml
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60
      arrivalRate: 10
scenarios:
  - name: "API Rate Limit Test"
    requests:
      - get:
          url: "/api/protected-endpoint"
          headers:
            Authorization: "Bearer test-token"
```

## Monitoring and Analytics

### Enable Analytics

```javascript
const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, "1 h"),
  analytics: true, // Enable built-in analytics
});
```

### Custom Monitoring

```javascript
const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, "1 h"),
  onHit: (identifier, info) => {
    // Log rate limit hits
    console.log(`Rate limit hit for ${identifier}:`, info);
  },
});
```

## Common Pitfalls to Avoid

1. **Not handling Redis failures**: Always have a fallback strategy
2. **Using only IP addresses**: Consider shared networks and mobile users
3. **Setting limits too low**: Start generous and tighten based on usage patterns
4. **Ignoring legitimate bursts**: Use appropriate algorithms for your use case
5. **Not providing clear error messages**: Help users understand what happened

## Conclusion

Rate limiting with Upstash Redis provides a robust, scalable solution for protecting your APIs. The combination of different algorithms, Redis persistence, and serverless compatibility makes it ideal for modern web applications.

Key takeaways:

- Choose the right algorithm for your use case
- Implement tiered limiting based on user types
- Always include informative headers in responses
- Test your limits thoroughly
- Monitor and adjust based on real-world usage

Start with conservative limits and adjust based on your application's behavior and user feedback. Remember, rate limiting should enhance user experience by ensuring service reliability, not frustrate legitimate users with overly restrictive policies.

## Rate limiting in Reviewcraft

ReviewCraft's rate limiting implementation leverages a modern, performant stack:

- **Upstash Redis**: Edge-optimized Redis for ultra-low latency
- **Upstash Ratelimit**: Purpose-built rate limiting library
- **Next.js 15**: Full-stack React framework with App Router
- **TypeScript**: Type-safe development experience

## Implementation Architecture

### 1. Core Rate Limiting Configuration

Our rate limiting system starts with a centralized configuration in `src/lib/upstash/ratelimit.ts`:

```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { redis } from "./redis";

export const ratelimit = new Ratelimit({
  redis: redis,
  limiter: Ratelimit.fixedWindow(10, "10 s"),
  analytics: true,
  prefix: "review-platform:ratelimit",
});
```

**Key Features:**

- **Fixed Window Algorithm**: 10 requests per 10-second window
- **Analytics Enabled**: Tracks usage patterns for monitoring
- **Namespaced Keys**: Prevents conflicts with other applications
- **Redis-Backed**: Distributed rate limiting across instances

### 2. Redis Infrastructure

We use Upstash Redis for its serverless-friendly architecture:

```typescript
import { Redis } from "@upstash/redis";

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL || "",
  token: process.env.UPSTASH_REDIS_REST_TOKEN || "",
});
```

**Benefits of Upstash Redis:**

- **Global Edge Network**: Low latency worldwide
- **Serverless Compatible**: Pay-per-request pricing
- **REST API**: Works seamlessly with Next.js Edge Runtime
- **High Availability**: Built-in redundancy and failover

### 3. Universal Rate Limiting Pattern

We've implemented a consistent rate limiting pattern across all API routes:

```typescript
export async function GET(request: Request) {
  // Skip rate limiting in development
  if (process.env.NODE_ENV !== "development") {
    const ip = request.headers.get("x-forwarded-for") ?? "127.0.0.1";
    const { success, limit, remaining, reset } = await ratelimit.limit(ip);

    if (!success) {
      return new Response("Rate limit exceeded. Please try again later.", {
        status: 429,
        headers: {
          "X-RateLimit-Limit": limit.toString(),
          "X-RateLimit-Remaining": remaining.toString(),
          "X-RateLimit-Reset": reset.toString(),
        },
      });
    }
  }

  // API logic continues...
}
```

## Protected Endpoints

Our rate limiting system protects all critical API endpoints:

### Client Management APIs

- `POST /api/clients` - Create new review clients
- `GET /api/clients` - List client accounts
- `GET /api/clients/[id]` - Get specific client details
- `PUT /api/clients/[id]` - Update client configuration
- `DELETE /api/clients/[id]` - Remove client accounts

### System & Integration APIs

- `GET /api/health` - Health check endpoint
- `GET /api/reviews` - Review data access
- `GET /api/webhooks` - Webhook management

## Additional Resources

- [Upstash Redis Documentation](https://docs.upstash.com/redis)
- [Rate Limiting Library GitHub](https://github.com/upstash/ratelimit)
- [Next.js Middleware Documentation](https://nextjs.org/docs/advanced-features/middleware)
- [HTTP Status Code 429 Specification](https://tools.ietf.org/html/rfc6585#section-4)
