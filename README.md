
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no" />
  <title>PulseCam — Realtime BPM via Phone Camera</title>
  <meta name="theme-color" content="#0ea5e9" />
  <style>
    :root {
      --bg: #0b1220;
      --card: #0f172a;
      --muted: #94a3b8;
      --text: #e2e8f0;
      --accent: #22d3ee;
      --good: #34d399;
      --warn: #fbbf24;
      --bad: #f87171;
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
      margin: 0; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial;
      background: radial-gradient(1200px 800px at 70% -10%, #10315955, transparent), var(--bg);
      color: var(--text);
    }
    .wrap { max-width: 960px; margin: 0 auto; padding: 16px; }
    header { display: flex; align-items: center; justify-content: space-between; gap: 12px; }
    h1 { font-size: clamp(20px, 3vw, 28px); margin: 8px 0; letter-spacing: 0.3px; }
    .sub { color: var(--muted); font-size: 14px; }
    .grid { display: grid; grid-template-columns: 1fr; gap: 12px; }
    @media (min-width: 820px) { .grid { grid-template-columns: 1.2fr 1fr; } }

    .card { background: linear-gradient(180deg, #0e172a, #0b1220); border: 1px solid #1f2b47; border-radius: 18px; box-shadow: 0 10px 30px rgba(0,0,0,0.25); }
    .card > .hd { display:flex; align-items:center; justify-content:space-between; padding: 14px 16px; border-bottom: 1px solid #1f2b47; }
    .card > .bd { padding: 14px 16px; }
    .btn {
      border: 1px solid #1f2b47; background: #0b2033; color: var(--text);
      border-radius: 10px; padding: 10px 14px; font-weight: 600; cursor: pointer;
      transition: all .2s; display: inline-flex; align-items: center; gap: 10px;
    }
    .btn:hover { transform: translateY(-1px); box-shadow: 0 6px 20px rgba(0,0,0,0.25); }
    .btn[disabled] { opacity: .6; cursor: not-allowed; filter: grayscale(.2); }
    .stat { display:flex; gap: 16px; flex-wrap: wrap; }
    .pill { background: #0b2033; border: 1px solid #1f2b47; border-radius: 999px; padding: 6px 10px; font-size: 12px; color: var(--muted); }

    #bpm { font-size: clamp(48px, 8vw, 76px); font-weight: 800; margin: 12px 0 4px; }
    #qual { font-size: 13px; color: var(--muted); }

    .row { display:flex; align-items:center; gap:10px; flex-wrap: wrap; }
    .row > * { flex: 1 1 auto; }

    canvas { width: 100%; height: 240px; background: #07101d; border-radius: 12px; border: 1px solid #1f2b47; display: block; }
    #video { position: absolute; width: 1px; height: 1px; opacity: 0; pointer-events:none; }

    .hint { font-size: 13px; color: var(--muted); line-height: 1.45; }
    .log { font-family: ui-monospace, Menlo, Consolas, monospace; font-size: 12px; color: #9fb3c8; max-height: 120px; overflow: auto; background: #07101d; border: 1px solid #1f2b47; border-radius: 10px; padding: 8px; }

    .badge { font-size: 11px; padding: 2px 8px; border-radius: 999px; border:1px solid #1f2b47; background:#0b2033; color:var(--muted); }
    .ok { color: var(--good); }
    .warn { color: var(--warn); }
    .bad { color: var(--bad); }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div>
        <h1>PulseCam <span class="badge">PPG</span></h1>
        <div class="sub">Place your fingertip gently on the rear camera. We'll analyze color changes to estimate your heart rate.</div>
      </div>
      <div class="row" style="justify-content:flex-end; max-width: 420px;">
        <button id="startBtn" class="btn">▶ Start</button>
        <button id="stopBtn" class="btn" disabled>■ Stop</button>
      </div>
    </header>

    <div class="grid" style="margin-top: 14px;">
      <section class="card">
        <div class="hd">
          <strong>Signal</strong>
          <div class="stat">
            <span class="pill" id="fr">FPS: —</span>
            <span class="pill" id="torchState">Torch: —</span>
            <span class="pill" id="coverage">Coverage: —</span>
          </div>
        </div>
        <div class="bd">
          <canvas id="wave"></canvas>
          <div class="hint" style="margin-top: 10px;">
            Tip: On Android Chrome, torch can be enabled automatically on supported devices. If torch fails, use an external light or press your finger a bit more gently.
          </div>
        </div>
      </section>

      <section class="card">
        <div class="hd"><strong>Heart Rate</strong><span class="pill" id="quality">Quality: —</span></div>
        <div class="bd">
          <div id="bpm">—</div>
          <div id="qual">Waiting for a stable signal…</div>
          <div class="hint" style="margin-top: 8px;">
            We compute BPM from peaks in a band‑passed red‑channel signal (0.7–3.3 Hz ~ 42–198 BPM) with outlier rejection.
          </div>
          <div class="log" id="log" style="margin-top: 10px;"></div>
        </div>
      </section>
    </div>

    <video id="video" playsinline muted></video>
  </div>

  <script>
    // --- Utilities ---
    const log = (...a) => { const el = document.getElementById('log'); el.textContent = [new Date().toLocaleTimeString(), ...a].join(' ') + '\n' + el.textContent; };
    const sleep = (ms) => new Promise(r=>setTimeout(r, ms));

    // --- State ---
    let stream, track, imageCapture, torchAvailable = false, torchOn = false;
    let rafId = null, procId = null;
    let ctx, cvs, v, drawCtx, drawCvs;
    let lastFrameTs = performance.now();

    // Signal buffers
    const BUFFER_SECS = 12;            // rolling window for analysis
    const TARGET_FPS = 30;             // analysis rate
    const MAX_SAMPLES = BUFFER_SECS * TARGET_FPS;
    let times = new Float64Array(MAX_SAMPLES);
    let samples = new Float32Array(MAX_SAMPLES);
    let wr = 0, count = 0;

    // Quality metrics
    function coverageMetric(frame, w, h) {
      // Estimate how much of the frame is covered by a finger by checking red dominance
      let redDominant = 0; const step = 4 * 8; // sample every 8 pixels
      for (let i=0; i<frame.data.length; i+=step) {
        const r = frame.data[i], g = frame.data[i+1], b = frame.data[i+2];
        if (r > g * 1.2 && r > b * 1.2) redDominant++;
      }
      const pct = redDominant / (frame.data.length/step);
      return pct; // 0..1
    }

    // Simple IIR filters for bandpass (highpass + lowpass)
    function makeOnePoleLP(alpha) { let y=0; return x=>{ y = y + alpha*(x - y); return y; }; }
    function makeOnePoleHP(alpha) { let y=0, prevX=0; return x=>{ const lp = y + alpha*(x - y); y = lp; const hp = x - lp; prevX = x; return hp; }; }

    // Peak detection with refractory period and dynamic threshold
    let lastPeakTime = 0; let dynThresh = 0.0;
    function detectPeaks(t, x) {
      // Update dynamic threshold as a fraction of signal RMS
      const rms = Math.sqrt(rollingRMS());
      dynThresh = 0.4 * rms; // 40% of RMS
      const minInterval = 0.35; // seconds (~171 BPM max)
      if (x > dynThresh && prevX <= dynThresh) {
        const dt = (t - lastPeakTime);
        if (dt > minInterval) { lastPeakTime = t; return true; }
      }
      return false;
    }
    let prevX = 0;

    // Rolling RMS over window (last 3s)
    function rollingRMS() {
      const secs = 3.0; const n = Math.min(count, Math.floor(secs*TARGET_FPS));
      if (n < 2) return 0.0001;
      let sum=0; for (let i=0;i<n;i++){ const idx=(wr - 1 - i + MAX_SAMPLES) % MAX_SAMPLES; const v=samples[idx]; sum += v*v; }
      return sum / n;
    }

    // Compute BPM from recent peaks (last 8s), with outlier rejection
    function bpmFromPeaks() {
      const window = 8.0; const now = times[(wr - 1 + MAX_SAMPLES) % MAX_SAMPLES] || 0;
      // collect peak times
      const peaks = [];
      let looking = false, last = -1, start = Math.max(0, count - Math.floor(window*TARGET_FPS));
      let localThresh = 0.35 * Math.sqrt(rollingRMS());
      for (let i = start; i < count; i++) {
        const idx = (wr - count + i + MAX_SAMPLES) % MAX_SAMPLES;
        const t = times[idx], x = samples[idx];
        if (!looking && x > localThresh && prevX <= localThresh) { looking = true; last = t; }
        if (looking && x < localThresh) { looking = false; if (last>0) peaks.push(last); last = -1; }
      }
      if (peaks.length < 2) return { bpm: NaN, rr: [] };
      const rr = [];
      for (let i=1;i<peaks.length;i++) { rr.push(peaks[i]-peaks[i-1]); }
      // reject outliers via IQR
      rr.sort((a,b)=>a-b);
      const q1 = rr[Math.floor(rr.length*0.25)], q3 = rr[Math.floor(rr.length*0.75)];
      const iqr = q3 - q1; const lo = Math.max(0.3, q1 - 1.5*iqr), hi = Math.min(2.0, q3 + 1.5*iqr);
      const filtered = rr.filter(v => v>=lo && v<=hi);
      if (filtered.length < 2) return { bpm: NaN, rr: rr };
      const meanRR = filtered.reduce((a,b)=>a+b,0)/filtered.length;
      const bpm = 60 / meanRR;
      return { bpm, rr: filtered };
    }

    // Drawing
    function draw() {
      const W = drawCvs.width, H = drawCvs.height; const ctx = drawCtx;
      ctx.clearRect(0,0,W,H);
      ctx.fillStyle = '#081325'; ctx.fillRect(0,0,W,H);
      ctx.strokeStyle = '#1f2b47'; ctx.lineWidth = 1; ctx.beginPath();
      for (let x=0; x<W; x+=Math.floor(W/12)) { ctx.moveTo(x,0); ctx.lineTo(x,H); }
      for (let y=0; y<H; y+=Math.floor(H/6)) { ctx.moveTo(0,y); ctx.lineTo(W,y); }
      ctx.stroke();

      // plot signal
      ctx.beginPath(); ctx.lineWidth = 2; ctx.strokeStyle = '#7dd3fc';
      const n = Math.min(count, MAX_SAMPLES);
      if (n>2) {
        const startIdx = (wr - n + MAX_SAMPLES) % MAX_SAMPLES;
        const t0 = times[startIdx];
        for (let i=0;i<n;i++) {
          const idx = (startIdx + i) % MAX_SAMPLES;
          const x = (times[idx] - t0) / BUFFER_SECS; // 0..1
          const y = samples[idx];
          const px = x * W; const py = H*0.5 - y * (H*0.35);
          if (i===0) ctx.moveTo(px, py); else ctx.lineTo(px, py);
        }
        ctx.stroke();
      }
      rafId = requestAnimationFrame(draw);
    }

    function updateStatus(id, text) { document.getElementById(id).textContent = text; }
    function setQuality(status, msg) {
      const el = document.getElementById('quality');
      el.textContent = `Quality: ${status}`;
      el.classList.remove('ok','warn','bad');
      el.classList.add(status==='Good'?'ok':status==='Fair'?'warn':'bad');
      document.getElementById('qual').textContent = msg;
    }

    async function start() {
      if (stream) return;
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          video: {
            facingMode: { ideal: 'environment' },
            width: { ideal: 640 }, height: { ideal: 480 },
            frameRate: { ideal: 60, max: 60 }
          }, audio: false
        });
        v.srcObject = stream; track = stream.getVideoTracks()[0]; await v.play();
        imageCapture = ('ImageCapture' in window) ? new ImageCapture(track) : null;
        torchAvailable = false;
        if (imageCapture && 'getPhotoCapabilities' in imageCapture) {
          try {
            const caps = await imageCapture.getPhotoCapabilities();
            torchAvailable = !!caps.torch || (track.getCapabilities && track.getCapabilities().torch);
          } catch (e) {
            // Some devices throw here; try track capabilities
            torchAvailable = track.getCapabilities && track.getCapabilities().torch;
          }
        } else if (track.getCapabilities) {
          torchAvailable = track.getCapabilities().torch;
        }
        updateStatus('torchState', torchAvailable ? 'Torch: available' : 'Torch: not available');
        if (torchAvailable) {
          try {
            await track.applyConstraints({ advanced: [{ torch: true }] });
            torchOn = true; updateStatus('torchState', 'Torch: on');
          } catch (e) { log('Torch enable failed:', e.message); }
        }
        runProcessing();
        document.getElementById('startBtn').disabled = true;
        document.getElementById('stopBtn').disabled = false;
      } catch (err) {
        log('Camera error:', err.message);
        alert('Could not access the camera. Please allow camera permissions and use HTTPS (or localhost).');
      }
    }

    function stop() {
      if (rafId) cancelAnimationFrame(rafId); rafId=null;
      if (procId) cancelAnimationFrame(procId); procId=null;
      if (stream) { stream.getTracks().forEach(t=>t.stop()); stream=null; }
      document.getElementById('startBtn').disabled = false;
      document.getElementById('stopBtn').disabled = true;
      updateStatus('fr', 'FPS: —'); updateStatus('coverage', 'Coverage: —'); updateStatus('torchState', torchOn?'Torch: on':'Torch: —');
      document.getElementById('bpm').textContent = '—';
      setQuality('Poor', 'Stopped. Tap Start to measure again.');
    }

    function initCanvas() {
      drawCvs = document.getElementById('wave'); drawCtx = drawCvs.getContext('2d');
      const dpr = Math.min(2, window.devicePixelRatio || 1);
      const rect = drawCvs.getBoundingClientRect();
      drawCvs.width = Math.floor(rect.width * dpr);
      drawCvs.height = Math.floor(rect.height * dpr);
      cvs = document.createElement('canvas'); ctx = cvs.getContext('2d', { willReadFrequently: true });
      cvs.width = 160; cvs.height = 120; // downsampled analysis res
    }

    function processFrame(ts) {
      if (!v.videoWidth) return;
      ctx.drawImage(v, 0, 0, cvs.width, cvs.height);
      const frame = ctx.getImageData(0, 0, cvs.width, cvs.height);

      // coverage quality
      const cov = coverageMetric(frame, cvs.width, cvs.height);
      updateStatus('coverage', `Coverage: ${(cov*100).toFixed(0)}%`);

      // compute mean red channel intensity
      let sumR=0; const data=frame.data; for (let i=0;i<data.length;i+=4){ sumR += data[i]; }
      const meanR = sumR / (data.length/4);

      // Bandpass via simple HP (0.5 Hz) + LP (4 Hz) IIR tuned to TARGET_FPS
      if (!processFrame.hp) {
        const hpAlpha = 1 - Math.exp(-2*Math.PI*0.5 / TARGET_FPS); // 0.5 Hz
        const lpAlpha = 1 - Math.exp(-2*Math.PI*4.0 / TARGET_FPS); // 4 Hz
        processFrame.hp = makeOnePoleHP(hpAlpha);
        processFrame.lp = makeOnePoleLP(lpAlpha);
      }
      const hp = processFrame.hp(meanR);
      const bp = processFrame.lp(hp);

      // Normalize
      const norm = (bp - (processFrame.mu = (processFrame.mu??bp) + 0.002*(bp - (processFrame.mu??bp)))) / (Math.sqrt(rollingRMS()) + 1e-6);

      const now = performance.now() / 1000;
      times[wr] = now; samples[wr] = norm; wr = (wr + 1) % MAX_SAMPLES; count = Math.min(count + 1, MAX_SAMPLES);

      // FPS
      const dt = now - (processFrame.lastTs || now); processFrame.lastTs = now;
      const fps = 1 / Math.max(dt, 1/120);
      updateStatus('fr', 'FPS: ' + fps.toFixed(0));

      // Quality feedback
      if (cov < 0.25) setQuality('Poor', 'Cover the camera fully with your fingertip.');
      else if (fps < 20) setQuality('Fair', 'Low frame rate. Close other apps or reduce motion.');
      else setQuality('Good', 'Hold steady. Breathe normally.');

      // BPM estimation every frame (uses 8s window internally)
      const { bpm } = bpmFromPeaks();
      if (!isNaN(bpm)) {
        const smooth = (processFrame.sBPM = (processFrame.sBPM ?? bpm) * 0.85 + bpm * 0.15);
        const bpmText = Math.round(smooth);
        document.getElementById('bpm').textContent = bpmText;
        if (bpmText < 40 || bpmText > 200) setQuality('Poor', 'Out-of-range reading. Adjust finger pressure and lighting.');
      }

      prevX = norm;
    }

    function runProcessing() {
      const loop = () => { processFrame(); procId = requestAnimationFrame(loop); };
      loop();
      draw();
    }

    // DOM init
    window.addEventListener('load', () => {
      v = document.getElementById('video');
      initCanvas();
      window.addEventListener('resize', initCanvas);
      document.getElementById('startBtn').addEventListener('click', start);
      document.getElementById('stopBtn').addEventListener('click', stop);
      setQuality('Poor', 'Tap Start, then place your fingertip gently over the rear camera.');
    });
  </script>
</body>
</html>
