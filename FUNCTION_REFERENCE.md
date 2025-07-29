# Function Reference Guide

This document provides detailed technical documentation for all internal functions, utilities, and components in the ONDC server.

## Table of Contents

1. [Core Utilities (`utils/utils.js`)](#core-utilities)
2. [Payload Transformation (`utils/buildPayload.js`)](#payload-transformation)
3. [Response Handling (`utils/response.js`)](#response-handling)
4. [Configuration Management (`configs/index.js`)](#configuration-management)
5. [Authentication (`header.js`)](#authentication)
6. [Router Functions (`router/route.js`)](#router-functions)

---

## Core Utilities (`utils/utils.js`)

### Cache Management Functions

#### `insertRequest(request, header)`
**Purpose:** Stores API requests in the node-cache with metadata for debugging and transaction history.

**Implementation Details:**
- Uses node-cache with TTL of 600 seconds (10 minutes)
- Stores requests as arrays with incremental order numbers
- Automatically adds timestamp and action metadata

**Parameters:**
- `request` (Object): BECKN protocol request with context and message
- `header` (Object): HTTP headers from the Express request

**Returns:** `number` - Order sequence number for the request

**Internal Logic:**
```javascript
// Validates transaction_id exists
if(!request?.context?.transaction_id) return

// Retrieves existing cache or creates new array
let response = myCache.get(request.context.transaction_id)||[]

// Calculates next order number
const order = response.length >= 1 ? response[response.length-1].order+1 : 1

// Stores with metadata
myCache.set(request.context.transaction_id, [...response, {
  action: request.context.action,
  order: order,
  header: header ? header : null,
  timestamp: new Date(),
  data: request
}], 1000 * 60 * 10)
```

#### `getCache(key)`
**Purpose:** Retrieves cached data with flexible key handling.

**Implementation Details:**
- Returns all cache keys when no key provided
- Returns specific transaction data when key provided
- Uses node-cache's native get() method

**Parameters:**
- `key` (string|undefined): Transaction ID or undefined for all keys

**Returns:** 
- `Array` - Cache keys when key is undefined
- `Array|undefined` - Transaction data array or undefined

#### `deleteCache(transactionId)`
**Purpose:** Clears cache data for a specific transaction.

**Implementation Details:**
- Sets cache value to empty array with short TTL
- Effectively removes data while maintaining cache structure

**Parameters:**
- `transactionId` (string): Transaction ID to clear

**Returns:** `boolean` - Always returns true (node-cache set result)

### Authentication Functions

#### `generateHeader(message)`
**Purpose:** Generates ONDC compliant authorization header using environment credentials.

**Implementation Details:**
- Uses ondc-crypto-sdk-nodejs for signature generation
- Reads credentials from environment variables
- Returns promise for async signature creation

**Parameters:**
- `message` (Object): BECKN protocol message to sign

**Returns:** `Promise<string>` - Authorization header string

**Environment Dependencies:**
- `PRIVATE_KEY`: Base64 encoded signing private key
- `BAPID`: Registered BAP identifier
- `UNIQUE_KEY`: Unique key identifier

#### `createHeaderFromId(body)`
**Purpose:** Alternative header creation function with hardcoded credentials (for testing).

**Implementation Details:**
- Uses hardcoded private key and identifiers
- Includes comprehensive error handling
- Logs header creation process

**Security Note:** Contains hardcoded credentials - should only be used for development.

#### `verifyHeader(req, lookupUrl)`
**Purpose:** Verifies digital signature of incoming ONDC requests.

**Implementation Details:**
- Extracts public key from ONDC registry
- Validates signature using ondc-crypto-sdk-nodejs
- Returns boolean verification result

**Parameters:**
- `req` (Object): Express request object with headers and body
- `lookupUrl` (string): ONDC registry lookup URL

**Returns:** `Promise<boolean>` - True if signature is valid

**Internal Process:**
1. Extracts subscriber ID and unique key ID from authorization header
2. Calls ONDC registry to get public key
3. Validates signature against public key

#### `getPublicKey(lookupUri, header)`
**Purpose:** Retrieves public key from ONDC registry for signature verification.

**Implementation Details:**
- Parses authorization header to extract identifiers
- Makes POST request to ONDC lookup service
- Returns signing public key from registry response

#### `extractSubscriberId(header)`
**Purpose:** Extracts BAP credentials from authorization header.

**Implementation Details:**
- Uses regex to parse keyId from authorization header
- Splits keyId by "|" delimiter to get subscriber ID and unique key ID
- Returns object with both identifiers

**Header Format Expected:**
```
Authorization: Signature keyId="subscriber_id|unique_key_id",algorithm="..."
```

### Session Management Functions

#### `insertSession(session)`
**Purpose:** Stores JSON mapper session data in cache.

**Implementation Details:**
- Prefixes session key with "jm_" for identification
- Uses transaction_id as cache key
- Sets TTL to 10 minutes

**Parameters:**
- `session` (Object): Session object with mapping configuration

#### `handleRequestForJsonMapper(response, unsolicited)`
**Purpose:** Processes ONDC callback responses for JSON mapper system.

**Implementation Details:**
- Finds matching session across all cached sessions
- Extracts business data using configured mappings
- Updates session state for next protocol steps
- Handles both solicited and unsolicited responses

**Parameters:**
- `response` (Object): BECKN protocol response
- `unsolicited` (boolean): Whether response was unsolicited (default: false)

**Complex Logic:**
1. Searches all sessions with "jm_" prefix
2. Matches response to session by transaction_id
3. Finds protocol call configuration by message_id
4. Extracts business payload using mapping rules
5. Updates session state for next request
6. Enables rendering for subsequent protocol calls

---

## Payload Transformation (`utils/buildPayload.js`)

### Core Transformation Functions

#### `createBecknObject(session, call, data)`
**Purpose:** Transforms business-friendly payload to BECKN protocol format.

**Implementation Details:**
- Loads YAML configuration for the session
- Applies session data transformations first
- Converts business payload using mapping rules
- Returns both transformed payload and updated session

**Parameters:**
- `session` (Object): Mapping session with configuration
- `call` (Object): Protocol call configuration
- `data` (Object): Business payload data

**Returns:** `Object` - `{payload, session}` with BECKN payload and updated session

**Transformation Process:**
1. Load protocol mapping YAML
2. Extract mapping configuration for the call
3. Apply session data mappings if configured
4. Transform business data to BECKN format
5. Return transformed payload and updated session

#### `extractBusinessData(type, payload, session)`
**Purpose:** Extracts business-friendly data from BECKN protocol responses.

**Implementation Details:**
- Uses inverse mapping configuration
- Applies extraction rules to convert BECKN to business format
- Handles complex nested object transformations

**Parameters:**
- `type` (string): Protocol call type (search, select, etc.)
- `payload` (Object): BECKN protocol payload
- `session` (Object): Mapping session

**Returns:** `Object` - Business-friendly data object

#### `extractPath(path, obj)`
**Purpose:** Extracts data from nested objects using path expressions.

**Implementation Details:**
- Parses complex path syntax with arrays and conditions
- Supports object transformation syntax
- Uses eval() for dynamic property access (security consideration)

**Parameters:**
- `path` (string): Path expression with special syntax
- `obj` (Object): Source object for extraction

**Returns:** `Object` - Extracted data in `{data: value}` format

**Path Syntax Support:**
- Simple paths: `"message.catalog.providers"`
- Array filtering: `"providers[?(_?.id === 'P1')]"`
- Object transformation: `"items{id: .id, name: .descriptor.name}"`

### Helper Functions

#### `createPayload(config, action, data, session)`
**Purpose:** Creates BECKN payload using mapping configuration.

**Implementation Details:**
- Iterates through mapping configuration
- Evaluates conditions and computations
- Builds nested object structure
- Uses eval() for dynamic value evaluation

**Security Note:** Uses eval() for flexibility but requires trusted configuration.

#### `createNestedField(obj, path, value)`
**Purpose:** Creates nested object properties using dot notation paths.

**Implementation Details:**
- Supports array index notation: `"items[0].descriptor"`
- Creates intermediate objects as needed
- Handles both object and array structures

#### `buildTags(tags)`
**Purpose:** Builds ONDC-specific tag structures from key-value objects.

**Implementation Details:**
- Converts flat object to ONDC tag format
- Creates descriptor objects with codes
- Maintains ONDC tag list structure

**Tag Format:**
```javascript
// Input: {BUYER_FINDER_FEES: {TYPE: "percent", PERCENTAGE: "1"}}
// Output: [{descriptor: {code: "BUYER_FINDER_FEES"}, list: [...]}]
```

#### `decodeInputString(input)`
**Purpose:** Parses complex path expressions with array/object syntax.

**Implementation Details:**
- Tokenizes input string by brackets and braces
- Builds configuration object for extraction logic
- Handles nested array and object transformations

**Syntax Examples:**
- `"message.catalog"` → Simple path access
- `"providers[?(_?.active)]"` → Array with condition
- `"items{id: .id, name: .name}"` → Object transformation

#### `extractData(obj, config, commData)`
**Purpose:** Recursively extracts data using parsed configuration.

**Implementation Details:**
- Handles Array, String, and Object extraction types
- Applies conditions for array filtering
- Merges common data across extractions
- Uses eval() for condition evaluation

#### `constructValueObject(data, key)`
**Purpose:** Constructs value objects from comma-separated mapping strings.

**Implementation Details:**
- Parses "key:path" pairs from comma-separated string
- Converts to extraction configuration objects
- Adds safe property access with "?." operator

#### `constructPath(data)`
**Purpose:** Constructs safe property access paths.

**Implementation Details:**
- Adds "obj." prefix for evaluation context
- Converts dot notation to safe access with "?."
- Handles paths starting with "."

---

## Response Handling (`utils/response.js`)

### Response Template Functions

#### `apiResponse(config, transactionId, messageId)`
**Purpose:** Generates mock ONDC response templates for different protocol calls.

**Implementation Details:**
- Contains extensive mock data for all ONDC actions
- Provides realistic transportation service responses
- Includes provider, item, and fulfillment data structures

**Parameters:**
- `config` (string): Response type (search, select, init, etc.)
- `transactionId` (string): Transaction identifier
- `messageId` (string): Message identifier

**Returns:** `Object` - Complete ONDC response with context and message

**Supported Response Types:**
- `search` → `on_search` with catalog data
- `select` → `on_select` with selected items
- `init` → `on_init` with booking initialization
- `confirm` → `on_confirm` with confirmed booking
- `status` → `on_status` with order status
- `update` → `on_update` with order updates
- `cancel` → `on_cancel` with cancellation details

**Response Structure:**
```javascript
{
  context: {
    action: "on_search",
    bap_id: "example-bap.com",
    transaction_id: transactionId,
    message_id: messageId,
    timestamp: "2023-12-09T13:41:16.161Z",
    // ... other context fields
  },
  message: {
    // Action-specific message content
  }
}
```

**Mock Data Features:**
- Realistic transportation provider data
- Multiple vehicle categories (Metro, Cab, Auto-rickshaw)
- Price ranges and service offerings
- Location coordinates for routes
- Fulfillment and payment information

---

## Configuration Management (`configs/index.js`)

### ConfigLoader Class

#### `constructor()`
**Purpose:** Initializes configuration loader instance.

**Implementation Details:**
- Sets initial config to null
- Prepares instance for async initialization

#### `async init()`
**Purpose:** Loads and processes YAML configuration with schema resolution.

**Implementation Details:**
- Reads main configuration YAML file
- Uses @apidevtools/json-schema-ref-parser for schema resolution
- Dereferences all $ref pointers in configuration
- Stores processed configuration in instance

**Returns:** `Promise<Object>` - Processed configuration object

**Error Handling:**
- Wraps errors in custom Error objects
- Propagates file reading and parsing errors

#### `getConfig()`
**Purpose:** Returns loaded configuration object.

**Returns:** `Object|null` - Configuration object or null if not initialized

### Configuration Structure

The configuration system supports:
- **YAML-based configuration** with JSON schema references
- **Modular flow definitions** for different transportation types
- **Protocol mapping rules** for business-to-BECKN transformation
- **Input field definitions** for form generation
- **Session data mappings** for stateful operations

---

## Authentication (`header.js`)

### Header Creation Functions

#### `createHeader(message)`
**Purpose:** Creates ONDC authorization header with hardcoded credentials.

**Implementation Details:**
- Uses predefined private key and BAP identifiers
- Calls ondc-crypto-sdk-nodejs for signature generation
- Includes error handling and logging

**Parameters:**
- `message` (Object): BECKN protocol message to sign

**Returns:** `Promise<string>` - Authorization header

**Hardcoded Values:**
- `SIGNING_PRIVATE_KEY`: Base64 encoded private key
- `BAP_ID`: "mobility-staging.ondc.org"
- `UNIQUE_KEY_ID`: "UK-MOBILITY"

**Security Considerations:**
- Contains hardcoded production-like credentials
- Should be replaced with environment variables for production
- Logs header creation for debugging

---

## Router Functions (`router/route.js`)

### Route Handler Analysis

#### Environment Configuration
The router relies on several environment variables:
- `mockUrl`: Mock service URL
- `callbackUrl`: Callback base URL  
- `GATEWAY_URL`: ONDC gateway URL
- `ENABLE_SIGNATURE_VALIDATION`: Toggle for signature verification
- `LOOKUP_URL`: ONDC registry lookup URL

#### Request Processing Flow

1. **Header Generation Route (`/createHeader`)**
   - Generates authorization header for any message
   - Sets CORS headers for browser access
   - Returns original message with authorization header

2. **Protocol Method Routes (`/:method`)**
   - Validates required fields (bap_uri, transaction_id, bpp_uri)
   - Automatically sets callback URI
   - Determines target URL (gateway for search, BPP for others)
   - Generates signature and forwards request
   - Caches request for debugging

3. **Callback Handler (`/ondc/:method`)**
   - Optionally verifies digital signatures
   - Logs incoming callback data
   - Processes requests through JSON mapper if configured
   - Returns ACK/NACK responses

4. **Cache Management Routes**
   - GET `/cache`: Retrieves transaction data
   - DELETE `/cache`: Clears transaction data

5. **JSON Mapper Routes**
   - `/mapper/session`: Creates mapping sessions
   - `/mapper/:config`: Executes protocol calls with business payloads
   - `/mapper/timeout`: Handles timeout scenarios
   - `/mapper/extractPath`: Path extraction utility

#### Error Handling Patterns

The router implements consistent error handling:
- Validation errors return 400 status
- Missing sessions return 400 with descriptive messages
- Server errors return 500 status
- Network errors from downstream services are propagated

#### Security Features

- Digital signature verification (optional)
- Request validation against ONDC requirements
- Header authentication for all outbound requests
- CORS configuration for cross-origin access

---

## Implementation Notes

### Security Considerations

1. **Eval Usage**: Several functions use `eval()` for dynamic property access and condition evaluation. This requires trusted configuration data.

2. **Hardcoded Credentials**: Some functions contain hardcoded credentials for development/testing purposes.

3. **Input Validation**: Path expressions and mapping configurations should be validated before processing.

### Performance Considerations

1. **Caching Strategy**: Uses node-cache with TTL for temporary data storage. Consider Redis for production scaling.

2. **YAML Processing**: Configuration loading involves file I/O and parsing. Cache processed configurations for better performance.

3. **HTTP Requests**: All outbound ONDC requests include signature generation overhead.

### Error Handling Patterns

1. **Graceful Degradation**: Functions return empty objects or null rather than throwing errors where possible.

2. **Comprehensive Logging**: Error conditions are logged with context for debugging.

3. **Promise Handling**: Async functions properly handle promise rejections.

### Configuration Flexibility

The system is designed for configuration-driven operation:
- Protocol mappings defined in YAML
- Business logic separated from protocol logic
- Modular flow definitions for different use cases

This architecture allows for easy adaptation to different transportation domains and business requirements without code changes.