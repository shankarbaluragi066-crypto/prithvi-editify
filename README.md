# Prithvi Editify — Freelance Creative Studio & Client CRM

Full-stack freelance creative studio website, client portal, project management, and INR invoicing CRM for **Prithvi Editify** (Owner: **Shankar Baluragi**).

Built with **React 19, TypeScript, Vite, Tailwind CSS, React Hook Form, Zod, Firebase Authentication, Cloud Firestore (NoSQL), Firebase Cloud Storage, and Firebase Cloud Functions**.

---

## 1. Architecture & Firestore NoSQL Data Model

All collections are defined in `firebase-blueprint.json` and protected by default-deny rules in `firestore.rules`:

| Collection Path | Purpose | Ownership & Access Rules |
| :--- | :--- | :--- |
| `/users/{userId}` | Authenticated user profile & role (`admin` \| `client`) | Owner can read/update own contact fields; self-assigned `admin` role is blocked. |
| `/admins/{adminId}` | Trusted admin UID registry | Verified studio owner (`shankarbaluragi066@gmail.com`) or existing admin only. |
| `/clients/{clientId}` | Client CRM records & private `adminNotes` | Admin full CRUD; Client can read own record where `userId == request.auth.uid`. |
| `/services/{serviceId}` | Studio services & INR starting prices | Public read where `published == true`; Admin full CRUD. |
| `/portfolio/{itemId}` | Creative portfolio showcase | Public read where `published == true`; Admin full CRUD. |
| `/inquiries/{inquiryId}` | Public contact/lead briefs | Visitors can `create` validated records with `status == 'New'`; Admin read/update/delete only. |
| `/projects/{projectId}` | Client projects, progress, deadlines | Admin full CRUD; Client can read own projects (`clientUserId == request.auth.uid`) and submit revision/approval status updates on non-terminal projects. |
| `/projectInternalNotes/{projectId}` | Private admin-only project notes | Strictly isolated to Admin only (clients can never read). |
| `/tasks/{taskId}` | Project milestones & tasks | Admin full CRUD; Client can read where `clientUserId == request.auth.uid && visibility == 'client'`. |
| `/files/{fileId}` | Deliverables, previews & references | Admin full CRUD; Client can read where `clientUserId == request.auth.uid && visibility == 'client'`. |
| `/invoices/{invoiceId}` | INR invoices & `/lineItems/{itemId}` | Admin full CRUD; Client can read non-draft invoices where `clientUserId == request.auth.uid`. Clients can never modify totals or status. |
| `/payments/{paymentId}` | Recorded INR payments | Admin full CRUD; Client can read own payment records. |
| `/projectMessages/{messageId}` | Project comments & feedback | Admin full CRUD; Client can read/post where `clientUserId == request.auth.uid && visibility == 'client'`. |
| `/notifications/{notificationId}` | In-app notifications | Recipient can read and mark `read: true`. |
| `/settings/public` | Public studio identity & WhatsApp config | Public read (`get`); Admin write. |
| `/settings/private` | Private invoice prefix, tax & payment instructions | Admin read/write only. |
| `/activityLogs/{logId}` | Immutable admin audit trail | Admin read/create only; updates and deletes are forbidden (`false`). |

---

## 2. Secure Initial Admin Setup Instructions

1. **Automatic Owner Bootstrap**:
   - The Firestore & Storage rules (`firestore.rules` and `storage.rules`) and `AuthContext.tsx` recognize **`shankarbaluragi066@gmail.com`** (when `email_verified == true`) as the Studio Administrator.
   - Sign in on `/login` using **"Continue with Google"** with `shankarbaluragi066@gmail.com`. You will automatically be routed to `/admin`.
2. **Adding Additional Admins via Firebase Admin SDK**:
   - Create a document at `/admins/{targetUserUid}` in Firestore from the Admin Console or via Firebase Admin SDK in a trusted server environment.
   - Self-registration on `/login` strictly assigns `role: "client"` and is enforced by `firestore.rules`.

---

## 3. Demo Seed Data Instructions

1. Sign in as the Administrator (`shankarbaluragi066@gmail.com`) and open `/admin`.
2. Click **"Sync Default Services & Demo Portfolio"** in the top-right corner of the Admin Dashboard (or in `/admin/services`).
3. This populates Firestore with:
   - All 10 starting-price INR services (Social Media Post Design ₹500/design, Promotional Ad Creative ₹650/design, Banner/Digital Flyer ₹500/design, Marketing Graphic ₹400/design, Campaign Creative ₹450/design, Brand-Focused Digital Content ₹750/design, YouTube Thumbnail ₹450–₹700/design, YouTube Video Editing Custom quote, Instagram Reels Editing Custom quote, Other creative services Custom quote).
   - 6 demonstration portfolio entries explicitly flagged with `isSampleContent: true` ("Demonstration Sample").
   - Default `/settings/public` and `/settings/private` documents.

---

## 4. Local Development & Testing Instructions

```bash
# Install dependencies
npm install

# Run automated calculation & security specification tests
npx tsx src/utils/crmTests.ts

# Lint Firestore security rules
npx eslint firestore.rules

# Start development server (port 3000)
npm run dev

# Build production bundle
npm run build
```

---

## 5. Firebase Deployment Instructions

```bash
# 1. Authenticate with Firebase CLI
firebase login

# 2. Select your provisioned Firebase project
firebase use oval-cedar-qw2pc

# 3. Build the production frontend
npm run build

# 4. Deploy Firestore rules, indexes, Storage rules, Cloud Functions, and Hosting
firebase deploy
```

---

## 6. Production Security Checklist & Firebase Limitations

- [x] Default-deny catch-all in both `firestore.rules` and `storage.rules`.
- [x] Every string field validated with `.size() <= MAX` and strict key allowlists (`hasAll` + `hasOnly`).
- [x] Email spoofing prevented by requiring `request.auth.token.email_verified == true`.
- [x] Private admin notes isolated into `/projectInternalNotes/{projectId}` so client project reads never leak internal notes.
- [x] Zero `get()`/`exists()` calls inside `allow list` rules to prevent O(n) read-cost amplification.
- **Firebase Client-Write Limitation**: Unauthenticated Firestore writes (`allow create` on `/inquiries`) validate schema and initial status (`New`), but Firestore rules alone cannot rate-limit by IP address. For strict IP rate-limiting in production, route public inquiries through the included `submitPublicInquiry` Cloud Function in `functions/src/index.ts` and enable **Firebase App Check (reCAPTCHA Enterprise)**.

---

## 7. External Accounts & Optional Paid Services

1. **Firebase Auth Providers (Free)**: Google Sign-In is enabled automatically. To use Email/Password sign-in, enable the **Email/Password** provider in the Firebase Console under *Authentication > Sign-in method*.
2. **Firebase Cloud Functions (Blaze Plan Required by Google Cloud)**: Deploying `functions/` (`submitPublicInquiry`, `setClientAccountStatus`, `verifyOnlinePaymentSignature`) requires upgrading the Firebase project to the pay-as-you-go Blaze plan.
3. **Razorpay Online Payments (Optional)**: Set `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET` in Cloud Functions secrets to activate automated online payment verification. Without credentials, the CRM displays manual UPI/Bank Transfer instructions and records admin-verified payments.
