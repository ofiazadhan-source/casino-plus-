<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Casino Plus Game</title>
<style>
    * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
        font-family: 'Arial', sans-serif;
    }
    
    body {
        background: linear-gradient(135deg, #1a1a2e, #16213e);
        color: white;
        min-height: 100vh;
        padding: 20px;
        display: flex;
        flex-direction: column;
        align-items: center;
    }
    
    .header {
        width: 100%;
        max-width: 500px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 20px;
        padding: 15px;
        background: rgba(0, 0, 0, 0.3);
        border-radius: 10px;
    }
    
    .balance {
        font-size: 24px;
        font-weight: bold;
        color: #4CAF50;
    }
    
    .game-container {
        width: 100%;
        max-width: 500px;
        background: rgba(0, 0, 0, 0.4);
        border-radius: 15px;
        padding: 20px;
        box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
    }
    
    .slots {
        display: flex;
        justify-content: center;
        gap: 10px;
        margin: 30px 0;
    }
    
    .slot {
        width: 80px;
        height: 80px;
        background: #2c3e50;
        border-radius: 10px;
        display: flex;
        justify-content: center;
        align-items: center;
        font-size: 40px;
        border: 3px solid #34495e;
    }
    
    .controls {
        display: flex;
        flex-direction: column;
        gap: 15px;
    }
    
    .bet-controls {
        display: flex;
        gap: 10px;
        align-items: center;
    }
    
    input[type="number"] {
        flex: 1;
        padding: 12px;
        border-radius: 8px;
        border: none;
        background: #2c3e50;
        color: white;
        font-size: 18px;
    }
    
    button {
        padding: 15px;
        border: none;
        border-radius: 8px;
        font-size: 18px;
        font-weight: bold;
        cursor: pointer;
        transition: all 0.2s;
    }
    
    button:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
    
    .spin-btn {
        background: #e74c3c;
        color: white;
    }
    
    .spin-btn:hover:not(:disabled) {
        background: #c0392b;
        transform: scale(1.02);
    }
    
    .message {
        text-align: center;
        font-size: 20px;
        margin: 20px 0;
        min-height: 30px;
        font-weight: bold;
    }
    
    .history {
        margin-top: 20px;
        max-height: 200px;
        overflow-y: auto;
    }
    
    .history-item {
        padding: 10px;
        border-bottom: 1px solid rgba(255, 255, 255, 0.1);
        font-size: 14px;
    }
    
    .win {
        color: #4CAF50;
    }
    
    .lose {
        color: #e74c3c;
    }
    
    .loading {
        text-align: center;
        padding: 20px;
        color: #f39c12;
    }
</style>
</head>
<body>
    <div class="header">
        <h2>Casino Plus</h2>
        <div class="balance">Credits: <span id="credits">0</span></div>
    </div>
    
    <div class="game-container">
        <div class="slots">
            <div class="slot" id="slot1">🍒</div>
            <div class="slot" id="slot2">🍋</div>
            <div class="slot" id="slot3">🍊</div>
        </div>
        
        <div class="message" id="message">Place your bet and spin!</div>
        
        <div class="controls">
            <div class="bet-controls">
                <input type="number" id="bet" value="10" min="1" max="1000">
                <button class="spin-btn" id="spinBtn" onclick="spin()">SPIN</button>
            </div>
        </div>
        
        <div class="history" id="history">
            <div class="loading">Connecting to Firebase...</div>
        </div>
    </div>

<script type="module">
    // Import Firebase SDK
    import { initializeApp } from "https://www.gstatic.com/firebasejs/12.13.0/firebase-app.js";
    import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/12.13.0/firebase-auth.js";
    import { getFirestore, doc, getDoc, setDoc, updateDoc, collection, addDoc, query, orderBy, limit, getDocs } from "https://www.gstatic.com/firebasejs/12.13.0/firebase-firestore.js";

    // Your Firebase configuration
    const firebaseConfig = {
        apiKey: "AIzaSyBnAiat7Y8YvBp4WVsSGmRdDMKCJYzkCiw",
        authDomain: "casino-plus-game.firebaseapp.com",
        projectId: "casino-plus-game",
        storageBucket: "casino-plus-game.firebasestorage.app",
        messagingSenderId: "293210899446",
        appId: "1:293210899446:web:a0e56e65c46972c6a0fdd7"
    };

    // Initialize Firebase
    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const db = getFirestore(app);

    // Game variables
    let credits = 0;
    let userId = null;
    const symbols = ['🍒', '🍋', '🍊', '🍇', '⭐', '💎'];
    
    const creditsEl = document.getElementById('credits');
    const messageEl = document.getElementById('message');
    const spinBtn = document.getElementById('spinBtn');
    const historyEl = document.getElementById('history');
    
    // Auto login anonymously
    signInAnonymously(auth).catch(error => {
        console.error("Auth error:", error);
        messageEl.textContent = "Connection error. Refresh page.";
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
        
        if (playerSnap.exists()) {
            credits = playerSnap.data().credits;
        } else {
            credits = 1000; // New player gets 1000 credits
            await setDoc(playerRef, { credits: credits, createdAt: Date.now() });
        }
        updateDisplay();
    }

    async function saveCredits() {
        if (!userId) return;
        const playerRef = doc(db, 'players', userId);
        await updateDoc(playerRef, { credits: credits });
    }

    async function saveHistory(bet, result, winAmount) {
        if (!userId) return;
        const historyRef = collection(db, 'players', userId, 'history');
        await addDoc(historyRef, {
            bet: bet,
            result: result,
            winAmount: winAmount,
            timestamp: Date.now()
        });
    }

    async function loadHistory() {
        if (!userId) return;
        const historyRef = collection(db, 'players', userId, 'history');
        const q = query(historyRef, orderBy('timestamp', 'desc'), limit(10));
        const querySnapshot = await getDocs(q);
        
        historyEl.innerHTML = '';
        if (querySnapshot.empty) {
            historyEl.innerHTML = '<div class="history-item">No history yet</div>';
            return;
        }
        
        querySnapshot.forEach((doc) => {
            const data = doc.data();
            const div = document.createElement('div');
            div.className = 'history-item ' + (data.winAmount > 0 ? 'win' : 'lose');
            div.textContent = `Bet ${data.bet} - ${data.result} ${data.winAmount > 0 ? '+'+data.winAmount : data.winAmount}`;
            historyEl.appendChild(div);
        });
    }

    function updateDisplay() {
        creditsEl.textContent = credits;
        spinBtn.disabled = credits <= 0;
    }

    window.spin = async function() {
        const bet = parseInt(document.getElementById('bet').value);
        
        if (bet > credits) {
            messageEl.textContent = "Not enough credits!";
            return;
        }
        
        if (bet < 1) {
            messageEl.textContent = "Minimum bet is 1!";
            return;
        }
        
        spinBtn.disabled = true;
        credits -= bet;
        updateDisplay();
        messageEl.textContent = "Spinning...";
        
        // Animate slots
        let spins = 0;
        const spinInterval = setInterval(() => {
            document.getElementById('slot1').textContent = symbols[Math.floor(Math.random() * symbols.length)];
            document.getElementById('slot2').textContent = symbols[Math.floor(Math.random() * symbols.length)];
            document.getElementById('slot3').textContent = symbols[Math.floor(Math.random() * symbols.length)];
            spins++;
            
            if (spins > 15) {
                clearInterval(spinInterval);
                checkResult(bet);
            }
        }, 100);
    }

    async function checkResult(bet) {
        const s1 = document.getElementById('slot1').textContent;
        const s2 = document.getElementById('slot2').textContent;
        const s3 = document.getElementById('slot3').textContent;
        
        let winAmount = 0;
        let resultText = "You lost!";
        
        if (s1 === s2 && s2 === s3) {
            winAmount = bet * 10;
            resultText = "JACKPOT! Triple match!";
        } else if (s1 === s2 || s2 === s3 || s1 === s3) {
            winAmount = bet * 2;
            resultText = "Nice! Double match!";
        }
        
        credits += winAmount;
        messageEl.textContent = resultText;
        updateDisplay();
        
        // Save to Firebase
        await saveCredits();
        await saveHistory(bet, resultText, winAmount - bet);
        await loadHistory();
        
        spinBtn.disabled = false;
    }
</script>
</body>
</html>
