# FedChain Enhanced - Feature Updates & Backend Integration Guide

## 🔄 Major Enhancements Implemented

### 1. **Real OTP Display (SMS & Email)**
- OTP codes now display in a prominent **highlighted box** that's easy to see
- When users resend codes, the **new code is immediately displayed** in full
- Demo testing shows codes clearly for non-production environments
- Ready for real SMS/Email provider integration

**Frontend Changes:**
- New `.otp-code-display` CSS class with highlighted styling
- Resend functions now show the generated code in the UI
- Clear visual feedback when codes are sent or resent

**Backend Integration Points:**

```javascript
// SMS Integration (e.g., Twilio, Vonage, Termii, Infobip)
async function sendSMSOTP(phoneNumber, formattedPhone) {
  const otp = genOTP();
  
  try {
    // Example: Twilio
    const response = await fetch('/api/send-sms', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        to: formattedPhone,
        message: `Your FedChain verification code is: ${otp}`
      })
    });
    
    return { success: true, otp, reference: response.data.sid };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

// Email Integration (e.g., SendGrid, Mailgun, AWS SES)
async function sendEmailOTP(emailAddress, otp) {
  try {
    const response = await fetch('/api/send-email', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        to: emailAddress,
        subject: 'FedChain Verification Code',
        template: 'otp-email',
        variables: {
          code: otp,
          expiry: 10 // minutes
        }
      })
    });
    
    return { success: true, messageId: response.data.id };
  } catch (error) {
    return { success: false, error: error.message };
  }
}
```

---

### 2. **Multi-Platform Transfer System**
Users can now send money to 5 different destinations with platform selection UI:

#### Supported Platforms:
1. **FedChain User** - Native wallet transfer
2. **Opay** - Mobile money platform
3. **Palmpay** - Digital financial service
4. **Moniepoint** - Mobile money & fintech
5. **Kuda** - Mobile banking app

**Frontend Implementation:**
- Platform selection grid with clear icons and labels
- Dynamic UI labels based on selected platform
- Recipient field placeholders adapted per platform
- Status tracking includes platform type

**Backend Integration Points:**

```javascript
// Platform API Structure
const platformIntegrations = {
  
  // FedChain internal transfer
  fedchain: {
    validateAccount: (accountNumber) => {
      // Check if account exists in FedChain system
      return getUser(accountNumber) !== null;
    },
    
    transfer: async (fromPhone, toPhone, amount, ref) => {
      // Internal ledger transaction
      const from = getUser(fromPhone);
      const to = getUser(toPhone);
      // Update both accounts and create records
    }
  },

  // Third-party platform transfers
  opay: {
    validateAccount: async (accountNumber) => {
      const response = await fetch('https://api.opay.ng/validate-account', {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${OPAY_API_KEY}` },
        body: JSON.stringify({ accountNumber })
      });
      return response.data.valid;
    },
    
    transfer: async (fromPhone, toAcct, amount, ref) => {
      const from = getUser(fromPhone);
      
      const response = await fetch('https://api.opay.ng/transfer', {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${OPAY_API_KEY}` },
        body: JSON.stringify({
          amount,
          accountNumber: toAcct,
          reference: ref,
          narration: `Transfer from FedChain user`
        })
      });
      
      if (response.success) {
        from.balance -= (amount + TRANSACTION_FEE);
        addTxDirect(from, {
          type: 'debit',
          platform: 'opay',
          amount: -amount,
          fee: TRANSACTION_FEE,
          reference: ref
        });
        return { success: true, transactionId: response.transactionId };
      }
      
      return { success: false, error: response.error };
    }
  },

  // Similar structure for palmpay, moniepoint, kuda
  palmpay: {
    validateAccount: async (accountNumber) => { /* ... */ },
    transfer: async (fromPhone, toAcct, amount, ref) => { /* ... */ }
  },

  moniepoint: {
    validateAccount: async (accountNumber) => { /* ... */ },
    transfer: async (fromPhone, toAcct, amount, ref) => { /* ... */ }
  },

  kuda: {
    validateAccount: async (accountNumber) => { /* ... */ },
    transfer: async (fromPhone, toAcct, amount, ref) => { /* ... */ }
  }
};
```

---

### 3. **Transaction Fees (₦30)**
Every transfer now deducts **₦30 as a transaction fee** from the sender's account.

**Implementation Details:**
- Constant: `TRANSACTION_FEE = 30`
- Displayed in real-time fee calculator
- Total debit shown before confirmation
- Minimum transfer amount updated to account for fee
- Fee tracking in transaction records

**Fee Display UI:**
- Amount breakdown: Original amount + ₦30 fee = Total debit
- Dynamically updates as user enters amount
- Error message if insufficient balance for amount + fee
- Transaction history shows fee deduction

**Backend Fee Distribution (Example):**
```javascript
// After transaction deduction
const feeDistribution = {
  platformFee: TRANSACTION_FEE * 0.5,        // ₦15 to payment processor
  fedchainRevenue: TRANSACTION_FEE * 0.3,    // ₦9 to FedChain
  merchantAcquiring: TRANSACTION_FEE * 0.2,  // ₦6 to acquirer/partner
};

// Track in separate ledger
async function recordFeeTransaction(reference, amount, distribution) {
  await db.feeTransactions.insert({
    reference,
    totalFee: amount,
    breakdown: distribution,
    timestamp: Date.now(),
    status: 'settled'
  });
}
```

---

### 4. **Enhanced OTP Code Display on Resend**
When users click "Resend SMS" or "Resend Email":
- ✅ New OTP is generated
- ✅ Code displays **in full** in a highlighted box
- ✅ Clear visual feedback message
- ✅ User can immediately see the new code for testing

**Code Examples:**

```javascript
// SMS Resend
function resendSMSOTP(){
  pendingPhoneOTP = genOTP();
  
  // Display code prominently
  $('demo_sms_code').textContent = pendingPhoneOTP;
  
  clearOTP('p');
  $('p0').focus();
  
  setMsg('otp_msg', 'New SMS code sent - check message', 'success');
  
  // Toast notification with code visible
  toast('New SMS code: '+pendingPhoneOTP, 'success');
}

// Email Resend  
function sendEmailOTP(resend=false){
  // ... validation ...
  
  pendingEmailOTP = genOTP();
  
  $('kyc_email_show').textContent = email;
  $('demo_email_code').textContent = pendingEmailOTP;  // Show in box
  
  clearOTP('e');
  
  // ... show step 2 ...
  
  if(resend){
    setMsg('kyc_msg2', 'New OTP: '+pendingEmailOTP+' (check email)', 'success');
  }
}
```

---

## 🚀 Backend Architecture Requirements

### 1. **API Endpoints Needed**

```
POST /api/auth/send-sms-otp
- Request: { phone: "08012345678" }
- Response: { success: true, reference: "SMS_REF_123" }

POST /api/auth/send-email-otp
- Request: { email: "user@example.com" }
- Response: { success: true, messageId: "MSG_ID_456" }

POST /api/auth/verify-sms-otp
- Request: { phone: "08012345678", otp: "123456" }
- Response: { success: true, token: "JWT_TOKEN" }

POST /api/auth/verify-email-otp
- Request: { email: "user@example.com", otp: "123456" }
- Response: { success: true, verified: true }

POST /api/transfer/validate-account
- Request: { accountNumber: "08012345678", platform: "opay" }
- Response: { valid: true, name: "John Doe" }

POST /api/transfer/initiate
- Request: { 
    from: "08012345678",
    to: "08087654321",
    amount: 5000,
    platform: "fedchain|opay|palmpay|moniepoint|kuda",
    reference: "FC_ABC123XYZ"
  }
- Response: { success: true, transactionId: "TXN_123456" }

GET /api/transfer/status/:transactionId
- Response: { status: "completed|pending|failed", ... }

POST /api/accounts/balance
- Request: { phone: "08012345678" }
- Response: { balance: 5000, lastUpdated: "2024-01-15T10:30:00Z" }
```

### 2. **Database Schema**

```sql
-- Users/Accounts
CREATE TABLE users (
  id UUID PRIMARY KEY,
  phone VARCHAR(11) UNIQUE,
  email VARCHAR(255),
  name VARCHAR(255),
  password_hash VARCHAR(255),
  balance DECIMAL(10,2),
  kyc_verified BOOLEAN DEFAULT false,
  frozen BOOLEAN DEFAULT false,
  ref_code VARCHAR(8) UNIQUE,
  referred_by VARCHAR(8),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- Transactions
CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  reference VARCHAR(20) UNIQUE,
  from_phone VARCHAR(11),
  to_identifier VARCHAR(255), -- Can be phone or external account
  amount DECIMAL(10,2),
  fee DECIMAL(10,2),
  platform VARCHAR(50), -- 'fedchain', 'opay', 'palmpay', etc
  type VARCHAR(20), -- 'credit', 'debit', 'kyc'
  status VARCHAR(20), -- 'completed', 'pending', 'failed'
  description TEXT,
  created_at TIMESTAMP,
  FOREIGN KEY (from_phone) REFERENCES users(phone)
);

-- Fee Ledger
CREATE TABLE fee_transactions (
  id UUID PRIMARY KEY,
  transaction_reference VARCHAR(20),
  total_fee DECIMAL(10,2),
  platform_fee DECIMAL(10,2),
  fedchain_revenue DECIMAL(10,2),
  merchant_acquiring DECIMAL(10,2),
  settled_at TIMESTAMP,
  FOREIGN KEY (transaction_reference) REFERENCES transactions(reference)
);

-- OTP Records (for audit/security)
CREATE TABLE otp_records (
  id UUID PRIMARY KEY,
  phone_or_email VARCHAR(255),
  otp_hash VARCHAR(255),
  type VARCHAR(10), -- 'sms', 'email'
  attempts INT DEFAULT 0,
  max_attempts INT DEFAULT 3,
  expires_at TIMESTAMP,
  verified_at TIMESTAMP,
  created_at TIMESTAMP
);

-- Referrals
CREATE TABLE referrals (
  id UUID PRIMARY KEY,
  referrer_phone VARCHAR(11),
  referred_phone VARCHAR(11),
  bonus_amount DECIMAL(10,2),
  status VARCHAR(20), -- 'pending', 'completed'
  created_at TIMESTAMP,
  FOREIGN KEY (referrer_phone) REFERENCES users(phone),
  FOREIGN KEY (referred_phone) REFERENCES users(phone)
);
```

### 3. **Security Best Practices**

```javascript
// OTP Security
const OTP_EXPIRY = 10 * 60 * 1000; // 10 minutes
const MAX_OTP_ATTEMPTS = 3;
const OTP_COOLDOWN = 60 * 1000; // 60 seconds between resends

// Rate Limiting
const RATE_LIMITS = {
  sms_otp: { max: 5, window: 3600000 }, // 5 attempts per hour
  email_otp: { max: 5, window: 3600000 },
  transfer: { max: 50, window: 3600000 }, // 50 transfers per hour
  login: { max: 10, window: 900000 } // 10 attempts per 15 mins
};

// Encryption
const encryptOTP = (otp) => {
  return bcrypt.hashSync(otp, 10);
};

const verifyOTP = (inputOTP, hashedOTP) => {
  return bcrypt.compareSync(inputOTP, hashedOTP);
};

// Transaction Verification
const validateTransaction = (tx) => {
  const validations = [
    tx.amount > 0,
    tx.amount <= MAX_TRANSFER_AMOUNT,
    tx.from !== tx.to || tx.platform !== 'fedchain',
    tx.fee === TRANSACTION_FEE,
    validatePhoneFormat(tx.from)
  ];
  
  return validations.every(v => v === true);
};
```

### 4. **Platform-Specific Integration Examples**

#### **Opay Integration**
```javascript
const opayConfig = {
  baseURL: 'https://api.opay.ng/v2',
  merchantId: process.env.OPAY_MERCHANT_ID,
  apiKey: process.env.OPAY_API_KEY,
  
  async transfer(toAccount, amount, reference) {
    const payload = {
      businessId: this.merchantId,
      txRef: reference,
      amount: amount * 100, // Convert to kobo
      accountNumber: toAccount,
      accountName: 'Recipient', // Fetch from validation
      narration: 'FedChain Transfer'
    };
    
    const signature = generateSignature(payload, this.apiKey);
    
    const response = await axios.post(
      `${this.baseURL}/transfers/create`,
      payload,
      { headers: { 'Authorization': signature } }
    );
    
    return response.data;
  }
};
```

#### **Kuda Integration**
```javascript
const kudaConfig = {
  baseURL: 'https://api.kuda.com/v1',
  clientId: process.env.KUDA_CLIENT_ID,
  clientSecret: process.env.KUDA_CLIENT_SECRET,
  
  async getAccessToken() {
    const response = await axios.post(
      `${this.baseURL}/authorization/oauth/token`,
      {
        grant_type: 'client_credentials',
        client_id: this.clientId,
        client_secret: this.clientSecret,
        scope: 'transfers'
      }
    );
    
    return response.data.access_token;
  },
  
  async transfer(toAccount, amount, reference) {
    const token = await this.getAccessToken();
    
    const response = await axios.post(
      `${this.baseURL}/transfers`,
      {
        destinationAccountNumber: toAccount,
        amount: amount,
        narration: 'FedChain Transfer',
        reference: reference
      },
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    
    return response.data;
  }
};
```

---

## 📊 Testing Checklist

- [ ] **OTP Display**
  - [ ] SMS OTP displays in highlighted box
  - [ ] Email OTP displays in highlighted box
  - [ ] Resend shows new code immediately
  - [ ] Code is fully visible for testing

- [ ] **Multi-Platform Transfer**
  - [ ] Platform selection UI works
  - [ ] FedChain user transfer completes
  - [ ] External platform transfers track correctly
  - [ ] Recipient lookup works per platform

- [ ] **Transaction Fees**
  - [ ] ₦30 fee deducted from every transfer
  - [ ] Fee shows in transaction summary
  - [ ] Insufficient balance error includes fee
  - [ ] Fee appears in transaction history
  - [ ] Fee breakdown recorded in ledger

- [ ] **Security**
  - [ ] OTP expires after 10 minutes
  - [ ] Max 3 OTP attempts enforced
  - [ ] Rate limiting working
  - [ ] Transactions validated server-side

---

## 🔌 Migration from Demo to Production

1. **Remove localStorage, implement real database**
2. **Enable real SMS/Email OTP providers**
3. **Configure platform API credentials securely**
4. **Implement PCI compliance for card storage**
5. **Add comprehensive audit logging**
6. **Set up transaction reconciliation**
7. **Deploy fee settlement processes**
8. **Enable webhook handlers for platform callbacks**

---

## 📝 Notes

- All OTP codes are **displayed prominently** for demo/testing
- Transaction fees are **₦30 flat rate** (customizable in code)
- Platform selection is **UI-ready** and API-ready for backend
- Code includes **detailed comments** for all integration points
- Error handling is **comprehensive** across all flows
