
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mate-Blast: Operación Robot</title>
    <style>
        :root {
            --primary: #00f2ff;
            --secondary: #7000ff;
            --accent: #39ff14;
            --error: #ff003c;
            --bg: #05050a;
        }

        body {
            background-color: var(--bg);
            color: white;
            font-family: 'Segoe UI', Roboto, sans-serif;
            margin: 0;
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            overflow: hidden;
            perspective: 1000px;
        }

        /* Fondo de Iconos Flotantes */
        .background-elements {
            position: fixed;
            top: 0; left: 0; width: 100%; height: 100%;
            z-index: -1;
            pointer-events: none;
        }

        .float-item {
            position: absolute;
            color: rgba(255, 255, 255, 0.1);
            font-size: 2rem;
            font-weight: bold;
            animation: float linear infinite;
        }

        @keyframes float {
            from { transform: translateY(110vh) rotate(0deg); }
            to { transform: translateY(-10vh) rotate(360deg); }
        }

        /* Contenedor Principal */
        #game-view {
            background: rgba(20, 20, 40, 0.9);
            border: 3px solid var(--primary);
            border-radius: 25px;
            padding: 40px;
            width: 500px;
            text-align: center;
            box-shadow: 0 0 30px rgba(0, 242, 255, 0.4);
            position: relative;
            z-index: 10;
        }

        /* Mate el Robot */
        #mate-robot {
            position: absolute;
            right: -120px;
            top: 50%;
            transform: translateY(-50%);
            font-size: 100px;
            text-shadow: 0 0 20px var(--primary);
            transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
        }

        h1 { font-size: 2.5rem; margin-top: 0; color: var(--primary); text-shadow: 2px 2px var(--secondary); }

        /* Barra de Tiempo */
        #timer-container {
            width: 100%;
            height: 12px;
            background: #222;
            border-radius: 6px;
            margin: 20px 0;
            border: 1px solid #444;
        }

        #timer-bar {
            width: 100%;
            height: 100%;
            background: linear-gradient(90deg, var(--primary), var(--accent));
            border-radius: 6px;
            box-shadow: 0 0 10px var(--primary);
        }

        #question { font-size: 60px; font-weight: 800; margin: 15px 0; letter-spacing: 5px; }

        .input-group { margin: 20px 0; }

        input {
            background: #000;
            border: 2px solid var(--secondary);
            color: var(--accent);
            font-size: 40px;
            width: 150px;
            text-align: center;
            border-radius: 12px;
            padding: 10px;
            outline: none;
            box-shadow: inset 0 0 10px rgba(112, 0, 255, 0.5);
        }

        /* Botones Estilo Gamer */
        .btn {
            background: var(--secondary);
            color: white;
            border: none;
            padding: 15px 30px;
            font-size: 18px;
            font-weight: bold;
            border-radius: 50px;
            cursor: pointer;
            text-transform: uppercase;
            transition: 0.2s;
            box-shadow: 0 5px 0 #4a00a8;
            margin: 10px;
        }

        .btn:hover { transform: translateY(-3px); box-shadow: 0 8px 0 #4a00a8; filter: brightness(1.2); }
        .btn:active { transform: translateY(2px); box-shadow: 0 2px 0 #4a00a8; }
        .btn-send { background: var(--accent); color: #000; box-shadow: 0 5px 0 #28b80e; }
        .btn-send:active { box-shadow: 0 2px 0 #28b80e; }

        .hidden { display: none; }
        #stats { font-weight: bold; color: var(--primary); letter-spacing: 2px; }

    </style>
</head>
<body>

    <div class="background-elements" id="bg-elements"></div>

    <div id="game-view">
        <div id="mate-robot">🤖</div>
        
        <div id="start-screen">
            <h1>MATE-BLAST</h1>
            <p>¡Supera el reto de las tablas!</p>
            <button class="btn" onclick="startGame()">EMPEZAR</button>
        </div>

        <div id="game-screen" class="hidden">
            <div id="stats">TABLA <span id="current-q">1</span> DE 20</div>
            <div id="timer-container"><div id="timer-bar"></div></div>
            <div id="question">7 x 8</div>
            <div class="input-group">
                <input type="number" id="answer-input" autofocus placeholder="?">
            </div>
            <button class="btn btn-send" onclick="checkAnswer()">ENVIAR RESPUESTA</button>
        </div>

        <div id="result-screen" class="hidden">
            <h1>RESULTADOS</h1>
            <p id="final-msg" style="font-size: 1.5rem;"></p>
            <h2 id="percentage" style="font-size: 3rem; color: var(--accent);">0%</h2>
            <button class="btn" onclick="resetGame()">VOLVER A EMPEZAR</button>
        </div>
    </div>

    <script>
        let score = 0;
        let questionCount = 0;
        let currentAnswer = 0;
        let timerInterval;
        let timeLeft = 100;
        const totalQuestions = 20;

        // Crear elementos flotantes de fondo
        const symbols = ['+', '-', '×', '÷', '8', '5', '3'];
        const bgContainer = document.getElementById('bg-elements');
        for (let i = 0; i < 20; i++) {
            let el = document.createElement('div');
            el.className = 'float-item';
            el.innerText = symbols[Math.floor(Math.random() * symbols.length)];
            el.style.left = Math.random() * 100 + 'vw';
            el.style.animationDuration = (Math.random() * 5 + 5) + 's';
            el.style.animationDelay = Math.random() * 5 + 's';
            bgContainer.appendChild(el);
        }

        // Sintetizador de Sonido Motivante
        const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        
        function playSfx(type) {
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.connect(gain);
            gain.connect(audioCtx.destination);

            if (type === 'success') {
                // Sonido tipo "moneda" o "subida de nivel"
                osc.type = 'square';
                osc.frequency.setValueAtTime(523.25, audioCtx.currentTime); // C5
                osc.frequency.exponentialRampToValueAtTime(1046.50, audioCtx.currentTime + 0.1); // C6
                gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.3);
                osc.start();
                osc.stop(audioCtx.currentTime + 0.3);
            } else if (type === 'fail') {
                // Sonido grave de error
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(110, audioCtx.currentTime);
                osc.frequency.linearRampToValueAtTime(55, audioCtx.currentTime + 0.2);
                gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
                gain.gain.linearRampToValueAtTime(0.01, audioCtx.currentTime + 0.2);
                osc.start();
                osc.stop(audioCtx.currentTime + 0.2);
            }
        }

        function startGame() {
            score = 0;
            questionCount = 0;
            document.getElementById('start-screen').classList.add('hidden');
            document.getElementById('result-screen').classList.add('hidden');
            document.getElementById('game-screen').classList.remove('hidden');
            nextQuestion();
        }

        function nextQuestion() {
            if (questionCount >= totalQuestions) {
                endGame();
                return;
            }

            questionCount++;
            document.getElementById('current-q').innerText = questionCount;
            
            // Generar tablas variadas (de la del 2 a la del 12)
            let n1 = Math.floor(Math.random() * 11) + 2;
            let n2 = Math.floor(Math.random() * 11) + 2;
            currentAnswer = n1 * n2;

            document.getElementById('question').innerText = `${n1} x ${n2}`;
            const input = document.getElementById('answer-input');
            input.value = '';
            input.focus();
            
            document.getElementById('mate-robot').innerText = '🤖';
            document.getElementById('mate-robot').style.transform = 'translateY(-50%) scale(1)';

            startTimer();
        }

        function startTimer() {
            clearInterval(timerInterval);
            timeLeft = 100;
            const bar = document.getElementById('timer-bar');
            
            timerInterval = setInterval(() => {
                timeLeft -= 2; // Baja 2% cada 100ms = 5 segundos total
                bar.style.width = timeLeft + '%';
                
                if (timeLeft <= 0) {
                    clearInterval(timerInterval);
                    playSfx('fail');
                    document.getElementById('mate-robot').innerText = '😴';
                    setTimeout(nextQuestion, 800);
                }
            }, 100);
        }

        function checkAnswer() {
            clearInterval(timerInterval);
            const input = document.getElementById('answer-input');
            const userAns = parseInt(input.value);
            const mate = document.getElementById('mate-robot');

            if (userAns === currentAnswer) {
                score++;
                playSfx('success');
                mate.innerText = '⚡';
                mate.style.transform = 'translateY(-50%) scale(1.4) rotate(10deg)';
            } else {
                playSfx('fail');
                mate.innerText = '❌';
                mate.style.transform = 'translateY(-50%) scale(0.8)';
            }

            setTimeout(nextQuestion, 600);
        }

        function endGame() {
            document.getElementById('game-screen').classList.add('hidden');
            document.getElementById('result-screen').classList.remove('hidden');
            
            const percent = Math.round((score / totalQuestions) * 100);
            document.getElementById('percentage').innerText = percent + '%';
            document.getElementById('final-msg').innerText = `Acertaste ${score} de ${totalQuestions}`;
            
            const mate = document.getElementById('mate-robot');
            mate.innerText = percent === 100 ? '👑' : (percent >= 70 ? '😎' : '👨‍🔧');
        }

        function resetGame() {
            startGame();
        }

        // Permitir usar la tecla Enter para enviar
        document.getElementById('answer-input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') checkAnswer();
        });
    </script>
</body>
</html>
