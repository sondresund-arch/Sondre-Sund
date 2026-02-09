<!DOCTYPE html>
<html lang="no">
<head>
    <meta charset="UTF-8">
    <title>Gemedasj – Avansert Spill</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            background: #222;
            color: white;
        }

        #gameArea {
            width: 600px;
            height: 400px;
            background: #333;
            margin: 20px auto;
            position: relative;
            border: 3px solid white;
            overflow: hidden;
        }

        #target {
            width: 50px;
            height: 50px;
            background: red;
            border-radius: 50%;
            position: absolute;
            cursor: pointer;
        }

        #startBtn {
            padding: 10px 20px;
            font-size: 18px;
            cursor: pointer;
            margin-top: 20px;
        }
    </style>
</head>
<body>

    <h1>Gemedasj – Klikk så fort du kan!</h1>
    <p>Poeng: <span id="score">0</span></p>
    <p>Liv: <span id="lives">3</span></p>

    <div id="gameArea">
        <div id="target"></div>
    </div>

    <button id="startBtn">Start spillet</button>

    <audio id="clickSound">
        <source src="https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg" type="audio/ogg">
    </audio>

    <script>
        const target = document.getElementById("target");
        const gameArea = document.getElementById("gameArea");
        const scoreDisplay = document.getElementById("score");
        const livesDisplay = document.getElementById("lives");
        const startBtn = document.getElementById("startBtn");
        const clickSound = document.getElementById("clickSound");

        let score = 0;
        let lives = 3;
        let speed = 1200;
        let gameRunning = false;

        function moveTarget() {
            if (!gameRunning) return;

            const maxX = gameArea.clientWidth - target.clientWidth;
            const maxY = gameArea.clientHeight - target.clientHeight;

            const x = Math.random() * maxX;
            const y = Math.random() * maxY;

            target.style.left = x + "px";
            target.style.top = y + "px";

            lives--;
            livesDisplay.textContent = lives;

            if (lives <= 0) {
                gameOver();
                return;
            }

            setTimeout(moveTarget, speed);
        }

        target.addEventListener("click", () => {
            if (!gameRunning) return;

            clickSound.play();
            score++;
            scoreDisplay.textContent = score;

            if (speed > 300) {
                speed -= 40;
            }

            moveTarget();
        });

        function startGame() {
            score = 0;
            lives = 3;
            speed = 1200;
            gameRunning = true;

            scoreDisplay.textContent = score;
            livesDisplay.textContent = lives;

            moveTarget();
        }

        function gameOver() {
            gameRunning = false;
            alert("Game Over! Du fikk " + score + " poeng.");
        }

        startBtn.addEventListener("click", startGame);
    </script>

</body>
</html>
