<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>ตรวจจับการนั่ง (Smart Posture)</title>

  <!-- Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <!-- TensorFlow -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/pose@0.8/dist/teachablemachine-pose.min.js"></script>
  <!-- Firebase -->
  <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-database-compat.js"></script>

  <style>
    body {
      font-family: 'Arial', sans-serif;
      margin: 0;
      padding: 20px;
      background: #f0f4f8;
      color: #333;
    }

    h1 {
      text-align: center;
      color: #2c3e50;
    }

    .controls {
      text-align: center;
      margin-bottom: 20px;
    }

    button {
      margin: 5px;
      padding: 10px 20px;
      font-size: 1rem;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      transition: 0.2s ease;
    }

    button:hover {
      transform: scale(1.05);
    }

    canvas {
      display: block;
      margin: 0 auto;
      border: 2px solid #3498db;
      border-radius: 10px;
    }

    #label-container {
      text-align: center;
      margin-top: 15px;
    }

    #alert-box {
      position: fixed;
      bottom: 20px;
      right: 20px;
      background-color: #e74c3c;
      color: white;
      padding: 12px 20px;
      border-radius: 8px;
      display: none;
      animation: fadeInOut 4s ease-in-out;
    }

    @keyframes fadeInOut {
      0% { opacity: 0; transform: translateY(20px); }
      10% { opacity: 1; transform: translateY(0); }
      90% { opacity: 1; transform: translateY(0); }
      100% { opacity: 0; transform: translateY(20px); }
    }

    #chart {
      max-width: 600px;
      margin: 30px auto;
    }

    .options {
      text-align: center;
      margin-top: 10px;
    }

    .options label {
      font-size: 0.95rem;
    }
  </style>
</head>
<body>
  <h1>ระบบตรวจจับการนั่ง</h1>

  <div class="controls">
    <button style="background: #27ae60; color: white;" onclick="init()">Start</button>
    <button style="background: #c0392b; color: white;" onclick="stop()">Stop</button>
    <button style="background: #f39c12; color: white;" onclick="reset()">Reset</button>
  </div>

  <canvas id="canvas" width="400" height="400"></canvas>
  <div id="label-container"></div>

  <div class="options">
    <label><input type="checkbox" id="soundToggle" checked /> เปิดเสียงแจ้งเตือน</label>
  </div>

  <div id="alert-box"></div>
  <audio id="alertSound" src="warning.mp3" preload="auto"></audio>

  <canvas id="chart"></canvas>

  <script>
    // 🔥 ใส่ Firebase config ของคุณที่นี่
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      databaseURL: "https://YOUR_PROJECT_ID.firebaseio.com",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT_ID.appspot.com",
      messagingSenderId: "XXXXXX",
      appId: "YOUR_APP_ID"
    };

    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    const URL = "https://teachablemachine.withgoogle.com/models/xtgDT3JxI/";
    let model, webcam, ctx, labelContainer, maxPredictions;
    let isRunning = false;

    async function init() {
      const modelURL = URL + "model.json";
      const metadataURL = URL + "metadata.json";

      model = await tmPose.load(modelURL, metadataURL);
      maxPredictions = model.getTotalClasses();

      webcam = new tmPose.Webcam(400, 400, true);
      await webcam.setup();
      await webcam.play();
      isRunning = true;
      window.requestAnimationFrame(loop);

      const canvas = document.getElementById("canvas");
      ctx = canvas.getContext("2d");
      labelContainer = document.getElementById("label-container");
    }

    async function loop(timestamp) {
      if (!isRunning) return;
      webcam.update();
      await predict();
      window.requestAnimationFrame(loop);
    }

    async function predict() {
      const { pose, posenetOutput } = await model.estimatePose(webcam.canvas);
      const prediction = await model.predict(posenetOutput);

      let showWarning = false;
      let outputHTML = "";

      for (let i = 0; i < maxPredictions; i++) {
        let className = prediction[i].className;
        const prob = prediction[i].probability.toFixed(2);

        if (className === "Class 1") {
          className = "นั่งถูกหลัก";
          outputHTML += `<div style="color: green">${className}: ${(prob * 100).toFixed(0)}%</div>`;
        } else if (className === "Class 2") {
          className = "นั่งผิดหลัก";
          outputHTML += `<div style="color: red">${className}: ${(prob * 100).toFixed(0)}%</div>`;

          if (prob > 0.5) {
            showWarning = true;
            logBadPosture();
            logToFirebase();
          }
        }
      }

      labelContainer.innerHTML = outputHTML;
      drawPose(pose);

      if (showWarning) {
        showAlert("คุณนั่งผิดหลักการยศาสตร์");
      }
    }

    function drawPose(pose) {
      if (webcam.canvas) {
        ctx.drawImage(webcam.canvas, 0, 0);
        if (pose) {
          const minConfidence = 0.5;
          tmPose.drawKeypoints(pose.keypoints, minConfidence, ctx);
          tmPose.drawSkeleton(pose.keypoints, minConfidence, ctx);
        }
      }
    }

    function stop() {
      isRunning = false;
      webcam.stop();
    }

    function reset() {
      localStorage.removeItem("badPostureLogs");
      location.reload();
    }

    function showAlert(message) {
      const alertBox = document.getElementById("alert-box");
      alertBox.innerText = message;
      alertBox.style.display = "block";

      const soundOn = document.getElementById("soundToggle").checked;
      if (soundOn) {
        document.getElementById("alertSound").play();
      }

      setTimeout(() => {
        alertBox.style.display = "none";
      }, 4000);
    }

    function logBadPosture() {
      const logs = JSON.parse(localStorage.getItem("badPostureLogs")) || [];
      logs.push({ time: new Date().toISOString() });
      localStorage.setItem("badPostureLogs", JSON.stringify(logs));
    }

    function logToFirebase() {
      const entry = { time: new Date().toISOString(), status: "bad" };
      db.ref("posture_logs").push(entry);
    }

    function renderChart() {
      const logs = JSON.parse(localStorage.getItem("badPostureLogs")) || [];
      const counts = {};

      logs.forEach(log => {
        const day = new Date(log.time).toLocaleDateString();
        counts[day] = (counts[day] || 0) + 1;
      });

      const labels = Object.keys(counts);
      const data = Object.values(counts);

      new Chart(document.getElementById("chart"), {
        type: 'bar',
        data: {
          labels: labels,
          datasets: [{
            label: 'จำนวนครั้งที่นั่งผิด',
            data: data,
            backgroundColor: '#e74c3c'
          }]
        },
        options: {
          responsive: true,
          scales: {
            y: { beginAtZero: true }
          }
        }
      });
    }

    window.onload = () => {
      renderChart();
    };
  </script>
</body>
</html>
