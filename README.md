# Inventory Ant System Architecture & Functional Specifications

Welcome to the technical specification and functional documentation for **Inventory Ant**, an intelligent warehouse and inventory management platform. This document fully details the system's architecture, APIs, core algorithms, database schema, and operational mechanics, omitting UI/style details.

---

## 1. System Architecture Overview

The system is composed of three primary microservices:
1. **Frontend (Vite + React)**: Serves as the dashboard and command console. Communicates with the backend using REST requests. Includes a persistent voice assistant core mounting custom Web Speech Speech-to-Text hooks.
2. **Backend (NestJS API)**: The core transaction, search, and orchestration layer. It exposes REST APIs for database operations, processes files, maps schemas dynamically, and handles Gemini AI orchestrations.
3. **AI Microservice (FastAPI)**: A lightweight Python microservice supporting supplementary pipeline integrations.

---

## 2. Core Functional Features

### A. Intelligent Hinglish Voice Command Processor (Ant X V2 Engine)
Located at `ProductsService.processAgentCommandV2`, this parses human speech commands (Hindi/English/Hinglish mixture) into structured inventory updates.

1. **Context Memory & Reference Binding**:
   - The backend tracks a persistent, memory-mapped session structure (`UserSession`) per user containing `lastProductId` and `lastProductName`.
   - When a user says context-dependent words (e.g., *"usmein 10 add karo"* or *"iska stock kam karo"*), the algorithm automatically binds the action to the last referenced product.
2. **Intent Parsing using Gemini 2.5**:
   - Utilizes `gemini-2.5-flash` under structured JSON output configuration.
   - Extracts:
     - `action`: `IN` (Inbound stock increase), `OUT` (Outbound stock decrease), `NAVIGATE` (Change screen), `CHAT` (Informational query), `QUERY` (Check stock values), `LOGIN` (Switch credentials), or `WIPE` (Clear database).
     - `itemName` / `productId`
     - `qty` (Quantity count)
     - `page` (Target viewport navigation slug)
   - Fallback logic: If the API endpoint is unavailable, a regex-based backup parser executes locally to parse codes, numbers, and basic operation keywords (e.g. "add", "remove", "kam", "hi").
3. **Ambiguity Resolution & Match Verification**:
   - Compares query tokens to database values using strict matching and fuzzy substring evaluations.
   - If multiple candidates are matched (e.g., matching "Studymate Notebook 80pg" and "Studymate Notebook 100pg" for query "Studymate Notebook"), the system blocks execution, flags the state as ambiguous, and instructs the voice engine to prompt the user for specific parameters (e.g., unique Item Code or details).
4. **Exception Handling & Confirmation Staging**:
   - **Exceeding Outbounds**: If the user requests an `OUT` action representing a quantity larger than current stock, the system generates a warning and stages a `PendingAction` asking: *"Aapke paas sirf X units hain. Zero kar doon?"*
   - **Large Batches**: Any outbound or inbound action representing a quantity > 50 prompts a verification prompt to prevent accidental massive increments.
   - **Destructive Commands**: Database deletion requests trigger a staged validation requiring vocal confirmation before executing.

### B. Intelligent Invoice Document Scanner
Located at `ProductsService.processBillWithGemini`, this processes invoice files (PNG, JPG, PDF) uploaded via Inbound or Outbound gates.

1. **Gemini Vision Extraction**:
   - Submits base64 invoice binary to `gemini-2.5-flash` to extract items list mapping `productId` (SKU/barcode), `name`, `qty`, and `mrp`.
2. **Double Fallback Local OCR**:
   - If the Gemini API fails, the service initiates a local `Tesseract.js` worker.
   - It performs Optical Character Recognition (OCR) on the image.
   - Passes extracted text into a custom line-by-line parser which filters headers, taxes, subtotal values, and isolates columns to deduce product metadata automatically.

### C. Voice Feedback System (Text-To-Speech)
Located at `ProductsService.getGoogleTTS`, this implements audio responses.
- Generates speech audio using a Google TTS proxy API.
- Generates an MP3 stream using a Hindi/Hinglish accent voice synthesizer (`tl=hi`) and sends the raw buffer output to the client.

### D. User Isolation Model
- The system is multi-tenant.
- All requests require the HTTP header `x-user-id` to perform credentials checks.
- Inside `database.json`, all records are grouped, filtered, and isolated by the parsed `userId` value, preventing data leakage between nodes.

---

## 3. Database Schema

All system data is stored in `database.json` as a flat collection of `Product` records. The model supports flexible mapping, preserving custom columns parsed from user CSV uploads.

```typescript
export interface Product {
  id: string;          // System-generated unique transaction key
  userId: string;      // User identifier (tenant mapping)
  productId?: string;  // Customer-provided unique Item Code (SKU)
  name?: string;       // Product description label
  quantity?: string;   // Current available stock unit count
  mrp?: string;        // Max Retail Price
  details?: string;    // Extra description
  _timestamp: number;  // Unix time of item registry creation
  _headers?: object;   // Dynamic custom schema headers map for bulk uploads
  [key: string]: any;  // Dynamically mapped columns (e.g., expiry date, category)
}
```

---

## 4. API Endpoint Reference

All endpoints are prefix-routed to `/products`.

### `POST /products/scan-bill`
Parses an invoice file to modify stock.
- **Headers**: `x-user-id: <string>`
- **Request Body**:
  ```json
  {
    "fileName": "invoice.png",
    "fileType": "image/png",
    "base64Image": "iVBORw0KGgo...",
    "actionType": "IN" // or "OUT"
  }
  ```
- **Response Schema** (Success):
  ```json
  {
    "success": true,
    "action": "IN",
    "parsedItems": [
      { "productId": "109", "name": "Marker Pen Blue", "qty": 10, "mrp": "25" }
    ],
    "auditLog": [
      "SUCCESS: [109] Marker Pen Blue +10 (Total: 45)"
    ]
  }
  ```

### `POST /products/agent-command-v2`
Executes vocal commands processed by browser Speech-to-Text.
- **Headers**: `x-user-id: <string>`
- **Request Body**:
  ```json
  {
    "text": "10 notebooks add kar do",
    "currentView": "dashboard",
    "role": "operator" // or "manager" (changes tone and verbosity)
  }
  ```
- **Response Schema**:
  ```json
  {
    "success": true,
    "speechText": "Sure, confirm kar diya! Notebook ke 10 items add kar diye hain.",
    "action": "IN",
    "shouldUpdateUI": true
  }
  ```

### `GET /products/tts`
Synthesizes speech to voice.
- **Parameters**: `?text=<string>`
- **Response**: Binary audio stream (`audio/mpeg`).

### `POST /products/bulk`
Bulk imports dynamically matched inventory columns from CSV parse lists.
- **Headers**: `x-user-id: <string>`
- **Request Body**: Array of dynamically formatted product structures.
- **Response**:
  ```json
  {
    "count": 142
  }
  ```

### `POST /products/sell`
Executes cart checkout deductions.
- **Headers**: `x-user-id: <string>`
- **Request Body**:
  ```json
  [
    { "id": "sys_9af241", "quantity": 2 }
  ]
  ```
- **Response**: `{"success": true}`

### `GET /products`
Retrieves the logged-in user's isolated inventory items.
- **Headers**: `x-user-id: <string>`
- **Response**: Array of product objects.

### `POST /products`
Creates a product record manually.
- **Headers**: `x-user-id: <string>`
- **Request Body**: Product fields.

### `PUT /products/:id`
Updates an existing product record.
- **Headers**: `x-user-id: <string>`
- **Request Body**: Updated attributes.

### `DELETE /products/:id`
Deletes a single product from registry.
- **Headers**: `x-user-id: <string>`

### `DELETE /products/all`
Wipes all inventory records mapping to the headers' `x-user-id`.
- **Headers**: `x-user-id: <string>`
