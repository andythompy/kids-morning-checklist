# Firebase Setup Instructions

This app now uses Firebase Firestore for persistent state storage across device restarts and timeouts.

## Step 1: Create a Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click "Add project" or select an existing project
3. Follow the setup wizard (you can disable Google Analytics if you want)

## Step 2: Enable Firestore

1. In your Firebase project, go to **Build** → **Firestore Database**
2. Click "Create database"
3. Choose **Start in production mode** (we'll set rules next)
4. Select a location close to you (e.g., australia-southeast1)
5. Click "Enable"

## Step 3: Set Firestore Security Rules

1. In Firestore Database, click on the **Rules** tab
2. Replace the default rules with:

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /checklists/{document=**} {
      allow read, write: if request.auth != null;
    }
    match /config/{document=**} {
      allow read: if request.auth != null;
      allow write: if false;
    }
    match /display/{document=**} {
      allow read: if request.auth != null;
      allow write: if false;
    }
  }
}
```

3. Click **Publish**

What these rules mean:

- **`checklists/`** holds the daily tick documents. Any signed-in client can read and write them. Because sign-in is anonymous, that means **any anonymous client can read (and write) `checklists`**, not just your devices. Don't store anything private there.
- **`config/`** holds the shared task list (`config/checklist`). The app reads it but can never write it. Edit it in the Firebase Console, or seed it from the `morning-canvas` service, which uses a service account and bypasses these rules.
- **`display/`** is reserved for the `morning-canvas` service's display state. It's read-only for the app too.
- Anything else is denied.

## Step 4: Enable Anonymous Authentication

1. Go to **Build** → **Authentication**
2. Click "Get started" (if first time)
3. Go to the **Sign-in method** tab
4. Enable **Anonymous** sign-in
5. Click "Save"

## Step 5: Get Your Firebase Config

1. Go to **Project settings** (gear icon near "Project Overview")
2. Scroll down to "Your apps"
3. Click the **Web** icon (`</>`) to add a web app
4. Register your app with a nickname (e.g., "Morning Checklist")
5. Copy the `firebaseConfig` object

## Step 6: Update index.html

1. Open `index.html`
2. Find the `firebaseConfig` object (around line 188)
3. Replace the placeholder values with your actual Firebase config:

```javascript
const firebaseConfig = {
    apiKey: "YOUR_ACTUAL_API_KEY",
    authDomain: "your-project.firebaseapp.com",
    projectId: "your-project-id",
    storageBucket: "your-project.appspot.com",
    messagingSenderId: "123456789",
    appId: "1:123456789:web:abcdef123456"
};
```

## Step 7: Deploy and Test

1. Commit and push your changes to GitHub
2. Wait for GitHub Actions to deploy
3. Open the app on your Google Nest Hub
4. Check the browser console for "Signed in to Firebase anonymously"

## How It Works

- **Daily Documents**: Each day gets its own document in the `checklists` collection (e.g., `2025-11-03`)
- **Shared Task List**: Children, tasks and per-child chores are read live from `config/checklist`. If that document is missing, invalid or can't be read, the app uses the built-in `CONFIG` in `index.html` instead
- **Real-time Sync**: All devices viewing the app will see updates in real-time
- **Persistent Storage**: State survives device restarts, timeouts, and cache clears
- **Anonymous Auth**: No user accounts needed - Firebase authenticates the device anonymously

## Troubleshooting

**"Missing or insufficient permissions"**
- Make sure you've enabled Anonymous Authentication
- Check your Firestore security rules allow anonymous users

**"Firebase: Error (auth/...)"**
- Verify your Firebase config is correct
- Check your API key and project ID

**State not persisting**
- Open browser console and check for errors
- Verify Firestore rules are published
- Check that the document is being created in Firestore Console

## Testing Locally

To test locally before deploying:
1. Open `index.html` in a modern browser (Chrome, Edge, Firefox)
2. Open Developer Tools → Console
3. You should see "Signed in to Firebase anonymously"
4. Check tasks off - they should persist after refresh
