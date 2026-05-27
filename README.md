<div align="center">
<img src="assets/logo.png" alt="bKash Sync Logo" width="64" height="64" />
<br/>

<h1>bKash Sync</h1>

</div>

**Real-time bKash SMS transaction monitoring and synchronization with custom backend APIs**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform: Android](https://img.shields.io/badge/Platform-Android-green.svg)](https://www.android.com/)
[![Server: Cloudflare Workers](https://img.shields.io/badge/Server-Cloudflare_Workers-orange.svg)](https://workers.cloudflare.com/)
[![Min SDK](https://img.shields.io/badge/minSdk-24-blue.svg)](https://developer.android.com/studio)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.2.10-purple.svg)](https://kotlinlang.org/)

## 📱 Overview

bKash Sync is an Android application that automatically monitors incoming bKash SMS messages, extracts transaction details, and synchronizes them with a custom backend server. The system uses an offline-first architecture with bidirectional sync capabilities.

## 💡 Use Case

bKash Sync enables automated P2P (person-to-person) bKash payment verification — no merchant account or bKash API access required.

**How it works:**

1. You create a payment page on your website with a product price and your personal bKash number
2. A customer sends money from their bKash to your number and receives an SMS confirmation with a TrxID
3. bKash Sync captures the SMS on your phone and syncs it to your Cloudflare backend
4. The customer enters their phone number and TrxID on your payment page
5. Your website queries your backend: `GET /user/transactions` and matches `senderNumber` + `trxId` + `amount`
6. Payment verified automatically — no manual SMS checking

**What developers can build with this:**

- **Self-hosted payment pages** — Sell digital products, services, or accept donations with automated bKash verification
- **Order processing systems** — Match incoming bKash payments to orders without polling the official bKash API
- **Personal finance dashboards** — Log and analyze all bKash income in your own database
- **Payment bots / webhooks** — Trigger actions (unlock content, send email, update status) when a payment is confirmed

## 📸 Screenshots

| Dashboard                          | Settings                         | Transaction List                         |
| ---------------------------------- | -------------------------------- | ---------------------------------------- |
| ![Dashboard](assets/dashboard.png) | ![Settings](assets/settings.png) | ![Transactions](assets/transactions.png) |

## 🏗️ Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌────────────────────┐
│  Incoming SMS   │────▶│  SmsReceiver     │────▶│  SmsParser         │
└─────────────────┘     └──────────────────┘     └────────────────────┘
                                │                         │
                                ▼                         ▼
                        ┌──────────────────┐     ┌────────────────────┐
                        │  Transaction     │     │  Parsed Data       │
                        │  Repository      │◀───│  (amount, sender,  │
                        └──────────────────┘     │   balance, trxId)    │
                                │                └────────────────────┘
                                ▼                         │
                        ┌──────────────────┐              ▼
                        │  Room Database   │     ┌────────────────────┐
                        │  (local cache)   │     │  Outbox Queue      │
                        └──────────────────┘     └────────────────────┘
                                │                         │
                                ▼                         ▼
                        ┌──────────────────┐     ┌────────────────────┐
                        │  SyncWorker      │────▶│  Cloudflare API    │
                        │  (WorkManager)   │     │  (D1 Database)     │
                        └──────────────────┘     └────────────────────┘
```

## 📂 Project Structure

```
bkash-sync/
├── bkash-sync_app/           # Android application source code
│   ├── app/src/main/java/com/example/
│   │   ├── receiver/        # SmsReceiver.kt - SMS broadcast handling
│   │   ├── util/            # SmsParser.kt - SMS parsing logic
│   │   ├── data/
│   │   │   ├── api/         # Retrofit client & data models
│   │   │   ├── database/    # Room entities & DAO
│   │   │   ├── pref/        # Encrypted preferences
│   │   │   ├── repository/  # TransactionRepository.kt
│   │   │   └── worker/      # SyncWorker.kt
│   │   ├── service/         # BackgroundSyncService.kt
│   │   └── ui/              # MainViewModel.kt & Compose UI
│   └── ...
├── server/cloudflare-server/  # Cloudflare Worker backend
│   ├── src/
│   │   ├── index.ts         # API endpoints
│   │   └── db.ts            # Database service
│   ├── schema.sql           # Database schema
│   └── wrangler.toml        # Worker configuration
└── assets/               # App screenshots
```

## 🗄️ Local Database Schema

The app uses Room (SQLite) for offline-first storage:

| Table          | Key Fields                                                                               | Purpose                                 |
| -------------- | ---------------------------------------------------------------------------------------- | --------------------------------------- |
| `transactions` | `trxId` (unique), amount, senderNumber, balance, datetime, rawSms, simSlot               | Parsed SMS storage with deduplication   |
| `outbox`       | `trxId` (unique), payloadJson, status (PENDING/SENT/FAILED), retryCount, lastAttemptTime | Sync queue with retry tracking          |
| `sync_state`   | `lastSmsTimestamp`, `lastSyncTime`                                                       | Tracks last processed SMS and sync time |

## 🚀 Server Setup

### Prerequisites

- [Node.js](https://nodejs.org/) (v18+ recommended)
- [Cloudflare account](https://dash.cloudflare.com/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) (`npm install -g wrangler`)

### Deploy to Cloudflare Workers

1. **Install dependencies**

   ```bash
   cd server/cloudflare-server
   npm install
   ```

2. **Create D1 Database**

   ```bash
   wrangler d1 create bkash-sync-db
   ```

   Copy the database ID and update `wrangler.toml`

3. **Set Admin API Key Secret**

   ```bash
   wrangler secret put ADMIN_API_KEY
   ```

4. **Initialize Database Schema**

   ```bash
   npm run db:init                     # Local development
   npm run db:deploy                   # Production
   # or directly:
   wrangler d1 execute bkash-sync-db --local --file=./schema.sql
   wrangler d1 execute bkash-sync-db --remote --file=./schema.sql
   ```

5. **Deploy Worker**
   ```bash
   npm run deploy
   # or directly:
   wrangler deploy
   ```

## 📱 Android App Setup

### Prerequisites

- [Android Studio](https://developer.android.com/studio)
- Android SDK (API 21+ minimum)

### Build Instructions

1. **Open in Android Studio**
   - Open Android Studio
   - Select "Open" and choose the `bkash-sync_app` directory

2. **Build APK**
   ```bash
   cd bkash-sync_app
   ./gradlew assembleRelease
   ```
   Output: `app/build/outputs/apk/release/app-release.apk`

## 📦 Installation

### Download Pre-built APK

Download the latest APK from the [Releases](../../releases) page.

### Install on Android Device

> ⚠️ **Important**: Google Play Protect may flag this app as unrecognized. Follow these steps:

1. **Download the APK** to your device
2. **Disable Play Protect temporarily**:
   - Open **Play Store** app
   - Tap your **profile icon** → **Play Protect**
   - Tap the **gear icon** (Settings)
   - **Turn off** "Scan apps with Play Protect"
   - Or tap "Manage apps" → "What to scan" → Disable for bKash Sync

3. **Install the APK**:
   - Open **Settings** → **Apps** → **Special access** → **Install unknown apps**
   - Select your browser/file manager
   - **Enable** "Allow from this source"
   - Open the downloaded APK and install

4. **Grant Permissions**:
   - SMS permissions (RECEIVE_SMS, READ_SMS)
   - Notification permission (Android 13+)
   - Battery optimization exemption (for background service)

5. **Re-enable Play Protect** after installation (optional)

## 🔧 Configuration

### Server URL Format

```
https://your-worker.your-subdomain.workers.dev
```

### API Key

- Obtain from your server administrator
- Or generate using admin endpoint:
  ```bash
  curl -X POST https://your-worker.workers.dev/admin/keys \
    -H "Authorization: Bearer YOUR_ADMIN_KEY" \
    -H "Content-Type: application/json" \
    -d '{"phoneNumber": "01XXXXXXXXX", "key": "your-custom-api-key"}'
  ```

## 📡 API Endpoints

| Endpoint              | Method | Auth      | Description                      |
| --------------------- | ------ | --------- | -------------------------------- |
| `/verify`             | POST   | API Key   | Verify API key validity          |
| `/transactions`       | POST   | API Key   | Upload batched transactions      |
| `/synctransaction`    | POST   | API Key   | Smart Delta-Sync (bidirectional) |
| `/user/transactions`  | GET    | API Key   | Get user's transactions          |
| `/admin/keys`         | POST   | Admin Key | Issue/revoke user API keys       |
| `/admin/users`        | GET    | Admin Key | List all registered users        |
| `/admin/transactions` | GET    | Admin Key | Get all transactions             |

## 🛠️ Features

- **Real-time SMS Monitoring**: Instant capture of bKash incoming transactions
- **Offline-First**: All data stored locally first, synced when online
- **Smart Delta-Sync**: Bidirectional reconciliation between device and server
- **Multi-SIM Support**: Tracks which SIM received each transaction
- **Encrypted Storage**: API key encrypted with AES/CBC/PKCS5Padding
- **Foreground Service**: Persistent background operation with notification
- **Historical Scan**: Import existing SMS transactions on setup
- **Auto-Retry**: Failed sync items retry with exponential backoff (max 8 attempts)
- **Configurable Sync**: Adjustable sync intervals and auto-reconciliation

### Smart Delta-Sync Protocol

The Smart Delta-Sync ensures bidirectional consistency:

1. Client POSTs to `/synctransaction` with `{ userPhoneNumber, localTrxIds[] }`
2. Server compares against D1 records for that user
3. **`missingOnServer`**: IDs client has but server doesn't → re-queued to outbox for upload
4. **`missingOnClient`**: Full records server has but client doesn't → inserted locally as `SENT`
5. Result: Both sides converge to the same dataset without full re-upload

> **Security Note**: The API key is encrypted at rest using AES/CBC/PKCS5Padding with a deterministic key derived from the app binary. This prevents trivial SharedPreferences dumps but does not protect against binary analysis. For production use, consider using the Android Keystore system.

## 🧪 Testing

### Run Tests

```bash
# Unit tests (JUnit + Robolectric)
./gradlew test

# Screenshot tests (generates PNGs in build/outputs)
./gradlew testDebug

# Instrumented tests (requires emulator/device)
./gradlew connectedAndroidTest
```

## 🔧 Troubleshooting

| Problem                    | Likely Cause                     | Solution                                                                 |
| -------------------------- | -------------------------------- | ------------------------------------------------------------------------ |
| SMS not captured           | App not configured               | Complete the Settings tab with valid API endpoint + key                  |
| Sync failing with 401/403  | Invalid API key                  | Verify key with `/verify` endpoint; check key status in `api_keys` table |
| Transactions stuck PENDING | Network issues                   | Ensure internet connectivity; check backend logs                         |
| Foreground service killed  | Battery optimization             | Exempt app from battery optimization in system settings                  |
| Duplicate transactions     | Same `trxId` from different SIMs | Room's `IGNORE` conflict strategy prevents duplicates                    |

## 📄 License

MIT License - See [LICENSE](LICENSE) for details

## ⚠️ Disclaimer

This application is for educational and personal use only. Ensure you comply with local laws and regulations when using SMS monitoring features. The developers are not responsible for any misuse of this software.
