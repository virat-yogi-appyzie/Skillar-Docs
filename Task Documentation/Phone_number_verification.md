# Mobile Number Verification Implementation Plan

## Overview

Add configurable authentication verification supporting both EMAIL and SMS OTP. New users will verify during signup (config-based), while existing users see an optional dashboard prompt. Using Firebase Phone Auth (free) initially, with easy migration path to MSG91.

## User Requirements

- **New Users**: Verify via SMS during signup only
- **Existing Users**: Optional dashboard prompt with incentive to add phone
- **Config-Driven**: Single config to switch between EMAIL/SMS verification
- **Provider**: Firebase Phone Auth (free tier), designed for easy MSG91 migration

## Current Codebase Analysis

**Authentication Structure:**

- Backend: NestJS with Prisma ORM (PostgreSQL)
- Frontend: Next.js with NextAuth
- Current verification: Email-based OTP system already implemented
- Key files: `auth.service.ts`, `auth.controller.ts`, User model in `schema.prisma`

**Existing OTP Infrastructure:**

- Email OTP via `OtpVerification` model
- 6-digit OTP with 5-minute expiry
- Cleanup cron job exists (`otpcleanup.cron.ts`)

## Architecture: Config-Driven Verification

### Central Configuration

Create `skiller-backend/src/config/auth.config.ts`:

```typescript
export const AUTH_CONFIG = {
  VERIFICATION_METHOD: process.env.VERIFICATION_METHOD || 'EMAIL', // 'EMAIL' | 'SMS'
  SMS_PROVIDER: process.env.SMS_PROVIDER || 'FIREBASE', // 'FIREBASE' | 'MSG91'
};
```

Switch EMAIL ↔ SMS by changing one env var, run both simultaneously for A/B testing.

## Database Schema Changes

Update `skiller-backend/prisma/schema.prisma`:

```prisma
model User {
  phoneNumber          String?
  countryCode          String?
  isPhoneVerified      Boolean      @default(false)
  phoneVerifications   PhoneVerification[]
  verificationMethod   String?      // 'EMAIL' | 'SMS'
}

model PhoneVerification {
  id          String   @id @default(uuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  phoneNumber String
  countryCode String   @default("+91")
  otp         String
  expiresAt   DateTime
  verified    Boolean  @default(false)
  provider    String   @default("FIREBASE")
  createdAt   DateTime @default(now())
  
  @@index([userId])
  @@index([otp])
  @@index([phoneNumber])
}
```

## Backend Implementation

### 1. Verification Service (`skiller-backend/src/verification/`)

Create abstraction layer with interface:

```typescript
export interface IVerificationProvider {
  sendOtp(identifier: string, otp: string): Promise<boolean>;
  validateFormat(identifier: string): boolean;
}
```

Factory pattern returns email or SMS service based on `AUTH_CONFIG.VERIFICATION_METHOD`.

### 2. SMS Service (`skiller-backend/src/sms/`)

**Files**: `sms.module.ts`, `sms.service.ts`, `providers/firebase-sms.provider.ts`, `providers/msg91-sms.provider.ts`

`SmsService` implements `IVerificationProvider`, delegates to Firebase or MSG91 based on config.

Firebase provider uses Admin SDK, MSG91 provider uses REST API (future).

### 3. Auth Service Updates

Add config-aware methods in `auth.service.ts`:

- `sendVerificationOtp(identifier)` - delegates to email or SMS
- `verifyOtp(identifier, otp)` - unified verification
- `sendPhoneOtp(phoneNumber)` - phone-specific
- `verifyPhoneOtp(phoneNumber, otp)` - phone-specific
- `validatePhoneNumber(phoneNumber)` - E.164 validation

### 4. Auth Controller Endpoints

New endpoints in `auth.controller.ts`:

- `POST /auth/send-verification-otp` - unified (email or SMS)
- `POST /auth/verify-verification-otp` - unified
- `POST /auth/send-phone-otp` - phone-specific
- `POST /auth/verify-phone-otp` - phone-specific
- `POST /auth/update-phone` - update user phone (authenticated)
- `GET /auth/phone-verification-status` - check status
- `GET /auth/verification-config` - return current method for frontend

### 5. Update Cron Job

Extend `otpcleanup.cron.ts` to cleanup both `OtpVerification` and `PhoneVerification` records.

## Frontend Implementation

### 1. Config Hook

Create `skiller-frontend/src/lib/useVerificationConfig.ts`:

```typescript
export function useVerificationConfig() {
  // Fetches config from backend
  return {
    isEmailVerification: config.method === 'EMAIL',
    isSmsVerification: config.method === 'SMS',
    verificationMethod: config.method,
  };
}
```

### 2. Server Actions

Create `skiller-frontend/src/actions/phone-verification/helper.ts`:

- `sendVerificationOtp(identifier)` - unified
- `verifyVerificationOtp(identifier, otp)` - unified
- `sendPhoneOtp(phoneNumber)` - phone-specific
- `verifyPhoneOtp(phoneNumber, otp)`
- `getPhoneVerificationStatus()`
- `updatePhoneNumber(phoneNumber)`

### 3. UI Components

Create in `skiller-frontend/src/components/auth/`:

**`PhoneInput.tsx`**: Country code selector + phone input with validation

**`OtpInput.tsx`**: Reusable 6-digit OTP input (works for email and SMS)

**`PhoneVerificationModal.tsx`**: Modal for phone verification flow (Enter phone → Enter OTP → Success)

**`PhoneVerificationPrompt.tsx`**: Dashboard banner for existing users with "Add Now" / "Remind Later" / "Don't show again"

### 4. Integration Points

**`SignupForm.tsx`**: Add conditional rendering based on `verificationMethod` config. Show phone input if SMS, email input if EMAIL.

**`dashboard/page.tsx`**: Check if user has verified phone, show `PhoneVerificationPrompt` if not (and not dismissed).

**`profile/page.tsx`**: Add phone number management section with edit/verify options, show verification badge.

## Environment Variables

### Backend `.env`

```bash
VERIFICATION_METHOD=EMAIL          # Switch to 'SMS' when ready
SMS_PROVIDER=FIREBASE              # or 'MSG91'

# Firebase (Current)
FIREBASE_PROJECT_ID=your_project_id
FIREBASE_PRIVATE_KEY=your_private_key
FIREBASE_CLIENT_EMAIL=your_client_email

# MSG91 (Future)
MSG91_AUTH_KEY=your_auth_key
MSG91_SENDER_ID=your_sender_id
```

### Frontend `.env.local`

```bash
NEXT_PUBLIC_PHONE_VERIFICATION_ENABLED=true
NEXT_PUBLIC_FIREBASE_API_KEY=your_api_key
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your_project_id
```

## Firebase Setup

1. Create Firebase project, enable Phone Authentication
2. Generate service account key (Admin SDK)
3. Initialize Firebase Admin in backend
4. Initialize Firebase client in frontend
5. reCAPTCHA auto-configured by Firebase

## Migration Strategy

**Phase 1 (Current)**: VERIFICATION_METHOD=EMAIL, build SMS in parallel

**Phase 2**: Enable dashboard prompt for existing users (optional phone)

**Phase 3**: VERIFICATION_METHOD=SMS for new signups

**Phase 4**: Switch SMS_PROVIDER=MSG91 (no code changes, just config)

## MSG91 Migration (Future)

### Why Later?

- Cost: ~₹0.15/SMS vs Firebase free tier
- Better for high volume (>10K/month)

### How to Migrate?

1. Add MSG91 credentials to env
2. Change `SMS_PROVIDER=MSG91`
3. Deploy backend
4. Zero code changes (thanks to abstraction layer)

## Security

1. **Rate Limiting**: Max 3 OTP/phone/hour, max 5 attempts/OTP
2. **Validation**: E.164 format, duplicate phone check
3. **Firebase**: Built-in reCAPTCHA, app verification
4. **Privacy**: Mask phone numbers in logs

## Testing

- Unit: Provider switching, validation, OTP logic
- Integration: Firebase mock, signup flow, config switching
- E2E: SMS signup, dashboard prompt, profile management, rate limiting

## Cost Analysis

**Firebase**: 10K verifications/month free, then $0.01 each

**MSG91**: ₹0.15-0.25 per SMS

**Switch when**: Crossing 10K/month or need <₹0.20 pricing

## Rollback

Change `VERIFICATION_METHOD=EMAIL` → restart → back to email OTP. No code deployment needed.

## Development Phases

**Week 1**: Schema migration, Firebase setup, SMS service module, auth config

**Week 2**: Auth endpoints, frontend hook, PhoneInput, PhoneVerificationModal, server actions

**Week 3**: SignupForm integration, dashboard prompt, profile page, testing

**Future**: MSG91 provider implementation (just new class, plug into existing interface)