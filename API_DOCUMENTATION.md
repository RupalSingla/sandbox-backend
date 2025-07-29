# ONDC Server API Documentation

## Project Overview

This is an ONDC (Open Network for Digital Commerce) server implementation that provides APIs for transportation and mobility services using the BECKN protocol. The server handles ONDC protocol compliance, digital signature verification, payload transformation, and provides both direct protocol APIs and a JSON mapper system for easier integration.

## Table of Contents

1. [Installation & Setup](#installation--setup)
2. [API Endpoints](#api-endpoints)
3. [Utility Functions](#utility-functions)
4. [Configuration System](#configuration-system)
5. [Authentication & Security](#authentication--security)
6. [Usage Examples](#usage-examples)

---

## Installation & Setup

### Prerequisites
- Node.js (v14 or higher)
- npm or yarn package manager

### Dependencies
```json
{
  "@apidevtools/json-schema-ref-parser": "^11.1.0",
  "axios": "^1.5.0",
  "cors": "^2.8.5",
  "dotenv": "^16.3.1",
  "express": "^4.18.2",
  "js-yaml": "^4.1.0",
  "node-cache": "^5.1.2",
  "ondc-crypto-sdk-nodejs": "^1.0.2",
  "uuid": "^9.0.1",
  "yaml": "^2.3.4"
}
```

### Environment Variables
Create a `.env` file with the following variables:

```env
PORT=3000
PRIVATE_KEY=your_signing_private_key
BAPID=your_bap_id
UNIQUE_KEY=your_unique_key_id
mockUrl=your_mock_url
callbackUrl=your_callback_url
GATEWAY_URL=https://staging.gateway.proteantech.in/
ENABLE_SIGNATURE_VALIDATION=true
LOOKUP_URL=https://preprod.registry.ondc.org/ondc/lookup
```

### Installation
```bash
npm install
npm start
```

---

## API Endpoints

### 1. Create Authorization Header

**Endpoint:** `POST /createHeader`

**Description:** Generates ONDC compliant authorization headers for API requests.

**Request Body:**
```json
{
  "context": {
    "action": "search",
    "bap_id": "example-bap.com",
    "bap_uri": "https://example-bap.com",
    "transaction_id": "unique-transaction-id",
    "message_id": "unique-message-id",
    "timestamp": "2024-01-12T04:21:51.682Z",
    "domain": "ONDC:TRV11",
    "version": "2.0.0",
    "ttl": "PT10M"
  },
  "message": {
    // Your message payload
  }
}
```

**Response Headers:**
- `Authorization`: Generated ONDC authorization header
- `Access-Control-Expose-Headers`: "*"

**Response Body:** Returns the original request body

**Usage Example:**
```javascript
const response = await fetch('/createHeader', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});
const authHeader = response.headers.get('Authorization');
```

---

### 2. ONDC Protocol Endpoints

**Endpoint:** `POST /:method`

**Description:** Handles all ONDC protocol methods (search, select, init, confirm, status, update, cancel, etc.)

**Parameters:**
- `method` (path parameter): ONDC action (search, select, init, confirm, etc.)

**Request Body:**
```json
{
  "context": {
    "bap_uri": "https://your-callback-url",
    "transaction_id": "unique-transaction-id", 
    "bpp_uri": "https://provider-url" // Required for non-search methods
  },
  "message": {
    // Action-specific message payload
  }
}
```

**Response:** Returns the BPP response for the action

**Validation Rules:**
- `bap_uri` is required for all methods
- `transaction_id` is required for all methods  
- `bpp_uri` is required for all methods except 'search'

**Usage Example:**
```javascript
// Search request
const searchResponse = await fetch('/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    context: {
      bap_uri: "https://your-callback.com/ondc/",
      transaction_id: "uuid-v4",
      action: "search",
      domain: "ONDC:TRV11"
    },
    message: {
      intent: {
        fulfillment: { vehicle: { category: "METRO" } }
      }
    }
  })
});
```

---

### 3. Callback Handler

**Endpoint:** `POST /ondc/:method`

**Description:** Handles ONDC callback responses from BPPs (Business Process Providers)

**Parameters:**
- `method` (path parameter): ONDC callback action (on_search, on_select, etc.)

**Features:**
- Optional digital signature verification
- Automatic ACK/NACK responses
- Request logging and caching
- JSON mapper integration

**Response:**
```json
{
  "message": {
    "ack": {
      "status": "ACK" // or "NACK" if validation fails
    }
  }
}
```

---

### 4. Cache Management

#### Get Cache Data
**Endpoint:** `GET /cache?transactionid=<transaction_id>`

**Description:** Retrieves cached transaction data

**Query Parameters:**
- `transactionid`: Transaction ID to retrieve data for

**Response:** Returns all cached requests for the transaction ID

#### Delete Cache Data  
**Endpoint:** `DELETE /cache?transactionid=<transaction_id>`

**Description:** Clears cached data for a transaction

**Query Parameters:**
- `transactionid`: Transaction ID to clear cache for

**Response:**
```json
{
  "message": "Response data for the transaction ID has been deleted"
}
```

---

### 5. JSON Mapper APIs

#### Create Session
**Endpoint:** `POST /mapper/session`

**Description:** Creates a new mapping session for simplified API interactions

**Request Body:**
```json
{
  "bap_id": "your-bap-id",
  "bap_uri": "https://your-bap-uri", 
  "domain": "ONDC:TRV11",
  "ttl": "PT10M",
  "version": "2.0.0",
  "country": "IND",
  "cityCode": "std:011", 
  "transaction_id": "unique-transaction-id",
  "configName": "metro-flow-1"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    // Session object with protocol calls configuration
  }
}
```

#### Execute Protocol Call
**Endpoint:** `POST /mapper/:config`

**Description:** Executes a specific protocol call using business-friendly payload

**Parameters:**
- `config`: Protocol call configuration name (e.g., "search", "select")

**Request Body:**
```json
{
  "transactionId": "session-transaction-id",
  "payload": {
    // Business-specific payload that gets transformed to BECKN format
  }
}
```

#### Handle Timeout
**Endpoint:** `POST /mapper/timeout`

**Description:** Handles timeout scenarios in protocol flows

**Request Body:**
```json
{
  "config": "protocol-call-name",
  "transactionId": "session-transaction-id"
}
```

#### Extract Path
**Endpoint:** `POST /mapper/extractPath`

**Description:** Extracts data from complex nested objects using path expressions

**Request Body:**
```json
{
  "path": "message.catalog.providers[0].items",
  "obj": {
    // Object to extract data from
  }
}
```

---

## Utility Functions

### Cache Utilities (`utils/utils.js`)

#### `insertRequest(request, header)`
**Description:** Stores API requests in cache with metadata

**Parameters:**
- `request`: Request object containing context and message
- `header`: HTTP headers from the request

**Returns:** Order number for the request

**Usage:**
```javascript
const { insertRequest } = require('./utils/utils');
const order = insertRequest(requestBody, req.headers);
```

#### `getCache(key)`
**Description:** Retrieves data from cache

**Parameters:**
- `key`: Cache key (transaction ID) or undefined to get all keys

**Returns:** Cached data or array of cache keys

**Usage:**
```javascript
const { getCache } = require('./utils/utils');
const data = getCache('transaction-id-123');
const allKeys = getCache(); // Get all cache keys
```

#### `generateHeader(message)`
**Description:** Generates ONDC authorization header for a message

**Parameters:**
- `message`: BECKN protocol message object

**Returns:** Promise resolving to authorization header string

**Usage:**
```javascript
const { generateHeader } = require('./utils/utils');
const header = await generateHeader(becknMessage);
```

#### `deleteCache(transactionId)`
**Description:** Clears cache for a transaction

**Parameters:**
- `transactionId`: Transaction ID to clear

**Returns:** Boolean indicating success

#### `verifyHeader(req, lookupUrl)`
**Description:** Verifies digital signature of incoming requests

**Parameters:**
- `req`: Express request object
- `lookupUrl`: ONDC registry lookup URL

**Returns:** Promise resolving to boolean (true if valid)

#### `insertSession(session)`
**Description:** Stores JSON mapper session data

**Parameters:**
- `session`: Session object with mapping configuration

#### `handleRequestForJsonMapper(response, unsolicited)`
**Description:** Processes responses for JSON mapper system

**Parameters:**
- `response`: BECKN protocol response
- `unsolicited`: Boolean indicating if response was unsolicited

---

### Payload Building Utilities (`utils/buildPayload.js`)

#### `createBecknObject(session, call, data)`
**Description:** Transforms business payload to BECKN protocol format

**Parameters:**
- `session`: Mapping session object
- `call`: Protocol call configuration  
- `data`: Business payload data

**Returns:** Object with `payload` (BECKN format) and updated `session`

**Usage:**
```javascript
const { createBecknObject } = require('./utils/buildPayload');
const { payload, session } = createBecknObject(session, protocolCall, businessData);
```

#### `extractBusinessData(type, payload, session)`
**Description:** Extracts business-friendly data from BECKN protocol response

**Parameters:**
- `type`: Protocol call type
- `payload`: BECKN protocol payload
- `session`: Mapping session

**Returns:** Business-friendly data object

#### `extractPath(path, obj)`
**Description:** Extracts data from nested objects using path expressions

**Parameters:**
- `path`: Dot-notation path with special syntax for arrays/objects
- `obj`: Source object

**Returns:** Extracted data

**Path Syntax Examples:**
```javascript
// Simple path
"message.catalog.providers"

// Array with condition
"message.catalog.providers[?(_?.id === 'P1')]"

// Object transformation
"message.catalog.providers[0].items{id: .id, name: .descriptor.name}"
```

---

## Configuration System

### ConfigLoader Class (`configs/index.js`)

The configuration system manages YAML-based configurations with JSON schema validation.

#### `configLoader.init()`
**Description:** Initializes and loads configuration from YAML files

**Returns:** Promise resolving to parsed configuration object

**Usage:**
```javascript
const { configLoader } = require('./configs/index');
await configLoader.init();
const config = configLoader.getConfig();
```

#### Configuration Structure
Configurations are stored in the `configs/` directory:
- `index.yaml`: Main configuration file
- `metro-flow-1/`: Metro transportation flow configuration
- `metro-flow-2/`: Alternative metro flow
- `ondemand-flow-1/`: On-demand transportation flow

Each flow directory contains:
- `protocolCalls.yaml`: Protocol call definitions
- `input.yaml`: Input field definitions  
- `protocolMapping.yaml`: Business-to-BECKN mapping rules

---

## Authentication & Security

### Digital Signature Creation (`header.js`)

#### `createHeader(message)`
**Description:** Creates ONDC compliant authorization header with digital signature

**Parameters:**
- `message`: BECKN protocol message to sign

**Returns:** Promise resolving to authorization header string

**Usage:**
```javascript
const { createHeader } = require('./header');
const authHeader = await createHeader(becknMessage);
```

### Security Features

1. **Digital Signature Verification**: All incoming requests can be verified using ONDC crypto SDK
2. **Request Validation**: Validates required fields in BECKN protocol messages
3. **Environment-based Configuration**: Sensitive keys stored in environment variables
4. **CORS Support**: Configurable cross-origin resource sharing

---

## Usage Examples

### Basic ONDC Flow

```javascript
// 1. Create search request
const searchPayload = {
  context: {
    action: "search",
    bap_id: "your-bap-id",
    bap_uri: "https://your-callback.com/ondc/",
    transaction_id: "uuid-v4",
    message_id: "uuid-v4", 
    timestamp: new Date().toISOString(),
    domain: "ONDC:TRV11",
    version: "2.0.0",
    ttl: "PT10M"
  },
  message: {
    intent: {
      fulfillment: { 
        vehicle: { category: "METRO" } 
      }
    }
  }
};

// 2. Send search request
const searchResponse = await fetch('/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(searchPayload)
});

// 3. Get cached responses
const cachedData = await fetch(`/cache?transactionid=${searchPayload.context.transaction_id}`);
```

### JSON Mapper Flow

```javascript
// 1. Create session
const sessionResponse = await fetch('/mapper/session', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    bap_id: "your-bap-id",
    bap_uri: "https://your-bap-uri",
    domain: "ONDC:TRV11", 
    ttl: "PT10M",
    version: "2.0.0",
    country: "IND",
    cityCode: "std:011",
    transaction_id: "uuid-v4",
    configName: "metro-flow-1"
  })
});

const session = await sessionResponse.json();

// 2. Execute search with business payload
const searchResponse = await fetch('/mapper/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    transactionId: session.data.transaction_id,
    payload: {
      sourceStation: "Station A",
      destinationStation: "Station B", 
      vehicleType: "METRO"
    }
  })
});
```

### Header Generation

```javascript
// Generate authorization header
const response = await fetch('/createHeader', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(becknMessage)
});

const authHeader = response.headers.get('Authorization');

// Use header in subsequent requests
const apiResponse = await fetch('/external-api', {
  method: 'POST',
  headers: { 
    'Authorization': authHeader,
    'Content-Type': 'application/json' 
  },
  body: JSON.stringify(becknMessage)
});
```

### Error Handling

```javascript
try {
  const response = await fetch('/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  });
  
  if (!response.ok) {
    const error = await response.json();
    console.error('API Error:', error);
  }
  
  const data = await response.json();
  console.log('Success:', data);
} catch (error) {
  console.error('Network Error:', error);
}
```

---

## Response Formats

### Successful Response
```json
{
  "context": {
    "action": "on_search",
    "bap_id": "your-bap-id",
    "transaction_id": "transaction-uuid",
    "message_id": "message-uuid",
    "timestamp": "2024-01-12T04:21:51.682Z"
  },
  "message": {
    // Action-specific response data
  }
}
```

### Error Response
```json
{
  "data": "Error message describing what went wrong",
  "error": true
}
```

### ACK Response (Callbacks)
```json
{
  "message": {
    "ack": {
      "status": "ACK"
    }
  }
}
```

---

## Best Practices

1. **Transaction Management**: Always use unique UUIDs for transaction_id and message_id
2. **Error Handling**: Implement proper error handling for all API calls
3. **Caching**: Use the cache endpoints to retrieve historical transaction data
4. **Security**: Enable signature validation in production environments
5. **Configuration**: Use appropriate config flows for different transportation types
6. **Timeouts**: Handle timeout scenarios properly using the timeout endpoint

---

## Configuration Examples

### Metro Flow Configuration
Located in `configs/metro-flow-1/`:

```yaml
# protocolCalls.yaml
search:
  type: "search"
  config: "search"
  shouldRender: true
  nextRequest: "on_search"
  
select:
  type: "select" 
  config: "select"
  shouldRender: false
  preRequest: "search"
  nextRequest: "on_select"
```

### Environment Configuration
```env
# Required for ONDC compliance
PRIVATE_KEY=your_base64_encoded_private_key
BAPID=your_registered_bap_id
UNIQUE_KEY=your_unique_key_identifier

# URLs
GATEWAY_URL=https://staging.gateway.proteantech.in/
callbackUrl=https://your-domain.com
LOOKUP_URL=https://preprod.registry.ondc.org/ondc/lookup

# Features
ENABLE_SIGNATURE_VALIDATION=true
```

This documentation covers all public APIs, utility functions, and components in the ONDC server. For additional technical details, refer to the inline code comments and configuration files.