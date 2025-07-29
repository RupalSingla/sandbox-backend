# Usage Examples

This document provides practical examples for using the ONDC server APIs in real-world scenarios.

## Table of Contents

1. [Basic Setup](#basic-setup)
2. [Complete ONDC Transaction Flow](#complete-ondc-transaction-flow)
3. [JSON Mapper Examples](#json-mapper-examples)
4. [Error Handling Examples](#error-handling-examples)
5. [Integration Patterns](#integration-patterns)

---

## Basic Setup

### Environment Configuration
```bash
# .env file
PORT=3000
PRIVATE_KEY=Un205TSOdDXTq8E+N/sJOLJ8xalnzZ1EUP1Wcv23sKx70fOfFd4Q2bzfpzPQ+6XZhZv65SH7Pr6YMk8SuFHpxQ==
BAPID=mobility-staging.ondc.org
UNIQUE_KEY=UK-MOBILITY
GATEWAY_URL=https://staging.gateway.proteantech.in/
callbackUrl=https://your-domain.com
ENABLE_SIGNATURE_VALIDATION=true
LOOKUP_URL=https://preprod.registry.ondc.org/ondc/lookup
```

### Starting the Server
```javascript
// app.js is the entry point
node app.js
// Server will start on the configured PORT
```

---

## Complete ONDC Transaction Flow

### 1. Search for Transportation Services

```javascript
const { v4: uuidv4 } = require('uuid');

const searchRequest = {
  context: {
    action: "search",
    bap_id: "mobility-staging.ondc.org",
    bap_uri: "https://your-callback.com/ondc/",
    transaction_id: uuidv4(),
    message_id: uuidv4(),
    timestamp: new Date().toISOString(),
    domain: "ONDC:TRV11",
    version: "2.0.0",
    ttl: "PT10M",
    location: {
      country: { code: "IND" },
      city: { code: "std:011" }
    }
  },
  message: {
    intent: {
      fulfillment: {
        vehicle: { category: "METRO" },
        stops: [
          {
            location: { gps: "28.7041,77.1025" }, // Delhi coordinates
            type: "START"
          },
          {
            location: { gps: "28.5355,77.3910" }, // Noida coordinates  
            type: "END"
          }
        ]
      }
    }
  }
};

// Send search request
const searchResponse = await fetch('http://localhost:3000/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(searchRequest)
});

const searchResult = await searchResponse.json();
console.log('Search Result:', searchResult);

// Wait for callbacks and check cache
setTimeout(async () => {
  const cacheResponse = await fetch(
    `http://localhost:3000/cache?transactionid=${searchRequest.context.transaction_id}`
  );
  const cachedData = await cacheResponse.json();
  console.log('Cached Search Responses:', cachedData);
}, 5000);
```

### 2. Select a Service

```javascript
// Assuming you got provider and item IDs from search response
const selectRequest = {
  context: {
    ...searchRequest.context,
    action: "select",
    message_id: uuidv4(),
    timestamp: new Date().toISOString(),
    bpp_id: "metro-provider.ondc.org", // From search response
    bpp_uri: "https://metro-provider.ondc.org/api"
  },
  message: {
    order: {
      provider: { id: "P1" }, // From search response
      items: [
        {
          id: "METRO_TICKET", // From search response
          quantity: { selected: { count: 2 } }
        }
      ],
      fulfillments: [
        {
          id: "F1",
          stops: [
            {
              location: { gps: "28.7041,77.1025" },
              type: "START"
            },
            {
              location: { gps: "28.5355,77.3910" },
              type: "END"
            }
          ]
        }
      ]
    }
  }
};

const selectResponse = await fetch('http://localhost:3000/select', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(selectRequest)
});

const selectResult = await selectResponse.json();
console.log('Select Result:', selectResult);
```

### 3. Initialize Booking

```javascript
const initRequest = {
  context: {
    ...selectRequest.context,
    action: "init",
    message_id: uuidv4(),
    timestamp: new Date().toISOString()
  },
  message: {
    order: {
      ...selectRequest.message.order,
      billing: {
        name: "John Doe",
        email: "john@example.com",
        phone: "+91-9876543210",
        address: "123 Main St, Delhi, India"
      },
      payments: [
        {
          type: "PRE-ORDER",
          params: {
            amount: "50.00",
            currency: "INR"
          }
        }
      ]
    }
  }
};

const initResponse = await fetch('http://localhost:3000/init', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(initRequest)
});

const initResult = await initResponse.json();
console.log('Init Result:', initResult);
```

### 4. Confirm Booking

```javascript
const confirmRequest = {
  context: {
    ...initRequest.context,
    action: "confirm",
    message_id: uuidv4(),
    timestamp: new Date().toISOString()
  },
  message: {
    order: {
      ...initRequest.message.order,
      payments: [
        {
          ...initRequest.message.order.payments[0],
          status: "PAID",
          params: {
            ...initRequest.message.order.payments[0].params,
            transaction_id: "txn_" + uuidv4()
          }
        }
      ]
    }
  }
};

const confirmResponse = await fetch('http://localhost:3000/confirm', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(confirmRequest)
});

const confirmResult = await confirmResponse.json();
console.log('Confirm Result:', confirmResult);
```

### 5. Check Order Status

```javascript
const statusRequest = {
  context: {
    ...confirmRequest.context,
    action: "status",
    message_id: uuidv4(),
    timestamp: new Date().toISOString()
  },
  message: {
    order_id: "order_123" // From confirm response
  }
};

const statusResponse = await fetch('http://localhost:3000/status', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(statusRequest)
});

const statusResult = await statusResponse.json();
console.log('Status Result:', statusResult);
```

---

## JSON Mapper Examples

### 1. Create a Mapping Session

```javascript
const sessionRequest = {
  bap_id: "mobility-staging.ondc.org",
  bap_uri: "https://your-callback.com/ondc/",
  domain: "ONDC:TRV11",
  ttl: "PT10M",
  version: "2.0.0",
  country: "IND",
  cityCode: "std:011",
  transaction_id: uuidv4(),
  configName: "metro-flow-1" // Must match a config in configs/ directory
};

const sessionResponse = await fetch('http://localhost:3000/mapper/session', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(sessionRequest)
});

const session = await sessionResponse.json();
console.log('Created Session:', session);
```

### 2. Execute Search with Business Payload

```javascript
const businessSearchPayload = {
  transactionId: session.data.transaction_id,
  payload: {
    // Business-friendly fields instead of complex BECKN structure
    sourceStation: "Rajiv Chowk",
    destinationStation: "Noida City Centre",
    vehicleType: "METRO",
    passengerCount: 2,
    travelDate: "2024-01-15",
    travelTime: "09:00"
  }
};

const mappedSearchResponse = await fetch('http://localhost:3000/mapper/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(businessSearchPayload)
});

const mappedSearchResult = await mappedSearchResponse.json();
console.log('Mapped Search Result:', mappedSearchResult);
```

### 3. Execute Select with Business Payload

```javascript
const businessSelectPayload = {
  transactionId: session.data.transaction_id,
  payload: {
    selectedProviderId: "P1",
    selectedItemId: "METRO_TICKET_BLUE_LINE",
    ticketCount: 2,
    fare: "50.00"
  }
};

const mappedSelectResponse = await fetch('http://localhost:3000/mapper/select', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(businessSelectPayload)
});

const mappedSelectResult = await mappedSelectResponse.json();
console.log('Mapped Select Result:', mappedSelectResult);
```

### 4. Handle Timeout Scenario

```javascript
const timeoutRequest = {
  config: "select", // The config that timed out
  transactionId: session.data.transaction_id
};

const timeoutResponse = await fetch('http://localhost:3000/mapper/timeout', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(timeoutRequest)
});

const timeoutResult = await timeoutResponse.json();
console.log('Timeout Handled:', timeoutResult);
```

### 5. Extract Data from Complex Objects

```javascript
const extractRequest = {
  path: "message.catalog.providers[0].items{id: .id, name: .descriptor.name, price: .price.value}",
  obj: {
    message: {
      catalog: {
        providers: [
          {
            id: "P1",
            items: [
              {
                id: "I1",
                descriptor: { name: "Metro Ticket" },
                price: { value: "25.00", currency: "INR" }
              }
            ]
          }
        ]
      }
    }
  }
};

const extractResponse = await fetch('http://localhost:3000/mapper/extractPath', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(extractRequest)
});

const extractResult = await extractResponse.json();
console.log('Extracted Data:', extractResult);
```

---

## Error Handling Examples

### 1. Handling Validation Errors

```javascript
const invalidRequest = {
  context: {
    action: "search"
    // Missing required fields: bap_uri, transaction_id
  },
  message: {}
};

try {
  const response = await fetch('http://localhost:3000/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(invalidRequest)
  });

  if (!response.ok) {
    const error = await response.json();
    console.error('Validation Error:', error);
    // Expected: 400 status with validation message
  }
} catch (error) {
  console.error('Network Error:', error);
}
```

### 2. Handling Missing Session Errors

```javascript
const invalidSessionRequest = {
  transactionId: "non-existent-transaction-id",
  payload: { someData: "value" }
};

try {
  const response = await fetch('http://localhost:3000/mapper/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(invalidSessionRequest)
  });

  if (!response.ok) {
    const error = await response.json();
    console.error('Session Error:', error);
    // Expected: 400 status with "No session exists" message
  }
} catch (error) {
  console.error('Request Error:', error);
}
```

### 3. Handling Network Timeouts

```javascript
const timeoutController = new AbortController();
const timeoutId = setTimeout(() => timeoutController.abort(), 5000); // 5 second timeout

try {
  const response = await fetch('http://localhost:3000/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(searchRequest),
    signal: timeoutController.signal
  });

  clearTimeout(timeoutId);
  
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const result = await response.json();
  console.log('Success:', result);
} catch (error) {
  clearTimeout(timeoutId);
  
  if (error.name === 'AbortError') {
    console.error('Request timed out');
  } else {
    console.error('Request failed:', error.message);
  }
}
```

---

## Integration Patterns

### 1. React Frontend Integration

```javascript
// React component for ONDC search
import React, { useState } from 'react';

function TransportSearch() {
  const [searchResults, setSearchResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [transactionId, setTransactionId] = useState(null);

  const handleSearch = async (fromLocation, toLocation) => {
    setLoading(true);
    const txnId = uuidv4();
    setTransactionId(txnId);

    try {
      const searchRequest = {
        context: {
          action: "search",
          bap_id: "your-bap-id",
          bap_uri: "https://your-callback.com/ondc/",
          transaction_id: txnId,
          message_id: uuidv4(),
          timestamp: new Date().toISOString(),
          domain: "ONDC:TRV11",
          version: "2.0.0",
          ttl: "PT10M"
        },
        message: {
          intent: {
            fulfillment: {
              vehicle: { category: "METRO" },
              stops: [
                { location: { gps: fromLocation }, type: "START" },
                { location: { gps: toLocation }, type: "END" }
              ]
            }
          }
        }
      };

      await fetch('/api/search', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(searchRequest)
      });

      // Poll for results
      setTimeout(() => pollForResults(txnId), 2000);
    } catch (error) {
      console.error('Search failed:', error);
      setLoading(false);
    }
  };

  const pollForResults = async (txnId) => {
    try {
      const response = await fetch(`/api/cache?transactionid=${txnId}`);
      const cachedData = await response.json();
      
      const searchResults = cachedData.filter(
        item => item.action === 'on_search'
      );

      if (searchResults.length > 0) {
        setSearchResults(searchResults);
        setLoading(false);
      } else {
        // Continue polling
        setTimeout(() => pollForResults(txnId), 2000);
      }
    } catch (error) {
      console.error('Failed to get results:', error);
      setLoading(false);
    }
  };

  return (
    <div>
      {loading && <p>Searching for transportation options...</p>}
      {searchResults.map((result, index) => (
        <div key={index}>
          {/* Render search results */}
        </div>
      ))}
    </div>
  );
}
```

### 2. Express.js Middleware Integration

```javascript
// middleware/ondc-auth.js
const { generateHeader } = require('../utils/utils');

const ondcAuthMiddleware = async (req, res, next) => {
  try {
    // Add ONDC authentication header to outbound requests
    if (req.body && req.body.context) {
      const authHeader = await generateHeader(req.body);
      req.headers['authorization'] = authHeader;
    }
    next();
  } catch (error) {
    res.status(500).json({ error: 'Failed to generate ONDC auth header' });
  }
};

// routes/transport.js
const express = require('express');
const router = express.Router();
const ondcAuthMiddleware = require('../middleware/ondc-auth');

router.use(ondcAuthMiddleware);

router.post('/search', async (req, res) => {
  // Your business logic here
  // ONDC headers are automatically added by middleware
});

module.exports = router;
```

### 3. Queue-based Processing

```javascript
// Using Bull Queue for async processing
const Queue = require('bull');
const ondcQueue = new Queue('ONDC processing');

// Producer: Add ONDC requests to queue
const queueOndcRequest = async (action, payload) => {
  await ondcQueue.add('process-ondc-request', {
    action,
    payload,
    timestamp: new Date().toISOString()
  });
};

// Consumer: Process ONDC requests
ondcQueue.process('process-ondc-request', async (job) => {
  const { action, payload } = job.data;
  
  try {
    const response = await fetch(`http://localhost:3000/${action}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    const result = await response.json();
    
    // Store result or trigger next step
    await handleOndcResponse(action, result);
    
    return result;
  } catch (error) {
    console.error(`ONDC ${action} failed:`, error);
    throw error;
  }
});

// Usage
await queueOndcRequest('search', searchPayload);
```

### 4. WebSocket Integration for Real-time Updates

```javascript
// WebSocket server for real-time ONDC updates
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// Store client connections by transaction ID
const clientConnections = new Map();

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    const data = JSON.parse(message);
    
    if (data.type === 'subscribe' && data.transactionId) {
      clientConnections.set(data.transactionId, ws);
    }
  });
});

// Callback handler integration
const handleOndcCallback = async (req, res) => {
  const { context, message } = req.body;
  
  // Process callback normally
  const ack = { message: { ack: { status: "ACK" } } };
  res.json(ack);
  
  // Send real-time update to subscribed client
  const client = clientConnections.get(context.transaction_id);
  if (client && client.readyState === WebSocket.OPEN) {
    client.send(JSON.stringify({
      type: 'ondc-callback',
      action: context.action,
      data: { context, message }
    }));
  }
};
```

### 5. Caching Strategy with Redis

```javascript
// Redis integration for production caching
const redis = require('redis');
const client = redis.createClient();

// Enhanced cache utilities with Redis
const enhancedCache = {
  async set(key, value, ttl = 600) {
    await client.setex(key, ttl, JSON.stringify(value));
  },

  async get(key) {
    const value = await client.get(key);
    return value ? JSON.parse(value) : null;
  },

  async append(key, newValue, ttl = 600) {
    const existing = await this.get(key) || [];
    existing.push(newValue);
    await this.set(key, existing, ttl);
    return existing.length;
  },

  async keys(pattern = '*') {
    return await client.keys(pattern);
  }
};

// Usage in ONDC functions
const insertRequestEnhanced = async (request, header) => {
  if (!request?.context?.transaction_id) return;
  
  const order = await enhancedCache.append(
    request.context.transaction_id,
    {
      action: request.context.action,
      order: Date.now(), // Use timestamp for ordering
      header: header || null,
      timestamp: new Date(),
      data: request
    }
  );
  
  return order;
};
```

This examples document provides practical, real-world usage scenarios that developers can use as a starting point for integrating with the ONDC server.