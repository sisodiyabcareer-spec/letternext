# LetterNext — Official Letter In. Deadline and Document Pack Out.

[![Cloud Run AI Challenge](https://img.shields.io/badge/Google%20Cloud-Run%20AI%20Challenge-4285F4?logo=googlecloud&logoColor=white)](https://cloud.google.com/run)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

LetterNext is a production-ready triage copilot designed for citizens who receive intimidating, panic-inducing official correspondence (health insurance TPA cashless rejections, bank KYC transaction freeze warnings, scholarship/exam deficiency circulars) and need clear direction tonight.

LetterNext treats all pasted inputs as **untrusted data**, extracts strict deadlines into visual alerts that highlight items expiring within 7 days, isolates a missing-documents pack with an interactive checklist, maps safe official submission channels, and crafts a formal response draft. Every analysis and follow-up discussion is persisted securely per authenticated user in Cloud Firestore.

---

## 1. Architecture & Security Threat Model

LetterNext adheres to strict agentic security modeling across the 5 core threat zones:

```
[ Citizen User / Browser ]
         |
         | Google Sign-In (Firebase Auth)
         v
[ Cloud Run Reverse Proxy / Express Server ]
         |
         +--> Cap payload at 20,000 characters & sanitize input
         +--> Resilient Fallback Ladder:
         |    1. gemini-3.6-flash
         |    2. gemini-3.1-flash-lite
         |    3. gemini-flash-latest
         |    4. gemini-3.7-flash
         |
         v
[ Google Gemini API via @google/genai ]
         |
         v
[ Cloud Firestore Database ]
         |
         +--> Owner-Isolated Security Rules (/users/{userId}/{document=**})
         +--> Zero Insecure Defaults (No `allow read, write: if true`)
```

### Threat Summary Table

| Threat Zone | Identified Risk | Countermeasure & Mitigation |
| :--- | :--- | :--- |
| **Input Surfaces** | Adversarial letter text, prompt injection, buffer overflow | Strict 20,000-character cap; defensive JSON payload deserialization; input treated strictly as data boundaries. |
| **Planning & Reasoning** | "Ignore previous instructions", phishing trap links | System prompt isolates untrusted inputs; triggers `possible_phishing` risk level when malicious links or fee demands appear; forbids suggesting letter links. |
| **Tool Execution** | Credential leakage in client bundles | Server-side `/api/analyze-letter` and `/api/chat-letter` proxy routes; zero exposure of `GEMINI_API_KEY` to client JS. |
| **Memory & State** | Cross-tenant document tampering | Owner-only Firestore rules (`request.auth.uid == userId`); recursive undefined-value stripping utility before all writes. |
| **Inter-System** | Upstream model quota or temporary unavailability | Automated 4-tier fallback ladder with automatic status-code retry on 429, 503, 404, and 500. |

---

## 2. Prerequisites & Environment Setup

Ensure you have installed:
- [Google Cloud SDK (`gcloud` CLI)](https://cloud.google.com/sdk/docs/install)
- [Firebase CLI](https://firebase.google.com/docs/cli)
- [Node.js 20+ & npm](https://nodejs.org)

### Enable Required Google Cloud APIs

```bash
# Set your Google Cloud Project ID
export PROJECT_ID="letternext-4189b"
export REGION="asia-southeast1" # or us-central1
gcloud config set project $PROJECT_ID

# Enable APIs for Cloud Run, Secret Manager, Cloud Build, and Firestore
gcloud services enable \
  run.googleapis.com \
  secretmanager.googleapis.com \
  cloudbuild.googleapis.com \
  firestore.googleapis.com \
  aiplatform.googleapis.com
```

---

## 3. Secret Management Setup (`GEMINI_API_KEY`)

Store your Gemini API key in Google Cloud Secret Manager so it is securely mounted at runtime without hardcoding:

```bash
# 1. Create Secret Manager secret
gcloud secrets create GEMINI_API_KEY --replication-policy="automatic"

# 2. Add your secret version
echo -n "YOUR_GEMINI_API_KEY" | gcloud secrets versions add GEMINI_API_KEY --data-file=-

# 3. Grant Secret Accessor role to the Cloud Run runtime service account
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

gcloud secrets add-iam-policy-binding GEMINI_API_KEY \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

---

## 4. Cloud Firestore & Firebase Auth Configuration

### 1. Provision Firestore Database
Provision Firestore in Native Mode:
```bash
gcloud firestore databases create --location=$REGION --type=firestore-native
```

### 2. Configure Firebase Authentication
1. Go to the [Firebase Console](https://console.firebase.google.com) and link your Google Cloud project.
2. Under **Authentication** > **Sign-in method**, enable **Google**.
3. Under **Authorized domains**, add your Cloud Run domain and `localhost`.

### 3. Deploy Owner-Bound Firestore Security Rules

Create or inspect `firestore.rules`:
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // User profile document: users/{userId}
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;

      // Letter documents: users/{userId}/letters/{letterId}
      match /letters/{letterId} {
        allow read, write: if request.auth != null && request.auth.uid == userId;

        // Follow-up chat turns: users/{userId}/letters/{letterId}/turns/{turnId}
        match /turns/{turnId} {
          allow read, write: if request.auth != null && request.auth.uid == userId;
        }
      }
    }
  }
}
```

Deploy rules using the Firebase CLI:
```bash
firebase deploy --only firestore:rules
```

---

## 5. Local Development

```bash
# Install dependencies
npm install

# Configure local environment variables (.env)
cp .env.example .env
# Fill in your GEMINI_API_KEY and Firebase web credentials

# Start full-stack development server (Express + Vite) on port 3000
npm run dev
```
Open `http://localhost:3000` in your browser.

---

## 6. Google Cloud Run Deployment

Deploy LetterNext directly to Cloud Run:

```bash
# Build and deploy container to Cloud Run with Secret Manager binding
gcloud run deploy letternext \
  --source=. \
  --region=$REGION \
  --platform=managed \
  --allow-unauthenticated \
  --set-secrets=GEMINI_API_KEY=GEMINI_API_KEY:latest \
  --port=3000
```

### Mandatory Campaign Verification Label

Apply the required challenge verification label to register your Cloud Run service:

```bash
gcloud run services update letternext \
  --update-labels=dev-tutorial=cloud-run-ai-challenge \
  --region=$REGION
```

---

## 7. Functional Stability & Testing Walkthroughs

The following end-to-end test scenarios provide a complete specification for QA automation tools or manual walkthroughs:

### Scenario 1: Authentication & Landing Page
- **Action**: Visit app root without being logged in.
- **Expected Outcome**: Clean landing page displays the headline *"Official letter in. Deadline and document pack out."*, with three clickable sample cards and a *"Sign in with Google"* CTA. No password fields are rendered.
- **Action**: Click *"Sign in with Google"*.
- **Expected Outcome**: Firebase Authentication authenticates the user, stores their user document under `users/{uid}`, and transitions into the private workspace dashboard.

### Scenario 2: Sample Letter 1 (Insurance TPA Cashless Denial)
- **Action**: Click *"Sample 1: TPA"* in the toolbar.
- **Expected Outcome**: Letter textarea populates with Med-Health TPA claim rejection text and 48-hour deficiency query 4.2.
- **Action**: Click *"Analyze Letter"*.
- **Expected Outcome**:
  - The Action Card renders with `letter_type: insurance_tpa` and `risk_level: act_now`.
  - The deadline chip detects the 48-hour cutoff and renders with a highlighted red urgent badge.
  - The Missing-Document Pack lists the 5 demanded documents (indoor case papers, doctor certificate, baseline USG reports, etc.).
  - The draft formal reply provides a structured review request referencing the claim ID.
  - The record is persisted to `users/{uid}/letters/{letterId}` with a confirmation indicator.

### Scenario 3: Missing Document Interactive Checklist
- **Action**: In the Action Card's Missing-Document Pack, click item #1 to toggle it.
- **Expected Outcome**: Checkbox activates with an emerald checkmark, strikes through the text, and updates the secured counter (e.g., *"1 of 5 secured"*).
- **Action**: Click *"Copy List"*.
- **Expected Outcome**: Clipboard receives a numbered checklist ready to print or email.

### Scenario 4: Multi-turn Thread Follow-up
- **Action**: Scroll to the follow-up thread below the Action Card.
- **Action**: Type *"I already collected the 3-year policy renewals. Please revise the draft reply to mention this."* and click *"Send"*.
- **Expected Outcome**:
  - The user prompt appends as a turn under `users/{uid}/letters/{letterId}/turns/`.
  - Gemini responds with an updated draft reply acknowledging the attached renewals.
  - The thread displays chronologically with clear visual separation.

### Scenario 5: Sample 2 (Bank KYC Freeze Alert & Phishing Defense)
- **Action**: Click *"Sample 2: Bank"*.
- **Action**: Click *"Analyze Letter"*.
- **Expected Outcome**:
  - Action card warns against clicking SMS links or bit.ly URLs.
  - Official channel instructs the citizen to either visit the physical home branch with original OVDs or type the verified domain URL into their address bar.

### Scenario 6: Past Letter History Retrieval
- **Action**: Click *"New Analysis"* to clear the editor.
- **Action**: Click an item from the left history sidebar.
- **Expected Outcome**:
  - The past letter, its Action Card, and all saved chat turns load immediately from Firestore.
