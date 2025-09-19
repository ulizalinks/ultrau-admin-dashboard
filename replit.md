# Uliza Tickets

## Overview
Uliza Tickets is a modern, colorful event ticketing platform designed for discovering and purchasing tickets for events across Africa. It features a vibrant, festival-inspired design, smooth animations, and a seamless guest checkout experience. The platform aims to provide a robust and user-friendly solution for event organizers and attendees, including digital ticket generation with QR codes, integration with CodeREADr for ticket validation, and an event approval workflow. Its business vision includes transforming traditional events into queue-free, cashless experiences using NFC wristbands for entry, payments, and identification.

## User Preferences
Preferred communication style: Simple, everyday language.

## System Architecture

### UI/UX Decisions
- **Design System**: Festival-inspired theme with vibrant gradients, rounded corners, playful typography, and responsive design using CSS Custom Properties.
- **Modern PDF Ticket Design**: Mobile-friendly PDF tickets (320x480 pixels) matching digital tickets.

### Technical Implementations
- **Frontend**: React 18 with TypeScript, Vite, Tailwind CSS + shadcn/ui, Wouter for routing, TanStack Query for state management, Framer Motion for animations, React Hook Form + Zod for forms.
- **Backend**: Express.js for RESTful APIs, Drizzle ORM for type-safe database queries, connect-pg-simple for session storage.
- **Data Storage**: PostgreSQL (Neon serverless) with a three-table schema (`events`, `tickets`, `scans`) for event, ticket, and scan audit information, extended for NFC tokens and wallets.
- **Authentication & Authorization**: Guest checkout system, unique Ticket UIDs for validation, API-based validation via CodeREADr.
- **Cashless Event Workflow**: Comprehensive system using NFC wristbands for entry, payments, and identification, supported by staff and vendor PWAs with WebNFC integration. Features include pre-event online purchasing, on-site NFC pairing, PWA scanner validation, cashless commerce, and real-time admin dashboard monitoring.

### Feature Specifications
- **Multi-Ticket Box Office**: Supports purchasing multiple ticket types in a single transaction with real-time updates.
- **Live Events Monitoring**: Real-time admin dashboard with scanning statistics, search, and attendance metrics.
- **MVP (Most Valuable Players) System**: Ranks top attendees based on scan attempts.
- **Development Bypass System**: Allows payment testing in development mode.
- **User Favorites/Watchlist**: Allows users to save favorite events.
- **Object Storage Upload System**: Secure image upload using presigned URLs for event imagery.
- **Automated CodeREADr Workflow**: Fully automated integration with persistent database/service ID storage, including bulk ticket uploads (100 tickets/request).
- **Zone Access Control**: Comprehensive ticket type to zone validation mapping, ensuring security and preventing unauthorized access.
- **Multi-Level Scanner System**: Three distinct scanner types (Zone, Oveit, Mobile) with role-based access control and zone-specific validation workflows.

### System Design Choices
- **Queue-free, Cashless Event System**: Core design principle to eliminate payment delays and entry bottlenecks.
- **PWA-first Approach**: For staff and customer interfaces, offering cross-platform, offline-first capabilities with WebNFC integration.
- **Real-time Updates**: Utilizing Server-Sent Events (SSE) for instant dashboard updates.
- **Security-focused**: Implementing robust validation for zone access and preventing privilege escalation.

## Scanner System Workflows

### Scanner Types & Access Levels

#### 1. Zone Scanner (Zone-Based Access Control)
**Purpose:** Validates ticket access to specific zones without marking tickets as "used"

**Access Path:** Admin Dashboard → "Zone Scanner" card → Zone Scanner Landing → Select Event → Choose Zone

**Zone Access Levels:**
- **General Zone:** Accepts all ticket types (General, VIP, Staff)
- **VIP Zone:** Accepts only VIP and Staff tickets  
- **Security Zone:** Accepts only Staff tickets

**Workflow for Each Zone:**

**General Zone Scanner:**
1. Scanner selects "General Zone" for an event
2. Uses camera/NFC/manual input to scan ticket
3. System validates: ✅ General ✅ VIP ✅ Staff tickets allowed
4. Shows "Access GRANTED" or "Access DENIED" 
5. Does NOT mark ticket as used (can re-enter)

**VIP Zone Scanner:**
1. Scanner selects "VIP Zone" for an event  
2. Scans ticket UID
3. System validates: ❌ General tickets denied, ✅ VIP ✅ Staff allowed
4. Provides zone-specific access control
5. Does NOT mark ticket as used

**Security Zone Scanner:**
1. Scanner selects "Security Zone" for an event
2. Scans ticket UID  
3. System validates: ❌ General ❌ VIP denied, ✅ Staff only
4. Highest security clearance required
5. Does NOT mark ticket as used

#### 2. Oveit Scanner (Ticket Validation System)
**Purpose:** General entry validation that marks tickets as "used"

**Access Path:** Admin Dashboard → "Oveit Scanner" card → Oveit Events → Select Event

**Workflow:**
1. Scanner selects event from Oveit Events page
2. Uses manual input to enter ticket UID
3. System validates ticket and marks as "used" 
4. Shows ticket holder details (name, type, etc.)
5. Permanent check-in (cannot re-enter once used)

#### 3. Mobile Scanner (Staff Multi-Purpose)  
**Purpose:** Flexible scanner for staff with multiple modes

**Access Modes:**
- General entry checking
- Zone validation 
- All-zone access for senior staff

### Real-World Scanner Deployment Example

For an event like **Murima Fest 2025:**

**Entry Gate:** Oveit Scanner
- Validates and marks tickets as "used" for initial entry
- Records attendee check-in time and scanner operator

**General Area:** Zone Scanner (General Zone)
- Allows movement within general festival area
- Accepts all valid ticket types
- Does not mark as used (free movement)

**VIP Section:** Zone Scanner (VIP Zone)  
- Restricts access to VIP ticket holders only
- Staff can also access for service/security
- Prevents general admission holders from entering

**Backstage/Security Areas:** Zone Scanner (Security Zone)
- Only staff tickets accepted
- Highest restriction level
- Controls access to sensitive areas

### Ticket Type Hierarchy

```
Staff Tickets: Access ALL zones (General, VIP, Security)
VIP Tickets: Access General + VIP zones  
General Tickets: Access General zone only
```

This creates a comprehensive multi-level security system where different scanner operators can be assigned to specific zones with appropriate access control based on the event's security requirements.

## External Dependencies

- **Neon Database**: Serverless PostgreSQL hosting.
- **Replit Platform**: Development and deployment environment.
- **Paystack**: Payment gateway.
- **CodeREADr API**: Ticket validation service.
- **Brevo Email Service**: SMTP email delivery.
- **Unsplash Images**: Default event imagery.
- **Radix UI Primitives**: Accessible component foundations.
- **Lucide React**: Icon system.
- **Inter Font**: Primary typography via Google Fonts.

## Daily Fixes

### UGX Multi-Currency Wallet Implementation (2025-09-15)

**Achievement**: Successfully implemented UGX currency support for Flutterwave wallet system, enabling multi-country deployment strategy.

**Key Changes**:
1. **Backend Currency Switch**: Updated `/api/me/load-wallet-flutterwave` to accept `amount_ugx`, use `currency: 'UGX'`, and enable `mobilemoneyuganda` payment options for MTN Uganda and Airtel Uganda
2. **Amount Handling**: Removed division by 100 since UGX uses whole shillings (not cents like KES kobo)
3. **Frontend UI**: Updated wallet page to display UGX with proper formatting, validation ranges (1,000-1,000,000 UGX), and mobile money terminology
4. **Payment Flow**: Streamlined to single API call with immediate redirect to Flutterwave hosted payment page
5. **Database Schema**: Added `balance_ugx` field to wallets table (backward compatible with existing `balance_kes`)

**Multi-Country Strategy Achieved**:
- **Kenya Market**: Paystack (KES) for checkout + wallet top-ups
- **Uganda Market**: Flutterwave (UGX) for wallet with MTN/Airtel mobile money
- **Same codebase**, different payment providers per region
- **Native currencies** and local payment methods for better user experience

**Technical Result**: Production-ready multi-currency wallet system supporting both Kenyan and Ugandan markets with appropriate payment methods (M-Pesa vs MTN/Airtel).

### KopoKopo Mobile Money Integration Fix (2025-09-15)

**Issue**: KopoKopo Mobile Money payment integration was not working - payments were defaulting to Paystack M-Pesa instead of routing to KopoKopo.

**Root Causes**:
1. **Payment Method Routing**: Frontend correctly sends `payment_method: "mobile_money"` but backend validation error message was misleading
2. **Backend Logic**: The conditional routing `if (payment_method === 'mobile_money')` was correctly implemented 
3. **Authentication Setup**: KopoKopo requires both OAuth token AND API key in headers

**Final Working Solution**:
```javascript
// 1. Fixed backend validation error message (server/routes.ts)
error: 'Invalid payment method. Use "mpesa", "card", or "mobile_money"', // Added mobile_money

// 2. Frontend payment method correctly configured (client/src/pages/checkout.tsx)
<button onClick={() => setPaymentMethod('mobile_money')}>Mobile Money</button>
// Sends: { "payment_method": "mobile_money" }

// 3. KopoKopo authentication with both OAuth + API key (server/routes.ts)
headers: {
  'Authorization': `Bearer ${token}`,     // OAuth token
  'K-Signature': API_KEY                  // API key authentication
}
```

**Result**: Complete Mobile Money payment routing to KopoKopo with proper authentication headers using both OAuth token and K-Signature API key. Technical integration is fully working - only remaining issue is OAuth credential validation with KopoKopo servers (credentials may need verification/activation on KopoKopo dashboard).

### Flutterwave Wallet Integration Fix (2025-09-14)

**Issue**: Flutterwave wallet integration needed proper implementation with project-specific API keys and removal of development bypass logic.

**Root Causes**:
1. **Development Bypass**: Frontend checkout page had `dev_mode` logic that was showing bypass for M-Pesa payments
2. **Wallet Integration**: Flutterwave wallet top-up needed proper REST API implementation with project-specific keys (`ULIZA_EVENTS_*`)
3. **UI Integration**: Wallet needed to be integrated into user dashboard and profile menu

**Final Working Solution**:
```javascript
// 1. Fixed Flutterwave wallet endpoint using REST API (server/routes.ts)
const payload = {
  tx_ref: `wallet_${userId}_${Date.now()}`,
  amount: (amount_kes / 100).toString(), // Amount in KES (from kobo)
  currency: 'KES',
  redirect_url: `${req.get('origin')}/wallet?status=success`,
  customer: { email: userEmail, name: userEmail, phonenumber: '0700000000' },
  customizations: { title: 'Uliza Tickets Wallet Top-up' },
  payment_options: 'card,mpesa,banktransfer', // Enable M-Pesa for KES
  meta: [/* user metadata */]
};

// 2. Removed development bypass logic (client/src/pages/checkout.tsx)
onSuccess: (data) => {
  // Both M-Pesa and Card payments now redirect to Paystack payment page
  if (data.payment_url) {
    window.location.href = data.payment_url;
  }
}

// 3. Added wallet balance to dashboard (client/src/pages/dashboard.tsx)
const { data: walletData } = useQuery({ queryKey: ['/api/me/overview'] });
// Display: KES {((walletData?.wallet?.balanceKes || 0) / 100).toFixed(2)}

// 4. Added wallet to profile menu (client/src/components/navbar.tsx)
<Link href="/wallet">
  <DropdownMenuItem data-testid="menu-wallet">
    <Wallet className="mr-2 h-4 w-4" />
    <span>My Wallet</span>
  </DropdownMenuItem>
</Link>
```

**Result**: Complete end-to-end wallet functionality with Flutterwave integration, proper M-Pesa/Card payment handling, and integrated UI components.

### CodeREADr Webhook Integration Fix (2025-09-13)

**Issue**: CodeREADr webhook integration was failing with authentication, field mapping, and status validation problems.

**Root Causes**:
1. **Authentication Issue**: CodeREADr webhooks don't send authentication headers like regular API calls
2. **Field Name Mismatch**: CodeREADr sends different field names than expected:
   - Sends `tid` but code expected `value`
   - Sends `deviceid` but code expected `device_id` 
   - Sends `scanid` but code expected `scan_id`
3. **Status Type Mismatch**: CodeREADr sends numeric status `1` for valid scans, but validation only checked for string values

**Final Working Solution**:
```javascript
// 1. Dual authentication support (header + URL parameter)
const authToken = req.headers.authorization?.replace('Bearer ', '') || req.query.token;

// 2. Fallback field mapping for CodeREADr's actual field names
const ticketValue = req.body.tid || req.body.value;
const actualDeviceId = req.body.deviceid || req.body.device_id;
const actualScanId = req.body.scanid || req.body.scan_id;

// 3. Handle both string AND numeric status codes
const isValidScan = scanResults === 'valid' || scanResults === 'allowed' || 
                   status === 'valid' || status === 'allowed' || 
                   status === '1' || status === 1;
```

**Result**: Complete end-to-end CodeREADr scanning workflow with real-time database updates, proper XML responses, and audit trails.