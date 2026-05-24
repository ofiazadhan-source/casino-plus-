<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Plus Game Casino</title>
<style>
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #1a1a2e;
  color: white;
  text-align: center;
}
.container {
  padding: 20px;
}
.credits {
  font-size: 24px;
  margin: 20px 0;
  color: #ffd700;
}
.bet-controls {
  margin: 20px 0;
}
input[type="number"] {
  padding: 10px;
  width: 150px;
  font-size: 16px;
  border-radius: 8px;
  border: none;
}
button {
  padding: 12px 25px;
  margin: 10px;
  font-size: 16px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background: #e94560;
  color: white;
  font-weight: bold;
}
button:hover {
  background: #ff6b81;
}
button:disabled {
  background: #555;
  cursor: not-allowed;
}
.result {
  font-size: 20px;
  margin-top: 20px;
  min-height: 30px;
}
.history {
  margin-top: 30px;
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}
.history-item {
  background: #16213e;
  padding: 8px;
  margin: 5px 0;
  border-radius: 5px;
  font-size: 14px;
}
.loading {
  color: #00f2fe;
}
.error {
  color: #ff4757;
}
</style>
</head>
<body>
<div class="container">
  <h1>🎰 Plus Game Casino</h1>
  <div class="credits" id="credits">Loading...</div>
  
  <div class="bet-controls">
    <input type="number" id="betAmount" placeholder="Enter bet amount" min="1">
    <br>
    <button id="betBtn" onclick="placeBet()">Place Bet</button>
  </div>
  
  <div class="result" id="result"></div>
  
  <div class="history">
    <h3>Game History</h3>
    <div id="historyList"></div>
  </div>
</div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";
import { getFirestore, doc, getDoc, setDoc, updateDoc, collection, addDoc, query, orderBy, limit, getDocs } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

// Your web app's Firebase configuration
const firebaseConfig = {
  apiKey: "AIzaSyDyqJOypfva6fwZ6hEV07C8xegQbA-ai-0",
  authDomain: "casino-web-dan.firebaseapp.com",
  projectId: "casino-web-dan",
  storageBucket: "casino-web-dan.firebasestorage.app",
  messagingSenderId: "347087435531",
  appId: "1:347087435531:web:02ce3b86d7d643e160940e"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

let userId = null;
let credits = 0;

const creditsEl = document.getElementById('credits');
const resultEl = document.getElementById('result');
const betBtn = document.getElementById('betBtn');
const historyList = document.getElementById('historyList');

// Sign in anonymously
signInAnonymously(auth).catch((error) => {
  console.error("Auth error:", error);
  creditsEl.innerHTML = '<span class="error">Auth Error. Check Firebase settings.</span>';
});

onAuthStateChanged(auth, async (user) => {
  if (user) {
    userId = user.uid;
    await loadPlayerData();
    await loadHistory();
  }
});

async function loadPlayerData() {
  const playerRef = doc(db, 'players', userId);
  const playerSnap = await getDoc(playerRef);
  
  if (!playerSnap.exists()) {
    await setDoc(playerRef, { credits: 1000, createdAt: Date.now() });
    credits = 1000;
  } else {
    credits = playerSnap.data().credits;
  }
  updateCreditsDisplay();
}

async function loadHistory() {
  const historyRef = collection(db, 'players', userId, 'history');
  const q = query(historyRef, orderBy('timestamp', 'desc'), limit(10));
  const querySnap = await getDocs(q);
  
  historyList.innerHTML = '';
  querySnap.forEach(doc => {
    const data = doc.data();
    const item = document.createElement('div');
    item.className = 'history-item';
    item.textContent = `${data.bet > 0 ? '+' : ''}${data.bet} credits - ${new Date(data.timestamp).toLocaleTimeString()}`;
    historyList.appendChild(item);
  });
}

function updateCreditsDisplay() {
  creditsEl.textContent = `Credits: ${credits}`;
}

window.placeBet = async function() {
  const betAmount = parseInt(document.getElementById('betAmount').value);
  
  if (!betAmount || betAmount <= 0) {
    resultEl.innerHTML = '<span class="error">Enter a valid bet amount</span>';
    return;
  }
  
  if (betAmount > credits) {
    resultEl.innerHTML = '<span class="error">Not enough credits</span>';
    return;
  }
  
  betBtn.disabled = true;
  resultEl.innerHTML = '<span class="loading">Spinning...</span>';
  
  // Simulate game logic - 50% chance to win 2x
  const won = Math.random() < 0.5;
  const winAmount = won ? betAmount * 2 : 0;
  const netChange = winAmount - betAmount;
  credits += netChange;
  
  // Update Firestore
  const playerRef = doc(db, 'players', userId);
  await updateDoc(playerRef, { credits: credits });
  
  // Add to history
  const historyRef = collection(db, 'players', userId, 'history');
  await addDoc(historyRef, {
    bet: netChange,
    timestamp: Date.now()
  });
  
  // Show result
  if (won) {
    resultEl.innerHTML = `You won ${winAmount} credits! 🎉`;
  } else {
    resultEl.innerHTML = `You lost ${betAmount} credits 😢`;
  }
  
  updateCreditsDisplay();
  await loadHistory();
  
  betBtn.disabled = false;
  document.getElementById('betAmount').value = '';
}
</script>
</body>
</html>
