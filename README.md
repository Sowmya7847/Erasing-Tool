<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Magic Eraser App</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: flex-start;
      margin: 0;
      padding: 0;
    }
    canvas, video, img {
      max-width: 100%;
      border: 1px solid #ccc;
      margin-top: 10px;
    }
    .controls, .mode-buttons {
      margin: 15px;
    }
    button {
      margin-right: 10px;
    }
    #preview {
      display: none;
    }
  </style>
</head>
<body>
  <h2>Magic Eraser Tool</h2>
  <div class="mode-buttons">
    <button onclick="chooseMode('upload')">Upload Image</button>
    <button onclick="chooseMode('ar')">Use AR Camera</button>
  </div>

  <div class="controls">
    <input type="file" id="fileInput" accept="image/*" style="display:none" />
    <video id="video" autoplay playsinline width="400" height="300" style="display:none;"></video>
    <button onclick="captureFromCamera()" id="captureBtn" style="display:none">Capture Photo</button>
    <canvas id="canvas" width="400" height="300"></canvas>
    <input type="range" min="5" max="50" value="20" id="brushSize" /> Brush Size
    <br />
    <button onclick="undo()">Undo</button>
    <button onclick="redo()">Redo</button>
    <button onclick="previewErased()">Preview</button>
    <button onclick="downloadImage()">Download</button>
  </div>

  <img id="preview" alt="Erased Preview" />

  <script>
    const canvas = document.getElementById("canvas");
    const ctx = canvas.getContext("2d");
    const brushSizeInput = document.getElementById("brushSize");
    const video = document.getElementById("video");
    const previewImg = document.getElementById("preview");

    let isErasing = false;
    let history = [];
    let redoStack = [];
    let mode = "upload";
    let lastX = null;
    let lastY = null;

    function chooseMode(selectedMode) {
      mode = selectedMode;
      if (mode === "upload") {
        document.getElementById("fileInput").click();
      } else {
        startCamera();
      }
    }

    document.getElementById("fileInput").addEventListener("change", (e) => {
      const file = e.target.files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = function (event) {
        const img = new Image();
        img.onload = function () {
          canvas.width = img.width;
          canvas.height = img.height;
          ctx.drawImage(img, 0, 0);
          saveState();
        };
        img.src = event.target.result;
      };
      reader.readAsDataURL(file);
    });

    function startCamera() {
      navigator.mediaDevices.getUserMedia({ video: true })
        .then((stream) => {
          video.srcObject = stream;
          video.style.display = "block";
          document.getElementById("captureBtn").style.display = "inline-block";
        });
    }

    function captureFromCamera() {
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      saveState();
      video.style.display = "none";
      document.getElementById("captureBtn").style.display = "none";
      const stream = video.srcObject;
      stream.getTracks().forEach(track => track.stop());
    }

    canvas.addEventListener("mousedown", (e) => {
      isErasing = true;
      const [x, y] = getCanvasCoords(e);
      lastX = x;
      lastY = y;
      ctx.globalCompositeOperation = "destination-out";
    });

    canvas.addEventListener("mousemove", (e) => {
      if (!isErasing) return;
      const [x, y] = getCanvasCoords(e);
      drawEraser(x, y);
      lastX = x;
      lastY = y;
    });

    canvas.addEventListener("mouseup", () => {
      isErasing = false;
      lastX = lastY = null;
      ctx.globalCompositeOperation = "source-over";
      saveState();
    });

    canvas.addEventListener("mouseleave", () => {
      isErasing = false;
      lastX = lastY = null;
      ctx.globalCompositeOperation = "source-over";
    });

    function getCanvasCoords(e) {
      const rect = canvas.getBoundingClientRect();
      return [e.clientX - rect.left, e.clientY - rect.top];
    }

    function drawEraser(x, y) {
      const size = parseInt(brushSizeInput.value);
      ctx.lineJoin = "round";
      ctx.lineCap = "round";
      ctx.lineWidth = size;
      ctx.beginPath();
      ctx.moveTo(lastX ?? x, lastY ?? y);
      ctx.lineTo(x, y);
      ctx.stroke();
    }

    function saveState() {
      history.push(canvas.toDataURL());
      redoStack = [];
    }

    function undo() {
      if (history.length > 1) {
        redoStack.push(history.pop());
        const img = new Image();
        img.onload = () => ctx.drawImage(img, 0, 0);
        img.src = history[history.length - 1];
      }
    }

    function redo() {
      if (redoStack.length > 0) {
        const next = redoStack.pop();
        history.push(next);
        const img = new Image();
        img.onload = () => ctx.drawImage(img, 0, 0);
        img.src = next;
      }
    }

    function previewErased() {
      previewImg.src = canvas.toDataURL("image/png");
      previewImg.style.display = "block";
    }

    function downloadImage() {
      const link = document.createElement("a");
      link.download = "erased-image.png";
      link.href = canvas.toDataURL();
      link.click();
    }
  </script>
</body>
</html>
