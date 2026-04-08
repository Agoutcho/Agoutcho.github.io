<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Mastermind Dweet</title>
    <style>
        :root {
            --primary: #4F46E5;
            --bg: #F3F4F6;
            --text: #1F2937;
            --card: #FFFFFF;
            --success: #10B981;
            --warning: #F59E0B;
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }
        .container {
            background-color: var(--card);
            padding: 24px;
            border-radius: 16px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.05);
            width: 100%;
            max-width: 400px;
            box-sizing: border-box;
        }
        h1 { font-size: 1.5rem; text-align: center; margin-top: 0; }
        label { font-weight: bold; font-size: 0.9rem; margin-top: 12px; display: block; }
        input {
            width: 100%;
            padding: 12px;
            margin-top: 6px;
            border: 1px solid #D1D5DB;
            border-radius: 8px;
            font-size: 1rem;
            box-sizing: border-box;
        }
        button {
            width: 100%;
            padding: 14px;
            margin-top: 20px;
            background-color: var(--primary);
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 1rem;
            font-weight: bold;
            cursor: pointer;
            transition: background-color 0.2s;
        }
        button:active { background-color: #4338CA; }
        button:disabled { background-color: #9CA3AF; cursor: not-allowed; }
        .hidden { display: none !important; }
        
        .status-box {
            background-color: #EEF2FF;
            padding: 12px;
            border-radius: 8px;
            text-align: center;
            font-size: 0.9rem;
            margin-bottom: 20px;
            color: var(--primary);
            font-weight: bold;
        }
        .history {
            margin-top: 20px;
            border-top: 1px solid #E5E7EB;
            padding-top: 16px;
        }
        .guess-row {
            display: flex;
            justify-content: space-between;
            padding: 8px 0;
            border-bottom: 1px solid #F3F4F6;
        }
        .guess-number { font-family: monospace; font-size: 1.2rem; font-weight: bold; letter-spacing: 2px;}
        .guess-result { font-size: 0.9rem; }
        .bulls { color: var(--success); font-weight: bold;}
        .cows { color: var(--warning); font-weight: bold;}
        .game-over { text-align: center; margin-top: 20px; font-size: 1.2rem; font-weight: bold; color: var(--success); }
    </style>
</head>
<body>

<div class="container" id="screen-setup">
    <h1>🔴 Mastermind 🔵</h1>
    <p style="text-align: center; font-size: 0.85rem; color: #6B7280;">Jouez en temps réel via Dweet.cc</p>
    
    <label for="roomName">Nom du salon (partagez-le avec l'adversaire)</label>
    <input type="text" id="roomName" placeholder="Ex: SalonDeThomas" autocapitalize="none">
    
    <label for="pseudo">Votre Pseudo</label>
    <input type="text" id="pseudo" placeholder="Ex: Joueur1">
    
    <label for="mySecret">Nombre à faire deviner (4 chiffres)</label>
    <input type="number" id="mySecret" placeholder="Ex: 4815" maxlength="4" pattern="\d*">
    
    <button onclick="joinGame()">Rejoindre la partie</button>
</div>

<div class="container hidden" id="screen-game">
    <div class="status-box" id="status-text">En attente de l'adversaire...</div>
    
    <div id="play-area" class="hidden">
        <label for="guessInput">Devinez le nombre de <span id="opponent-name">l'adversaire</span> :</label>
        <div style="display: flex; gap: 8px;">
            <input type="number" id="guessInput" placeholder="4 chiffres..." maxlength="4" pattern="\d*">
            <button onclick="makeGuess()" style="margin-top: 6px; width: auto; padding: 0 20px;">OK</button>
        </div>
        
        <div class="history" id="history">
            </div>
    </div>

    <div id="game-over-area" class="hidden game-over"></div>
</div>

<script>
    // Variables d'état du jeu
    let state = {
        room: "",
        pseudo: "",
        mySecret: "",     // Le nombre que l'autre doit deviner
        myAttempts: 0,
        won: false
    };

    let opponent = {
        pseudo: null,
        secret: null,     // Le nombre que je dois deviner
        attempts: 0,
        won: false
    };

    let pollingInterval;

    // 1. Initialisation et connexion au salon
    function joinGame() {
        state.room = document.getElementById("roomName").value.trim();
        state.pseudo = document.getElementById("pseudo").value.trim();
        state.mySecret = document.getElementById("mySecret").value.trim();

        if (!state.room || !state.pseudo || state.mySecret.length !== 4) {
            alert("Veuillez remplir tous les champs (le code doit faire 4 chiffres).");
            return;
        }

        // Changement d'écran
        document.getElementById("screen-setup").classList.add("hidden");
        document.getElementById("screen-game").classList.remove("hidden");

        // Envoyer notre état initial
        sendStateToDweet();

        // Commencer à écouter le salon toutes les 2 secondes
        pollingInterval = setInterval(pollDweet, 2000);
        pollDweet(); // Premier appel immédiat
    }

    // 2. Envoyer notre état au serveur Dweet
    function sendStateToDweet() {
        const url = `https://dweet.cc/dweet/for/${encodeURIComponent(state.room)}?pseudo=${encodeURIComponent(state.pseudo)}&secret=${state.mySecret}&attempts=${state.myAttempts}&won=${state.won}`;
        fetch(url).catch(e => console.error("Erreur Dweet:", e));
    }

    // 3. Lire les messages du salon pour trouver l'adversaire
    async function pollDweet() {
        if (state.won || opponent.won) return; // Si la partie est finie, on arrête de poll

        try {
            const response = await fetch(`https://dweet.cc/get/dweets/for/${encodeURIComponent(state.room)}`);
            const data = await response.json();

            if (data.this === "succeeded") {
                const dweets = data.with;
                
                // On cherche le dweet le plus récent de quelqu'un d'autre
                for (let dweet of dweets) {
                    const content = dweet.content;
                    
                    if (content.pseudo && content.pseudo !== state.pseudo) {
                        // On a trouvé l'adversaire !
                        let isNewOpponent = (opponent.pseudo === null);
                        
                        opponent.pseudo = content.pseudo;
                        opponent.secret = String(content.secret); // Ce qu'on doit deviner
                        opponent.attempts = parseInt(content.attempts) || 0;
                        opponent.won = (content.won === "true" || content.won === true);

                        if (isNewOpponent) {
                            startGame();
                        }

                        updateStatusUI();
                        checkGameOver();
                        break; // On a les infos les plus récentes, on arrête la boucle
                    }
                }
            }
        } catch (e) {
            console.error("Erreur lecture:", e);
        }
    }

    // 4. L'adversaire est là, on lance l'interface de jeu
    function startGame() {
        document.getElementById("status-text").innerText = `Partie en cours !`;
        document.getElementById("opponent-name").innerText = opponent.pseudo;
        document.getElementById("play-area").classList.remove("hidden");
    }

    // 5. Mise à jour du texte d'information
    function updateStatusUI() {
        const statusBox = document.getElementById("status-text");
        if (opponent.pseudo) {
            statusBox.innerText = `L'adversaire (${opponent.pseudo}) a fait ${opponent.attempts} essai(s).`;
        }
    }

    // 6. Logique de vérification d'un essai
    function makeGuess() {
        const guessInput = document.getElementById("guessInput");
        const guess = guessInput.value.trim();

        if (guess.length !== 4) {
            alert("Veuillez entrer 4 chiffres.");
            return;
        }

        state.myAttempts++;
        let result = evaluateMastermind(guess, opponent.secret);

        // Ajouter à l'historique visuel
        addGuessToHistory(guess, result.bulls, result.cows);

        if (result.bulls === 4) {
            state.won = true;
        }

        // On nettoie l'input et on envoie notre nouvel état
        guessInput.value = "";
        sendStateToDweet();
        checkGameOver();
    }

    // 7. Calcul des Bien placés (Bulls) et Mal placés (Cows)
    function evaluateMastermind(guess, secret) {
        let bulls = 0;
        let cows = 0;
        let gArr = guess.split('');
        let sArr = secret.split('');

        // 1er passage : chercher les bien placés (bulls)
        for (let i = 0; i < 4; i++) {
            if (gArr[i] === sArr[i]) {
                bulls++;
                gArr[i] = null; // Marquer comme utilisé
                sArr[i] = null;
            }
        }

        // 2ème passage : chercher les mal placés (cows)
        for (let i = 0; i < 4; i++) {
            if (gArr[i] !== null) {
                let index = sArr.indexOf(gArr[i]);
                if (index !== -1) {
                    cows++;
                    sArr[index] = null; // Marquer comme utilisé
                }
            }
        }
        return { bulls, cows };
    }

    // 8. Affichage de l'historique
    function addGuessToHistory(guess, bulls, cows) {
        const historyDiv = document.getElementById("history");
        const row = document.createElement("div");
        row.className = "guess-row";
        row.innerHTML = `
            <div class="guess-number">${guess}</div>
            <div class="guess-result">
                <span class="bulls">${bulls} bien placé(s)</span> | 
                <span class="cows">${cows} mal placé(s)</span>
            </div>
        `;
        // Ajouter en haut de la liste
        historyDiv.insertBefore(row, historyDiv.firstChild);
    }

    // 9. Vérification des conditions de victoire
    function checkGameOver() {
        const playArea = document.getElementById("play-area");
        const gameOverArea = document.getElementById("game-over-area");
        const statusBox = document.getElementById("status-text");

        if (state.won || opponent.won) {
            playArea.classList.add("hidden");
            statusBox.classList.add("hidden");
            gameOverArea.classList.remove("hidden");
            clearInterval(pollingInterval);

            if (state.won && opponent.won) {
                // Cas d'égalité très rare sur la même seconde
                gameOverArea.innerText = "Match Nul ! Vous avez trouvé en même temps !";
            } else if (state.won) {
                gameOverArea.innerText = `🎉 Gagné ! Vous avez trouvé le code de ${opponent.pseudo} en ${state.myAttempts} essais.`;
            } else {
                gameOverArea.innerHTML = `💀 Perdu !<br>${opponent.pseudo} a trouvé votre code en ${opponent.attempts} essais.<br><br><span style="font-size:0.9rem; color:#6B7280">Son code secret était : ${opponent.secret}</span>`;
            }
        }
    }
</script>

</body>
</html>
