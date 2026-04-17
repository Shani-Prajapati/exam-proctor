# AI Exam Proctor — Setup Guide

## YOUR FOLDER STRUCTURE
```
Attention/
├── login.html
├── exam.html  
├── faculty.html
├── SETUP_GUIDE.md
├── freesound_community-siren-alert-96052.mp3
└── model/
    ├── model.json
    ├── weights.bin
    └── metadata.json
```
Open with VS Code Live Server — do NOT double-click (camera won't work on file://)

---

## STEP 1 — EmailJS Setup (Free OTP Email)

1. Go to https://www.emailjs.com and click **Sign Up Free**
2. Add an **Email Service**:
   - Click "Email Services" → "Add New Service"
   - Choose Gmail → Connect your Gmail account → Click "Create Service"
   - Copy the **Service ID** (e.g. `service_abc123`)

3. Create an **Email Template**:
   - Click "Email Templates" → "Create New Template"
   - Set Subject: `Your OTP for AI Exam Proctor`
   - Set Body:
     ```
     Hello {{student_name}},
     
     Your OTP for the exam session is:
     
     {{otp_code}}
     
     This OTP expires in 10 minutes.
     Do not share it with anyone.
     
     AI Exam Proctor System
     ```
   - Copy the **Template ID** (e.g. `template_xyz789`)

4. Get your **Public Key**:
   - Click your name (top right) → "Account"
   - Copy the **Public Key** (e.g. `abc123XYZ`)

5. Open `login.html` and replace these 3 lines near the top of the `<script>`:
   ```javascript
   const EMAILJS_SERVICE_ID  = 'service_abc123';   // your service ID
   const EMAILJS_TEMPLATE_ID = 'template_xyz789';  // your template ID
   const EMAILJS_PUBLIC_KEY  = 'abc123XYZ';        // your public key
   ```

---

## STEP 2 — Firebase Setup (Database + Auth)

### Create Project
1. Go to https://console.firebase.google.com
2. Click **"Add Project"** → Name it `ai-exam-proctor` → Continue
3. Disable Google Analytics (optional) → Click **Create Project**

### Enable Authentication
1. In left sidebar → **Authentication** → **Get Started**
2. Under **Sign-in method** tab → Enable **Email/Password** → Save

### Create Firestore Database
1. In left sidebar → **Firestore Database** → **Create database**
2. Choose **Start in test mode** (allows read/write) → Next
3. Select your region (e.g. asia-south1 for India) → Enable

### Get Firebase Config
1. Click the **gear icon** (Project Settings) → **General** tab
2. Scroll to **"Your apps"** → Click **Web** icon (`</>`)
3. Name it `exam-proctor` → Click **Register app**
4. Copy the `firebaseConfig` object — it looks like:
   ```javascript
   const firebaseConfig = {
     apiKey: "AIzaSy...",
     authDomain: "ai-exam-proctor.firebaseapp.com",
     projectId: "ai-exam-proctor",
     storageBucket: "ai-exam-proctor.appspot.com",
     messagingSenderId: "123456789",
     appId: "1:123456789:web:abc123"
   };
   ```

### Add Firebase to Your Files
Add this to the `<head>` of login.html, exam.html, and faculty.html:
```html
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
<script>
  const firebaseConfig = {
    // PASTE YOUR CONFIG HERE
  };
  firebase.initializeApp(firebaseConfig);
  const auth = firebase.auth();
  const db = firebase.firestore();
</script>
```

### Replace localStorage with Firebase (faculty login)
In login.html, replace `facultyLogin()` with:
```javascript
async function facultyLogin(){
  const email = document.getElementById('f-email').value.trim();
  const pass = document.getElementById('f-pass').value;
  try {
    await auth.signInWithEmailAndPassword(email, pass);
    const user = auth.currentUser;
    const doc = await db.collection('faculty').doc(user.uid).get();
    localStorage.setItem('proctor_session', JSON.stringify({
      role:'faculty', name: doc.data().name, email, 
      uid: user.uid, loginTime: new Date().toISOString()
    }));
    window.location.href='faculty.html';
  } catch(err) {
    showErr('f-err', 'Invalid email or password.');
  }
}
```

### Replace localStorage with Firestore (student violations)
In exam.html, replace `pushToFaculty()` with:
```javascript
async function pushToFaculty(msg, timeStr){
  if(!db) return;
  await db.collection('active_students').doc(session.id).set({
    name: session.name, id: session.id, subject: session.subject,
    examId: session.examId, roomCode: session.roomCode,
    violations: S.alerts, lastViolation: msg,
    lastViolationTime: timeStr, violationLog: S.violationLog,
    integrity: Math.max(0, 100 - S.flags * 7),
    status: S.alerts >= 5 ? 'critical' : 'warning',
    updatedAt: firebase.firestore.FieldValue.serverTimestamp()
  }, { merge: true });
}
```

### Real-time listener in faculty.html
Replace the `setInterval` refresh with:
```javascript
db.collection('active_students')
  .where('roomCode', '==', currentRoomCode)
  .onSnapshot((snapshot) => {
    const students = snapshot.docs.map(d => d.data());
    renderGrid(students);
    updateStats(students);
  });
```

---

## STEP 3 — Firestore Security Rules
In Firebase Console → Firestore → Rules tab, paste:
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /faculty/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
    match /active_students/{studentId} {
      allow read: if request.auth != null;
      allow write: if true; // students write without login
    }
    match /rooms/{code} {
      allow read, write: if request.auth != null;
    }
  }
}
```

---

## HOW THE ROOM CODE SYSTEM WORKS
1. Faculty logs in → a random 6-character code is generated (e.g. `X7K2MQ`)
2. A popup shows this code to the faculty
3. Faculty shares the code with students verbally or on screen
4. Students enter the code when signing in
5. Students without the correct code CANNOT join
6. Faculty dashboard only shows students from their room
7. Code is deleted when faculty logs out
