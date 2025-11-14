# Authentication Migration - JWT Token Approach

## Overview
Moving from frontend AWS Cognito to backend JWT token-based authentication. Backend will issue access tokens (15-20 min expiry) and refresh tokens (1 day expiry) via HTTP-only cookies. No Redis or session storage needed - tokens are stateless and validated using secret key.

---

## How It Works

**Login Flow:**
1. User enters email/password → Frontend sends to backend
2. Backend authenticates with AWS Cognito
3. Backend generates JWT access token (15 min) and refresh token (1 day)
4. Backend sends both tokens as HTTP-only cookies
5. Frontend gets user data in response (no tokens visible)

**API Request Flow:**
1. Browser automatically sends cookies with every request
2. Backend checks access token from cookie
3. Backend verifies token signature using secret key
4. If valid → Process request
5. If expired → Auto-generate new access token using refresh token
6. Return data to frontend

**No session storage needed** - Everything is in the JWT token itself (stateless).

---

## Backend Tasks

### 1. Environment Setup
- [ ] Create Node.js backend project (or add to existing)
- [ ] Install packages:
  ```bash
  npm install express
  npm install jsonwebtoken
  npm install @aws-sdk/client-cognito-identity-provider
  npm install cookie-parser
  npm install cors
  npm install dotenv
  npm install helmet
  npm install express-rate-limit
  ```
- [ ] Set up `.env` file:
  ```
  AWS_COGNITO_USER_POOL_ID=your_pool_id
  AWS_COGNITO_CLIENT_ID=your_client_id
  AWS_REGION=us-east-1
  JWT_ACCESS_SECRET=your_super_secret_key_for_access_token
  JWT_REFRESH_SECRET=your_super_secret_key_for_refresh_token
  NODE_ENV=development
  ```

### 2. JWT Token Service
- [ ] Create `services/tokenService.js`
- [ ] Write function to generate access token:
  - [ ] Include user data (userId, email, customerId, role, permissions)
  - [ ] Set expiry to 15-20 minutes
  - [ ] Sign with JWT_ACCESS_SECRET
  ```javascript
  // Example structure
  const accessToken = jwt.sign(
    { userId, email, customerId, role, permissions },
    JWT_ACCESS_SECRET,
    { expiresIn: '15m' }
  );
  ```
- [ ] Write function to generate refresh token:
  - [ ] Include only userId and email
  - [ ] Set expiry to 1 day
  - [ ] Sign with JWT_REFRESH_SECRET
  ```javascript
  const refreshToken = jwt.sign(
    { userId, email },
    JWT_REFRESH_SECRET,
    { expiresIn: '1d' }
  );
  ```
- [ ] Write function to verify access token (using JWT_ACCESS_SECRET)
- [ ] Write function to verify refresh token (using JWT_REFRESH_SECRET)
- [ ] Write function to decode token and extract user data

### 3. AWS Cognito Service
- [ ] Create `services/cognitoService.js`
- [ ] Write function to authenticate user:
  - [ ] Call AWS Cognito with email + password
  - [ ] Return user attributes (userId, email, firstName, lastName)
  - [ ] Handle errors (wrong password, account locked, etc.)
- [ ] Write function to change password via Cognito
- [ ] Write function to reset password

### 4. Database Service
- [ ] Create `services/userService.js`
- [ ] Write function to get user details by userId:
  - [ ] Query users table
  - [ ] Get systemRole, clientStatus, projectStatus, permissions
- [ ] Write function to get company details by customerId:
  - [ ] Get companyName, countryId, settings
- [ ] Write function to get theme settings by customerId:
  - [ ] Get colors, fonts, logo, etc.
- [ ] Write function to get custom messages by customerId
- [ ] Create one combined function that fetches everything:
  ```javascript
  async function getAllUserData(userId) {
    const user = await getUserById(userId);
    const company = await getCompanyById(user.customerId);
    const theme = await getThemeById(user.customerId);
    const messages = await getMessageById(user.customerId);

    return { user, company, theme, messages };
  }
  ```

### 5. Login Endpoint - POST /api/auth/login
- [ ] Create `routes/auth.js` and `controllers/authController.js`
- [ ] Implement login logic:
  - [ ] Validate email and password from request body
  - [ ] Call Cognito service to authenticate
  - [ ] If Cognito success, get userId from response
  - [ ] Fetch all user data from database (user + company + theme + messages)
  - [ ] Check if account is active (userstatus field)
  - [ ] If account disabled → Return 403 error
  - [ ] Generate access token with all user data
  - [ ] Generate refresh token with userId only
  - [ ] Set both tokens as HTTP-only cookies:
    ```javascript
    res.cookie('accessToken', accessToken, {
      httpOnly: true,
      secure: true, // HTTPS only
      sameSite: 'strict',
      maxAge: 15 * 60 * 1000 // 15 minutes
    });

    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 24 * 60 * 60 * 1000 // 1 day
    });
    ```
  - [ ] Return user data in response (no tokens in response body)
- [ ] Add error handling:
  - [ ] 400 - Invalid email/password format
  - [ ] 401 - Wrong credentials
  - [ ] 403 - Account disabled
  - [ ] 500 - Server error

### 6. Token Verification Middleware
- [ ] Create `middleware/verifyToken.js`
- [ ] Extract access token from cookie
- [ ] Verify access token using JWT_ACCESS_SECRET:
  ```javascript
  try {
    const decoded = jwt.verify(accessToken, JWT_ACCESS_SECRET);
    req.user = decoded; // Attach user data to request
    next(); // Token valid, proceed
  } catch (error) {
    // Token expired or invalid
  }
  ```
- [ ] If access token expired:
  - [ ] Extract refresh token from cookie
  - [ ] Verify refresh token using JWT_REFRESH_SECRET
  - [ ] If refresh token valid:
    - [ ] Get userId from refresh token
    - [ ] Fetch fresh user data from database
    - [ ] Generate new access token with updated data
    - [ ] Set new access token cookie
    - [ ] Attach user data to req.user
    - [ ] Continue processing request
  - [ ] If refresh token also expired or invalid:
    - [ ] Return 401 Unauthorized
    - [ ] Clear both cookies
    - [ ] Frontend should redirect to login
- [ ] This middleware runs on every protected API route

### 7. Logout Endpoint - POST /api/auth/logout
- [ ] Simply clear both cookies:
  ```javascript
  res.cookie('accessToken', '', { maxAge: 0 });
  res.cookie('refreshToken', '', { maxAge: 0 });
  return res.json({ success: true });
  ```
- [ ] No need to invalidate tokens (they expire naturally)

### 8. Get Current User - GET /api/auth/me
- [ ] Use verifyToken middleware
- [ ] Return user data from req.user (already decoded from token):
  ```javascript
  router.get('/me', verifyToken, (req, res) => {
    res.json({ success: true, user: req.user });
  });
  ```
- [ ] If token expired, middleware auto-refreshes and returns updated data

### 9. Security Setup
- [ ] Configure CORS in `app.js`:
  ```javascript
  app.use(cors({
    origin: ['https://app.hirewing.ca', 'http://localhost:3000'],
    credentials: true, // Important for cookies
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
  }));
  ```
- [ ] Add Helmet.js for security headers
- [ ] Add rate limiting on login endpoint:
  ```javascript
  const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // 5 attempts
    message: 'Too many login attempts, try again later'
  });

  app.post('/api/auth/login', loginLimiter, authController.login);
  ```
- [ ] Add request logging (IP, timestamp, endpoint)

### 10. Apply Middleware to Protected Routes
- [ ] Apply verifyToken middleware to all protected routes:
  ```javascript
  // Example
  router.get('/api/applicants', verifyToken, applicantsController.getAll);
  router.post('/api/jobs', verifyToken, jobsController.create);
  router.get('/api/dashboard', verifyToken, dashboardController.getData);
  ```
- [ ] Middleware automatically checks token and refreshes if needed
- [ ] req.user contains decoded user data (customerId, userId, role, etc.)

### 11. Backend Testing
- [ ] Test login with correct credentials → Should set 2 cookies
- [ ] Test login with wrong password → Should return 401
- [ ] Test login with disabled account → Should return 403
- [ ] Test access token expiration (wait 15 min or change expiry to 1 min for testing)
- [ ] Test automatic refresh when access token expires
- [ ] Test refresh token expiration (wait 1 day or change expiry)
- [ ] Test logout → Cookies should be cleared
- [ ] Test GET /api/auth/me with valid token
- [ ] Test protected route with expired access token but valid refresh token
- [ ] Test rate limiting on login (try 6+ times)
- [ ] Check cookies in browser DevTools (should be httpOnly, secure, sameSite)

---

## Frontend Tasks

### 1. Remove AWS Cognito SDK
- [ ] Uninstall package:
  ```bash
  npm uninstall amazon-cognito-identity-js
  ```
- [ ] Remove all AWS Cognito imports from files
- [ ] Keep `CognitoSlice.js` commented out for reference (delete later)

### 2. Update Login Component (Login.jsx)
- [ ] Remove all Cognito code:
  - [ ] Remove `signInWithEmailAsync` dispatch
  - [ ] Remove `getAccessTokenAsync` dispatch
  - [ ] Remove AWS Cognito related functions
- [ ] Create new login API call:
  ```javascript
  const handleLogin = async (email, password) => {
    try {
      const response = await axios.post('/api/auth/login', {
        email,
        password,
        rememberMe
      }, {
        withCredentials: true // Important! Allows cookies
      });

      // Response contains user data (no tokens)
      const userData = response.data.user;

      // Store user data in Redux
      dispatch(updateAuth(userData));

      // Navigate to dashboard
      navigate('/ats/layout/dashboard');

    } catch (error) {
      if (error.response?.status === 401) {
        showError('Invalid email or password');
      } else if (error.response?.status === 403) {
        showError('Account disabled. Contact admin.');
      }
    }
  };
  ```
- [ ] Remove localStorage token storage code
- [ ] Remove encryption/decryption code (not needed)

### 3. Configure Axios
- [ ] Update Axios config file or create new one:
  ```javascript
  import axios from 'axios';

  const api = axios.create({
    baseURL: process.env.REACT_APP_API_URL, // e.g., https://api.hirewing.ca
    withCredentials: true, // CRITICAL - sends cookies with every request
    headers: {
      'Content-Type': 'application/json'
    }
  });

  export default api;
  ```
- [ ] Use this configured axios instance for all API calls
- [ ] Add interceptor for 401 errors (redirect to login):
  ```javascript
  api.interceptors.response.use(
    (response) => response,
    (error) => {
      if (error.response?.status === 401) {
        // Token expired and refresh failed
        localStorage.clear(); // Clear any stored data
        window.location.href = '/ats/login';
      }
      return Promise.reject(error);
    }
  );
  ```

### 4. Update API Helper (_apihelper.js)
- [ ] Remove all token-related code:
  - [ ] Remove `getAccessToken()` function
  - [ ] Remove Authorization header injection
  - [ ] Remove bearer token code
  - [ ] Remove CustomerId, CountryId, CustomerName headers (backend gets from JWT)
- [ ] Enable credentials:
  ```javascript
  axios.defaults.withCredentials = true;
  ```
- [ ] Cookies are sent automatically - no manual headers needed
- [ ] Backend extracts user info from JWT token

### 5. Update Redux Slices
- [ ] Update `AuthSlice.js`:
  - [ ] Keep user data fields (userId, email, firstName, lastName, role, etc.)
  - [ ] Remove any token fields
  - [ ] Store complete user object from login response
- [ ] Remove/update `UserManagerSlice.js`:
  - [ ] Remove `getAccessTokenAsync` (bearer token fetch - not needed)
  - [ ] Remove `getCustDetailsAsync` call from login (data comes from login response)
  - [ ] Keep for other features if needed
- [ ] Update `ThemeSlice.js`:
  - [ ] Theme data comes from login response
  - [ ] No separate API call needed
  - [ ] Just apply theme from login data

### 6. Handle Page Refresh / App Initialization
- [ ] In App.js or similar:
  ```javascript
  useEffect(() => {
    const checkAuth = async () => {
      try {
        const response = await api.get('/api/auth/me');
        // User is logged in, token auto-refreshed if needed
        dispatch(updateAuth(response.data.user));
      } catch (error) {
        // Not logged in or token expired
        // Redirect to login if on protected route
        if (window.location.pathname !== '/ats/login') {
          navigate('/ats/login');
        }
      }
    };

    checkAuth();
  }, []);
  ```
- [ ] Remove code that reads tokens from localStorage
- [ ] Remove token refresh logic (backend handles it)

### 7. Update AuthGuard (Route Protection)
- [ ] Update `AuthGuard.js`:
  - [ ] Remove localStorage token check
  - [ ] Check if user data exists in Redux
  - [ ] If no user data, call GET `/api/auth/me`
  - [ ] If 401, redirect to login
  - [ ] If 200, user is authenticated
  ```javascript
  const AuthGuard = ({ children }) => {
    const user = useSelector(state => state.Auth);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      if (!user.userId) {
        // Check session with backend
        api.get('/api/auth/me')
          .then(res => {
            dispatch(updateAuth(res.data.user));
            setLoading(false);
          })
          .catch(() => {
            navigate('/ats/login');
          });
      } else {
        setLoading(false);
      }
    }, []);

    if (loading) return <Loader />;
    return children;
  };
  ```

### 8. Update Logout
- [ ] Update logout function:
  ```javascript
  const handleLogout = async () => {
    try {
      await api.post('/api/auth/logout');

      // Clear Redux state
      dispatch(resetStore());

      // Clear localStorage (non-sensitive data)
      localStorage.clear();

      // Redirect to login
      navigate('/ats/login');

    } catch (error) {
      // Even if API fails, clear local state and redirect
      dispatch(resetStore());
      localStorage.clear();
      navigate('/ats/login');
    }
  };
  ```
- [ ] Remove manual cookie deletion (backend handles it)
- [ ] Remove AWS Cognito signOut call

### 9. Clean Up localStorage
- [ ] Remove all token storage code:
  - [ ] Remove `localStorage.setItem("accessToken", ...)`
  - [ ] Remove `localStorage.setItem("idToken", ...)`
  - [ ] Remove `localStorage.setItem("refreshToken", ...)`
  - [ ] Remove `localStorage.setItem("bearer_token", ...)`
  - [ ] Remove `localStorage.setItem("getAttributes", ...)`
- [ ] Can keep theme, preferences, etc. if needed (non-sensitive)

### 10. Update All API Calls
- [ ] Review all API calls across the app
- [ ] Ensure they use the configured axios instance with `withCredentials: true`
- [ ] Remove manual header injection (Authorization, CustomerId, etc.)
- [ ] Backend gets user context from JWT token automatically
- [ ] Examples:
  ```javascript
  // Before (manual headers)
  axios.get('/api/applicants', {
    headers: {
      Authorization: `Bearer ${token}`,
      CustomerId: userId,
      CountryId: countryId
    }
  });

  // After (cookies sent automatically)
  api.get('/api/applicants');
  // Backend gets userId, customerId, etc. from JWT token
  ```

### 11. Remove Token Refresh Logic
- [ ] Delete `refreshToken()` function from CognitoSlice
- [ ] Remove auto-refresh timers/intervals
- [ ] Remove token expiry checks
- [ ] Backend handles refresh automatically and transparently

### 12. Frontend Testing
- [ ] Test login with valid credentials
- [ ] Check browser DevTools → Cookies tab:
  - [ ] Should see `accessToken` cookie (httpOnly, secure)
  - [ ] Should see `refreshToken` cookie (httpOnly, secure)
- [ ] Test login with wrong password
- [ ] Test login with disabled account
- [ ] Test page refresh → User stays logged in
- [ ] Test opening in new tab → User logged in
- [ ] Wait 15-20 min → Make API call → Should work (auto-refresh)
- [ ] Test logout → Cookies cleared, redirected to login
- [ ] Test expired refresh token (wait 1 day or change backend expiry) → Auto logout
- [ ] Test all app features (applicants, jobs, dashboard, etc.)
- [ ] Verify NO tokens in localStorage
- [ ] Verify NO AWS Cognito calls in Network tab

---

## Key Differences from Session Approach

| Aspect | Session ID + Redis | JWT Token (Our Approach) |
|--------|-------------------|-------------------------|
| **Storage** | Redis database | No storage (stateless) |
| **Token Type** | Random session ID | JWT with user data inside |
| **Validation** | Lookup in Redis | Verify signature with secret key |
| **Scalability** | Need shared Redis for multiple servers | Works across multiple servers (no shared state) |
| **Logout** | Delete from Redis | Just clear cookies (token expires naturally) |
| **User Data** | Stored in Redis, fetched on each request | Embedded in JWT, decoded from token |
| **Refresh** | Update Redis session | Generate new JWT from refresh token |
| **Complexity** | Higher (need Redis setup) | Lower (just secret keys) |

---

## Important Notes

### Access Token Contents
The access token should contain all necessary user data so backend doesn't need to query database on every request:
```javascript
{
  userId: "cognito-user-id",
  email: "user@example.com",
  customerId: 123,
  companyName: "TechCorp",
  systemRole: "Admin",
  clientStatus: 1,
  projectStatus: 1,
  permissions: {
    canViewApplicants: true,
    canCreateJobs: true
  },
  exp: 1234567890 // Expiry timestamp
}
```

### Refresh Token Contents
The refresh token should be minimal (only identification):
```javascript
{
  userId: "cognito-user-id",
  email: "user@example.com",
  exp: 1234567890 // Expiry timestamp
}
```

### Cookie Security
Both cookies MUST have these flags:
- `httpOnly: true` - JavaScript cannot access (XSS protection)
- `secure: true` - Only sent over HTTPS
- `sameSite: 'strict'` - CSRF protection

### Token Refresh Flow
When access token expires:
1. Middleware detects expired access token
2. Checks refresh token from cookie
3. If refresh token valid, queries database for latest user data
4. Generates new access token with fresh data
5. Sets new access token cookie
6. Proceeds with request
7. Frontend never knows refresh happened

### Secret Keys
- Use long, random strings for JWT_ACCESS_SECRET and JWT_REFRESH_SECRET
- Never commit secrets to Git (use .env)
- Different secrets for access and refresh tokens (extra security)

---

## Environment Variables

**Backend (.env):**
```
# AWS Cognito
AWS_COGNITO_USER_POOL_ID=us-east-1_ABC123
AWS_COGNITO_CLIENT_ID=abcdefg123456
AWS_REGION=us-east-1

# JWT Secrets (generate random strings)
JWT_ACCESS_SECRET=super_secret_key_for_access_token_min_32_chars
JWT_REFRESH_SECRET=different_secret_for_refresh_token_min_32_chars

# Server
PORT=5000
NODE_ENV=production

# Database
DATABASE_URL=postgresql://user:pass@host:5432/dbname

# CORS
ALLOWED_ORIGINS=https://app.hirewing.ca,https://www.hirewing.ca
```

**Frontend (.env):**
```
REACT_APP_API_URL=https://api.hirewing.ca
```

---

## Timeline Estimate

**Backend Development:** 1.5 weeks
- JWT token service: 2 days
- AWS Cognito integration: 2 days
- Login endpoint + middleware: 2 days
- Testing: 2 days

**Frontend Development:** 1 week
- Remove Cognito SDK: 1 day
- Update login + axios config: 2 days
- Update all API calls: 2 days
- Testing: 2 days

**Integration & Testing:** 1 week
- End-to-end testing: 3 days
- Security testing: 2 days
- Bug fixes: 2 days

**Total: 3.5 - 4 weeks**

---

## Success Checklist

✅ **Backend:**
- [ ] JWT tokens generated correctly
- [ ] Cookies set with proper security flags
- [ ] Access token expires in 15-20 minutes
- [ ] Refresh token expires in 1 day
- [ ] Middleware auto-refreshes expired access tokens
- [ ] Protected routes require valid token
- [ ] CORS configured correctly
- [ ] Rate limiting works on login

✅ **Frontend:**
- [ ] No AWS Cognito SDK in package.json
- [ ] No tokens in localStorage
- [ ] Login calls backend API
- [ ] Cookies sent automatically with all requests
- [ ] Page refresh keeps user logged in
- [ ] Logout clears cookies and redirects
- [ ] 401 errors redirect to login

✅ **Security:**
- [ ] httpOnly cookies (check DevTools)
- [ ] secure flag enabled (HTTPS)
- [ ] sameSite: strict
- [ ] JWT verified with secret key
- [ ] Cannot decode JWT without secret
- [ ] XSS cannot steal cookies
- [ ] CSRF protection via sameSite

✅ **User Experience:**
- [ ] Login works same as before
- [ ] No visible changes to users
- [ ] Faster (1 API call instead of 5)
- [ ] Session persists across tabs
- [ ] Auto-refresh seamless (no logout)

---

## Deployment Checklist

- [ ] Generate strong JWT secrets (use crypto.randomBytes or similar)
- [ ] Set environment variables on production server
- [ ] Enable HTTPS (required for secure cookies)
- [ ] Configure CORS with production domains
- [ ] Test in staging environment first
- [ ] Monitor error logs for token issues
- [ ] Have rollback plan ready
- [ ] Update API documentation

---

**This approach is simpler than Redis sessions, fully stateless, and scales better across multiple servers.**
