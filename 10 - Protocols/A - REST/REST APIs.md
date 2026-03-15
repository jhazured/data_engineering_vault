
**Tags:** #rest #api #http #web-services #authentication #oauth #pagination

## Overview

A REST API (Representational State Transfer Application Programming Interface) is a web service that follows REST principles to allow systems to communicate over HTTP using simple, predictable URLs and standard HTTP methods like GET, POST, PUT, and DELETE.

## Common HTTP Methods in REST

|Method|Purpose|Example URL|
|---|---|---|
|GET|Read or retrieve data|/api/users|
|POST|Create a new resource|/api/users|
|PUT|Update an existing resource|/api/users/123|
|DELETE|Remove a resource|/api/users/123|

---

## Basic REST Example

### Request: Create a New User

```http
POST /api/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

### Response

```json
{
  "id": 101,
  "name": "Alice",
  "email": "alice@example.com",
  "created_at": "2025-06-30T10:15:00Z"
}
```

---

## Complex REST API Examples

**API Base URL:** `https://api.example.com/v1`

### 1. GET – Retrieve Filtered List of Orders

```http
GET /v1/orders?status=shipped&start_date=2025-06-01&end_date=2025-06-30&page=2&page_size=20
Authorization: Bearer eyJhbGciOi...
Accept: application/json
```

**Description:** Retrieves a paginated list of orders with status=shipped between June 1–30, 2025.

**Response (200 OK):**

```json
{
  "page": 2,
  "page_size": 20,
  "total": 136,
  "orders": [
    {
      "order_id": "ORD202506123",
      "customer": "Jane Doe",
      "status": "shipped",
      "total": 154.00,
      "shipped_date": "2025-06-20"
    }
  ]
}
```

### 2. POST – Create a New Order

```http
POST /v1/orders
Content-Type: application/json
Authorization: Bearer eyJhbGciOi...

{
  "customer_id": "CUST12345",
  "items": [
    { "product_id": "PROD1001", "quantity": 2 },
    { "product_id": "PROD1010", "quantity": 1 }
  ],
  "shipping_address": {
    "street": "123 Elm St",
    "city": "Springfield",
    "state": "IL",
    "zip": "62704",
    "country": "USA"
  },
  "payment_method": "credit_card",
  "notes": "Leave package at the front door"
}
```

**Response (201 Created):**

```json
{
  "order_id": "ORD20250630123",
  "status": "processing",
  "estimated_delivery": "2025-07-03"
}
```

### 3. PUT – Update Existing Order's Shipping Address

```http
PUT /v1/orders/ORD20250630123
Content-Type: application/json
Authorization: Bearer eyJhbGciOi...

{
  "shipping_address": {
    "street": "456 Oak Ave",
    "city": "Shelbyville",
    "state": "IL",
    "zip": "62705",
    "country": "USA"
  }
}
```

**Response (200 OK):**

```json
{
  "order_id": "ORD20250630123",
  "status": "processing",
  "shipping_address": {
    "street": "456 Oak Ave",
    "city": "Shelbyville",
    "state": "IL",
    "zip": "62705",
    "country": "USA"
  }
}
```

### 4. DELETE – Cancel an Order

```http
DELETE /v1/orders/ORD20250630123
Authorization: Bearer eyJhbGciOi...
```

**Response (204 No Content):** No response body, just confirmation the order was deleted/canceled.

### Key Features Demonstrated

|Feature|Where It Applies|
|---|---|
|Auth (Bearer token)|Used in all requests to authorize the user|
|Pagination|Seen in GET with page, page_size|
|Nested JSON|Used in POST and PUT for complex data|
|Response Codes|200 OK, 201 Created, 204 No Content|

---

## API Tools

|Tool|Purpose|
|---|---|
|Swagger / OpenAPI|Define REST API specs (machine & human readable)|
|Postman|Test and explore APIs interactively|

## Advanced REST Concepts

|Concept|Why It Matters|
|---|---|
|Rate Limiting|Control usage to prevent abuse|
|Caching (ETag, Cache-Control)|Reduce load and speed up response|
|Versioning|Manage API lifecycle without breaking old clients|
|Webhooks|API that pushes data to your service when events happen|
|Idempotency|Ensures safe retrying of requests|
|HATEOAS (REST principle)|API responses include links to related actions|
|Pagination|Manage large datasets (page, limit, cursor, etc.)|

---

## HTTP Response Codes

HTTP response codes are crucial for understanding the outcome of a REST API request. Each response code has a standard meaning that tells the client what happened.

### 2xx – Success

Indicates that the request was successfully received, understood, and processed.

|Code|Meaning|When Used|
|---|---|---|
|200 OK|Standard success|GET, PUT, PATCH, or DELETE completed normally|
|201 Created|Resource created|POST created a new resource (e.g. new user/order)|
|202 Accepted|Request accepted|Asynchronous processing; the action will complete later|
|204 No Content|Success, no body returned|DELETE successful or PUT didn't need to return anything|

### 4xx – Client Error

Indicates a problem with the request (bad input, missing data, etc.)

|Code|Meaning|When Used|
|---|---|---|
|400 Bad Request|Request is malformed|Invalid JSON, missing required fields|
|401 Unauthorized|Authentication failed|Token missing, expired, or incorrect|
|403 Forbidden|Authenticated, but not allowed|Authenticated but lacks permission|
|404 Not Found|Resource doesn't exist|Wrong URL or ID not found|
|405 Method Not Allowed|Wrong HTTP method|Used POST instead of GET, etc.|
|429 Too Many Requests|Rate limiting|Exceeded API call limits|

### 5xx – Server Error

These indicate that the server failed to fulfill a valid request.

|Code|Meaning|When Used|
|---|---|---|
|500 Internal Server Error|Unexpected server error|Generic, catch-all failure (often a bug)|
|502 Bad Gateway|Proxy/gateway issue|Upstream server failure (e.g., if using NGINX)|
|503 Service Unavailable|Server temporarily overloaded|Maintenance or too many connections|
|504 Gateway Timeout|Upstream server didn't respond|Server didn't get a response in time from another service|

### Response Code Summary

|Code Group|Meaning|Example|
|---|---|---|
|2xx|Success|200 OK, 201 Created|
|4xx|Client Error|400 Bad Request, 404 Not Found|
|5xx|Server Error|500 Internal Server Error|

---

## Pagination

Pagination is a technique used in APIs to break large sets of data into smaller, manageable chunks, or "pages", so that clients don't get overwhelmed by loading everything at once.

### Why Pagination Matters

Imagine an API that returns all 10,000 user records in one go — this would:

- Be slow
- Consume a lot of memory
- Possibly crash the client or server
- Waste bandwidth if you only need the first 20 users

Pagination solves this by returning only a limited number of results per request, along with info that lets the client fetch the next (or previous) page.

### Common Pagination Parameters

|Parameter|Description|Example|
|---|---|---|
|page|Which page you want|page=2|
|page_size|How many results per page|page_size=25|
|limit|Maximum number of records to return|limit=10|
|offset|How many records to skip before starting|offset=30|
|cursor|Used in cursor-based pagination (next token)|cursor=xyz123|

### Example: Page-based Pagination

**Request:**

```http
GET /api/users?page=2&page_size=5
```

**Response:**

```json
{
  "page": 2,
  "page_size": 5,
  "total_records": 23,
  "users": [
    { "id": 6, "name": "User 6" },
    { "id": 7, "name": "User 7" },
    { "id": 8, "name": "User 8" },
    { "id": 9, "name": "User 9" },
    { "id": 10, "name": "User 10" }
  ]
}
```

### Types of Pagination

|Type|How It Works|Best Use Case|
|---|---|---|
|Page-based|page & page_size (or limit + offset)|Easy to implement|
|Offset-based|Use offset=N and limit=M|Useful for SQL queries|
|Cursor-based|Uses a token (cursor) to fetch next/prev page|Good for large, real-time data sets (e.g. Twitter)|

### Pagination Best Practices

- **Use Standard Parameters:** Employ universally recognized parameter names like page, page_size, offset, or limit for pagination controls
- **Provide Metadata:** Include pagination metadata in responses, such as total, total_pages, current_page, next_page, and prev_page
- **Limit Page Size:** Set a maximum limit for page_size to prevent clients from requesting too much data at once
- **Handle Edge Cases:** Implement logic to handle scenarios like empty pages, last pages, and invalid parameters gracefully

---

## OAuth 2.0

OAuth 2.0 is an authorization framework that allows third-party applications to access a user's resources (like data or services) without sharing the user's credentials (like their password). It's the standard protocol used for secure authorization on the web.

### How OAuth 2.0 Works

Here's how a typical OAuth 2.0 authorization code flow works:

**Client App → Authorization Server → Resource Server (API)**

#### Step-by-Step Process

1. **User Requests Login via Third-Party App**  
    The client app (e.g. Zoom) sends you to the authorization server (e.g. Google) to log in.
    
2. **Authorization Server Prompts User**  
    Google asks, "Do you want to allow Zoom to access your calendar?"
    
3. **User Approves or Denies Access**  
    If approved, Google sends an authorization code to Zoom.
    
4. **Client App Exchanges Code for Access Token**  
    Zoom sends that code (securely) to Google's token endpoint and receives an access token.
    
5. **Client Uses Access Token to Access API**  
    Zoom uses the access token to call the Google Calendar API.
    

### Example: Authorization Code Flow

**1. User redirected to authorization URL:**

```http
GET https://auth.example.com/authorize?
  response_type=code&
  client_id=abc123&
  redirect_uri=https://app.example.com/callback&
  scope=read_profile email
```

**2. User approves → redirected with code:**

```
https://app.example.com/callback?code=xyz456
```

**3. Client exchanges code for token:**

```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

client_id=abc123
client_secret=shhh
code=xyz456
grant_type=authorization_code
redirect_uri=https://app.example.com/callback
```

**4. Response:**

```json
{
  "access_token": "eyJhbGciOi...",
  "expires_in": 3600,
  "refresh_token": "xyz789",
  "token_type": "Bearer"
}
```

### Key Components of OAuth 2.0

|Component|Description|
|---|---|
|Resource Owner|The user who owns the data or account|
|Client|The third-party application requesting access|
|Authorization Server|Where the user authenticates (e.g. Google, GitHub, Microsoft)|
|Resource Server|The API or service holding the user's data (e.g. Google Calendar)|
|Access Token|Short-lived token used by the client to access protected resources|
|Refresh Token|Long-lived token used to get a new access token after it expires (optional)|

### Grant Types

In OAuth 2.0, a grant type defines the way an application gets an access token. Each grant type is used in different contexts, based on the app type and security needs.

|Flow Name|Used For|Example|
|---|---|---|
|Authorization Code|Web & mobile apps with user login|Google login on a web app|
|PKCE (w/ Auth Code)|Secure for mobile/SPAs|Spotify mobile login|
|Client Credentials|Server-to-server (no user)|Backend service-to-service|
|Password (deprecated)|Direct login with user credentials (avoid)|Legacy apps only|
|Implicit (deprecated)|Frontend-only apps (unsafe)|Replaced by PKCE|

### Common Scopes in OAuth

In OAuth 2.0, a scope is a permission — it defines what access a third-party application is requesting on behalf of the user.

#### Examples of Scopes

|Provider|Scope|What It Allows|
|---|---|---|
|Google|email|Read user's email address|
||calendar.readonly|Read-only access to the user's Google Calendar|
||drive.file|Access to files the user created with your app|
|GitHub|repo|Full control of private repositories|
||read:user|Read public user profile info|
|Microsoft|User.Read|Read basic user profile from Microsoft Graph|
||Mail.Send|Send mail as the user|
|Spotify|playlist-modify-public|Modify a user's public playlists|

#### How Scopes Work in the Flow

The client includes scopes in the authorization URL:

```http
GET https://auth.example.com/authorize?
response_type=code&
client_id=client123&
scope=read_profile write_calendar
```

- The authorization screen shows what the app is asking for
- The access token returned will be limited to those scopes
- The resource server (API) checks that the access token has the right scope before allowing access

#### Why Scopes Matter

- **🔐 Security:** Limit access to the minimum necessary
- **✅ User Control:** Clear visibility into what the app can do
- **💡 API Design:** Scopes help you group and manage API capabilities

### Scope Features

|Feature|Description|
|---|---|
|Required by API|Some APIs won't work unless specific scopes are included|
|Partial approval|Some providers let users reject individual scopes (rare)|
|Scope in token|Many OAuth tokens carry scopes in their payload (JWT claims)|
|Scope validation|Resource server must check that the token has the right scope|

> [!important] OAuth vs. Authentication
> 
> - **OAuth = Authorization** (what the app can do)
> - **Authentication = Login/Identity** (who the user is)
> 
> Combine OAuth with OpenID Connect (OIDC) to handle authentication.

### OAuth 2.0 Best Practices

- Use PKCE for public clients (mobile, SPA)
- Never expose client secrets in front-end apps
- Always use HTTPS
- Use short-lived tokens and refresh tokens

---

## Registering an OAuth 2.0 App

To register an app with OAuth 2.0, you need to create a client on the OAuth authorization server. This registration process gives your app a client ID and (sometimes) a client secret.

### General OAuth 2.0 App Registration Steps

1. **Log in to the Authorization Server's Developer Console**
    
    - Google Cloud Console
    - [GitHub Developer Settings](https://github.com/settings/developers)
    - Microsoft Azure Portal
    - Your own OAuth server (if self-hosted)
2. **Create/Register a New Application** Provide:
    
    - App name
    - Description
    - Logo (optional)
    - Website or contact info
3. **Set Redirect URI(s)** This is where the OAuth server will send the user back with the code or token after login. Example: `https://yourapp.com/oauth/callback`
    
4. **Choose the Allowed Scopes** Define which user data your app will ask for:
    
    - email, profile, calendar.readonly, etc.
5. **Choose Grant Types** Specify which OAuth 2.0 flows your app will support:
    
    - Authorization Code
    - PKCE
    - Client Credentials
    - Refresh Tokens
6. **Save & Copy the Credentials** After registering, you'll get:
    
    - Client ID (public identifier)
    - Client Secret (keep this private; only for confidential apps)

---

## Register OAuth 2.0 App in Azure AD

### 1. Go to Azure Portal

Visit [https://portal.azure.com](https://portal.azure.com/)

### 2. Search for "App registrations"

- In the top search bar, type **App registrations**
- Click on **App registrations** under Services

### 3. Click "New registration"

|Field|What to Enter|
|---|---|
|Name|Friendly app name (e.g. My CRM App)|
|Supported account types|Choose who can use the app:<br>- Single tenant<br>- Multitenant<br>- With personal Microsoft accounts|
|Redirect URI|https://yourapp.com/auth/callback (or http://localhost:3000/callback for dev)|

> [!note] You can add more redirect URIs later.

Click **Register**

### 4. After Registration

|Field|Description|
|---|---|
|Application (client) ID|Public identifier (used in token requests)|
|Directory (tenant) ID|Your Azure AD tenant's unique ID|

### 5. (Optional) Create a Client Secret

If you're building a confidential client (like a backend app):

1. In the left menu, go to **Certificates & secrets**
2. Click **New client secret**
3. Add a description and expiration
4. Click **Add**
5. **Copy the client secret immediately** — you won't see it again!

### 6. (Optional) Define API Permissions

1. In the left menu, go to **API permissions**
2. Click **+ Add a permission**
3. Choose **Microsoft Graph** (or another API)
4. Select permissions like:
    - User.Read (read profile info)
    - Mail.Read (read email)
    - Calendars.ReadWrite
5. Click **Add permissions**
6. For certain permissions, click **Grant admin consent** if needed

### 7. Final Credentials

|Credential|Use|
|---|---|
|Client ID|Identifies your app to Azure AD|
|Tenant ID|Identifies the Azure AD instance|
|Client Secret|Used to authenticate your app (keep safe)|
|Redirect URI|Where Azure will send auth code/token|

---

## Related Topics

- [[SOAP (Simple Object Access Protocol)|SOAP]] - Compare with SOAP web services
- [[gRPC & GraphQL]] - Modern API protocols (binary, schema-first)
- [[Data Contracts & Schema Enforcement]] - Schema enforcement across API boundaries
- [[Trust Stores & Certificate Management]] - SSL/TLS for HTTPS connections