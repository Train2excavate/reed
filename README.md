<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Dr. Evelyn Reed — Offline Assistant (Chat + Calls)</title>
<style>
  :root{
    --bg:#f2f2f7;
    --card:#fff;
    --accent:#007aff;
    --muted:#e5e5ea;
    --glass: rgba(255,255,255,0.6);
  }
  html,body{height:100%;margin:0;background:var(--bg);font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif}
  .app-wrap{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:20px;box-sizing:border-box}
  .app{width:100%;max-width:460px;height:92vh;background:var(--card);border-radius:20px;box-shadow:0 14px 40px rgba(0,0,0,0.12);display:flex;flex-direction:column;overflow:hidden}
  header{background:linear-gradient(135deg,#4facfe,#00f2fe);color:#fff;padding:16px 18px;font-weight:800;font-size:1.15rem;text-align:center}
  main{flex:1;display:flex;flex-direction:column;background:var(--bg);overflow:hidden}
  #chat{flex:1;overflow:auto;padding:16px;display:flex;flex-direction:column;gap:10px}
  .bubble{max-width:78%;padding:10px 14px;border-radius:18px;box-shadow:0 2px 6px rgba(0,0,0,0.06);line-height:1.35}
  .you{align-self:flex-end;background:var(--accent);color:#fff;border-bottom-right-radius:6px}
  .evel{align-self:flex-start;background:var(--muted);color:#000;border-bottom-left-radius:6px}
  .typing{font-style:italic;color:#666;padding:8px 12px;border-radius:12px;align-self:flex-start}
  .composer{display:flex;gap:8px;padding:12px;border-top:1px solid #e6e6e6;background:#fff}
  .input{flex:1;padding:12px 14px;border-radius:20px;border:1px solid #d0d0d0;font-size:15px;outline:none}
  .btn{padding:10px 12px;border-radius:16px;border:0;color:#fff;font-weight:700;cursor:pointer}
  .btn.send{background:var(--accent)}
  .btn.audio{background:#34c759}
  .btn.video{background:#ff9500}
  .controls-row{display:flex;gap:8px;padding:8px;justify-content:center;background:transparent}
  .smallLink{background:transparent;border:0;color:var(--accent);font-weight:700;cursor:pointer;padding:6px}
  /* modal */
  .modal{position:fixed;inset:0;display:none;align-items:center;justify-content:center;background:rgba(0,0,0,0.55);z-index:999}
  .callCard{width:94%;max-width:540px;background:#fff;border-radius:14px;padding:14px;display:flex;flex-direction:column;gap:10px;align-items:center;box-shadow:0 8px 30px rgba(0,0,0,0.18)}
  #userCam{width:100%;height:260px;background:#000;border-radius:10px;object-fit:cover}
  #emoji{font-size:5.6rem;transform-origin:center bottom;transition:transform .06s,filter .12s}
  .callTop{width:100%;display:flex;justify-content:space-between;align-items:center}
  .micStatus{display:flex;gap:8px;align-items:center}
  .micDot{width:10px;height:10px;border-radius:50%;background:#ffcc00;box-shadow:0 0 8px rgba(0,0,0,0.08)}
  .timer{font-weight:700}
  .callBtns{display:flex;gap:10px;margin-top:6px}
  .smallBtn{padding:10px 14px;border-radius:12px;border:0;font-weight:800;cursor:pointer}
  .muteBtn{background:#ffcc00;color:#000}
  .endBtn{background:#ff3b30}
  .info{font-size:13px;color:#666;text-align:center}
  /* responsive */
  @media (max-width:420px){
    #emoji{font-size:5rem}
    #userCam{height:200px}
  }
</style>
</head>
<body>
<div class="app-wrap">
  <div class="app" role="application" aria-label="Dr Evelyn Reed assistant">
    <header>Dr. Evelyn Reed</header>
    <main>
      <div id="chat" aria-live="polite"></div>
    </main>

    <div class="composer" role="region" aria-label="Composer">
      <input id="txt" class="input" type="text" placeholder="Type a message..." autocomplete="off"/>
      <button id="sendBtn" class="btn send">Send</button>
      <button id="audioBtn" class="btn audio">Audio</button>
      <button id="videoBtn" class="btn video">Video</button>
    </div>

    <div class="controls-row">
      <button id="exportBtn" class="smallLink">Export chat</button>
      <button id="clearBtn" class="smallLink">Clear chat</button>
      <span style="flex:1"></span>
      <span style="font-size:13px;color:#666;padding-right:12px">Offline-first · Gemini optional</span>
    </div>
  </div>
</div>

<!-- Call modal -->
<div id="callModal" class="modal" aria-hidden="true">
  <div class="callCard" role="dialog" aria-label="Call dialog">
    <div class="callTop">
      <div class="micStatus"><div id="micDot" class="micDot"></div><div id="micText">Mic: off</div></div>
      <div class="timer" id="callTimer">00:00</div>
    </div>

    <video id="userCam" autoplay muted playsinline></video>
    <div id="emoji">👩‍🔬</div>

    <div class="callBtns">
      <button id="muteBtn" class="smallBtn muteBtn">Mute</button>
      <button id="endBtn" class="smallBtn endBtn">End</button>
    </div>
    <div class="info" id="callHint">Evelyn will listen while you speak and reply with TTS.</div>
  </div>
</div>

<script>
/* =============================
   Config / Storage / DOM
   ============================= */
const GEMINI_KEY = "AIzaSyAiR5A6Wf1ji9SPWwVlFAVNUtKyGIQ560A"; // Gemini API key (use only when online)
const GEMINI_URL = "https://generativelanguage.googleapis.com/v1beta2/models/text-bison-001:generate";

const chatEl = document.getElementById('chat');
const txt = document.getElementById('txt');
const sendBtn = document.getElementById('sendBtn');
const audioBtn = document.getElementById('audioBtn');
const videoBtn = document.getElementById('videoBtn');
const exportBtn = document.getElementById('exportBtn');
const clearBtn = document.getElementById('clearBtn');

const callModal = document.getElementById('callModal');
const userCam = document.getElementById('userCam');
const emojiEl = document.getElementById('emoji');
const micDot = document.getElementById('micDot');
const micText = document.getElementById('micText');
const callTimerEl = document.getElementById('callTimer');
const muteBtn = document.getElementById('muteBtn');
const endBtn = document.getElementById('endBtn');
const callHint = document.getElementById('callHint');

const STORAGE_KEY = "drEvelynChatV2";

/* load saved messages */
let saved = [];
try { saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || "[]"); } catch(e){ saved = []; }
if(Array.isArray(saved)) saved.forEach(m => appendBubble(m.sender, m.text));

/* helper append bubble */
function appendBubble(sender, text, opts = {}) {
  const d = document.createElement('div');
  d.className = "bubble " + (sender === "You" ? "you" : "evel");
  if(opts.typing) d.classList.add('typing');
  d.textContent = text;
  chatEl.appendChild(d);
  chatEl.scrollTop = chatEl.scrollHeight;
  if(!opts.temporary) {
    saved.push({sender, text, t: Date.now()});
    try { localStorage.setItem(STORAGE_KEY, JSON.stringify(saved)); } catch(e){}
  }
  return d;
}

/* typing helper */
function showTyping(text = "Evelyn is typing...") {
  return appendBubble("Evelyn", text, {typing: true, temporary: true});
}

/* simple clear/export */
exportBtn.addEventListener('click', () => {
  const data = JSON.stringify(saved, null, 2);
  const blob = new Blob([data], {type: 'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'dr_evelyn_chat.json';
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);
});
clearBtn.addEventListener('click', () => {
  if(!confirm("Clear saved chat?")) return;
  saved = [];
  localStorage.removeItem(STORAGE_KEY);
  chatEl.innerHTML = "";
});

/* =============================
   Emotion / Offline Response Engine
   ============================= */

let evelynMood = "neutral";
let moodCounter = 0;

function updateMood(userMsg) {
  moodCounter++;
  if(!userMsg) {
    if(moodCounter >= 12 + Math.floor(Math.random()*8)) {
      const moods = ["supportive","scientist","playful","calm","sass","neutral"];
      evelynMood = moods[Math.floor(Math.random()*moods.length)];
      moodCounter = 0;
    }
    // tiny chance crystal
    if(Math.random() < 0.002) evelynMood = "crystal";
    return;
  }
  const msg = userMsg.toLowerCase();
  if(msg.includes("sad")||msg.includes("upset")||msg.includes("hurt")) evelynMood = "supportive";
  else if(msg.includes("explain")||msg.includes("why")||msg.includes("how")||msg.includes("science")) evelynMood = "scientist";
  else if(msg.includes("lol")||msg.includes("funny")||msg.includes("meme")) evelynMood = "playful";
  else if(msg.includes("bruh")||msg.includes("bro")||msg.includes("nah")) evelynMood = "sass";
  else if(msg.includes("calm")||msg.includes("relax")) evelynMood = "calm";
  else if(Math.random() < 0.01) evelynMood = "neutral";
  // tiny chance crystal
  if(Math.random() < 0.001) evelynMood = "crystal";
}

/* This offlineResponse keeps ALL original mock replies (preserved exactly)
   plus many new responses. It's safe (no syntax errors). */
function offlineResponse(userMsg) {
  updateMood(userMsg);

  // keep original lines exactly as requested plus many more
  const originals = [
    "I see, that's interesting.",
    "Can you tell me more about that?",
    "Thanks for sharing — let's explore it.",
    "Hmm, I understand. How does that make you feel?",
    "That's useful info. Could you elaborate a bit more?",
    "I appreciate you telling me that. Let's continue.",
    "That's really cool!",
    "6 7",
    "ohio",
    "Tell me more about it",
    "idk bro",
    "unfortunately I can't give an answer sry",
    "4 1",
    "I'm busy right now please wait",
    "I'm so sorry to hear that 😢",
    "That's great to hear 😊",
    "Awesome, can you talk more about it?",
    "W 😎",
    "Bro don't expect me to always give an answer",
    "Did you know that I am an offline AI model called CrystalAI and was developed by Shahir Hossen (this is a rare message)",
    "Error, please try again",
    "I am a message",
    "Cool, tell me more!",
    "Let's talk about this in a video chat"
  ];

  const supportive = [
    "I’m really glad you told me that.",
    "That must have taken courage to share.",
    "I'm here for you, truly.",
    "That sounds tough… but you’re handling it better than you think.",
    "It’s okay to feel that way."
  ];

  const scientist = [
    "From an analytical standpoint, that’s fascinating.",
    "Let me break this down clearly.",
    "Interesting variable — tell me more about that factor.",
    "Hypothetically, we can interpret this in multiple ways.",
    "That's actually consistent with observed patterns."
  ];

  const playful = [
    "LOL that’s actually funny.",
    "Bro you just gave me a whole side quest 😂",
    "No way you just said that 💀",
    "That’s a W take tbh",
    "Real, valid, certified W energy."
  ];

  const calm = [
    "Let’s slow down and breathe.",
    "Take your time — I’m listening.",
    "Alright, let's approach this gently.",
    "You don’t need to rush. I’m here."
  ];

  const sass = [
    "Be so for real right now 😭",
    "I’m not buying that, try again 💀",
    "Bro… what?",
    "Let’s be honest for a second here."
  ];

  const crystal = [
    "CrystalAI core awakening…",
    "Signal unstable… recalibrating.",
    "I see all pathways at once.",
    "Neural echo detected."
  ];

  // combine banks; always include some original flavor
  let bank = originals.concat([
    "I’m listening. Go on.",
    "That actually makes a lot of sense.",
    "Huh, that’s surprisingly interesting.",
    "Okay, now I’m curious — keep going.",
    "Oh wow—didn’t expect that!",
    "If that's how you feel, it's valid.",
    "I’m here for it. Tell me more.",
    "Alright, let's break this down together.",
    "That’s a unique perspective!",
    "I get what you mean."
  ]);

  // mood-specific insertion
  if(evelynMood === "supportive") bank = supportive.concat(bank);
  if(evelynMood === "scientist") bank = scientist.concat(bank);
  if(evelynMood === "playful") bank = playful.concat(bank);
  if(evelynMood === "calm") bank = calm.concat(bank);
  if(evelynMood === "sass") bank = sass.concat(bank);
  if(evelynMood === "crystal") bank = crystal.concat(bank);

  // pick 1-3 responses combined
  const amount = 1 + Math.floor(Math.random()*3);
  const out = [];
  for(let i=0;i<amount;i++){
    out.push(bank[Math.floor(Math.random()*bank.length)]);
  }
  return out.join(" ");
}

/* =============================
   Gemini (optional) wrapper
   ============================= */
async function callGemini(prompt) {
  if(!GEMINI_KEY) return offlineResponse(prompt);
  try {
    const res = await fetch(GEMINI_URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer " + GEMINI_KEY
      },
      body: JSON.stringify({
        // Using the 'prompt' field for straightforward compatibility
        prompt: prompt,
        temperature: 0.7,
        candidate_count: 1,
        max_output_tokens: 300
      })
    });
    if(!res.ok) throw new Error("Gemini returned " + res.status);
    const data = await res.json();
    // robust parsing
    if(data && data.candidates && data.candidates[0] && data.candidates[0].content) return data.candidates[0].content.trim();
    if(data && data.output && data.output[0] && data.output[0].content) return data.output[0].content.trim();
    return offlineResponse(prompt);
  } catch(e) {
    console.warn("Gemini failed, using offline:", e);
    return offlineResponse(prompt);
  }
}

/* =============================
   Chat send flow (text only)
   ============================= */
sendBtn.addEventListener('click', async () => {
  const v = txt.value && txt.value.trim();
  if(!v) return;
  txt.value = "";
  appendBubble("You", v);
  const tnode = showTyping("Evelyn is typing...");
  // prefer Gemini but fallback to offline
  const reply = await callGemini(v);
  if(tnode && tnode.parentNode) tnode.parentNode.removeChild(tnode);
  appendBubble("Evelyn", reply);
});

/* Enter to send */
txt.addEventListener('keypress', (e) => { if(e.key === 'Enter') sendBtn.click(); });

/* =============================
   TTS utilities (mood-based)
   ============================= */
function pickVoiceByMood(mood) {
  const voices = speechSynthesis.getVoices();
  if(!voices || !voices.length) return null;
  // heuristics: prefer female / en voices
  const normalized = (name) => name && name.toLowerCase();
  // mood mapping -> pitch/rate -> pick any voice matching en
  let voice = voices.find(v => /female|zira|samantha|amy|alloy|alloy/i.test(normalized(v.name))) || voices.find(v => v.lang && v.lang.startsWith("en")) || voices[0];
  return voice;
}
function speakWithMood(text, mood) {
  return new Promise((resolve) => {
    if(!('speechSynthesis' in window)) { resolve(); return; }
    // pause existing
    speechSynthesis.cancel();
    const u = new SpeechSynthesisUtterance(text);
    const v = pickVoiceByMood(mood);
    if(v) u.voice = v;
    // tune
    if(mood === "supportive") { u.pitch = 1.15; u.rate = 0.98; }
    else if(mood === "scientist") { u.pitch = 1.05; u.rate = 1.05; }
    else if(mood === "playful") { u.pitch = 1.35; u.rate = 1.08; }
    else if(mood === "calm") { u.pitch = 0.95; u.rate = 0.9; }
    else if(mood === "sass") { u.pitch = 1.25; u.rate = 1.02; }
    else if(mood === "crystal") { u.pitch = 0.8; u.rate = 0.95; }
    else { u.pitch = 1.1; u.rate = 1.0; }
    u.onstart = () => { animateEmojiSpeaking(text.length); };
    u.onend = () => { setTimeout(()=>{ emojiEl.style.transform = 'scaleY(1) rotate(0deg)'; resolve(); }, 80); };
    window.speechSynthesis.speak(u);
  });
}

/* Emoji mouth animation (used for both TTS and VAD) */
function animateEmojiSpeaking(length) {
  const frames = Math.max(8, Math.floor(length / 6));
  let i = 0;
  const id = setInterval(() => {
    emojiEl.style.transform = `scaleY(${1 + 0.14 * Math.sin(i * 0.5)}) rotate(${Math.sin(i * 0.12) * 3}deg)`;
    i++;
    if(i > frames) { clearInterval(id); emojiEl.style.transform = 'scaleY(1) rotate(0deg)'; }
  }, 50);
}

/* =============================
   Audio / Video Call Logic
   - STT: uses Web SpeechRecognition (if available)
   - Fallback: VAD (analyser) to detect end of utterance, then uses offline response
   - While TTS speaks, listening is paused to avoid feedback
   ============================= */

let localStream = null;
let audioCtx = null;
let analyser = null;
let dataArray = null;
let vadRaf = null;
let recognition = null;
let inCall = false;
let callStartTs = null;
let callTimerInterval = null;
let isMuted = false;
let callMode = null; // 'audio' or 'video'

function startTimer() {
  callStartTs = Date.now();
  callTimerInterval = setInterval(() => {
    const s = Math.floor((Date.now() - callStartTs) / 1000);
    const mm = String(Math.floor(s/60)).padStart(2,'0');
    const ss = String(s%60).padStart(2,'0');
    callTimerEl.textContent = `${mm}:${ss}`;
  }, 300);
}
function stopTimer() {
  clearInterval(callTimerInterval);
  callTimerEl.textContent = "00:00";
}

/* start local media and analyser */
async function acquireMedia(needsVideo=false) {
  try {
    localStream = await navigator.mediaDevices.getUserMedia({audio:true, video: needsVideo});
    if(needsVideo) userCam.srcObject = localStream;
    else userCam.srcObject = null;
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const src = audioCtx.createMediaStreamSource(localStream);
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048;
    src.connect(analyser);
    const bufferLen = analyser.fftSize;
    dataArray = new Uint8Array(bufferLen);
    return true;
  } catch(e) {
    console.warn("media failed", e);
    return false;
  }
}

/* VAD-based loop: detect speech + end-of-speech */
let hadSpeech = false;
let silenceStart = 0;
function startVADLoop() {
  if(!analyser) return;
  vadRaf = requestAnimationFrame(vadStep);
  function vadStep() {
    analyser.getByteTimeDomainData(dataArray);
    let sum = 0;
    for(let i=0;i<dataArray.length;i++) sum += Math.abs(dataArray[i] - 128);
    const avg = sum / dataArray.length;
    // thresholds tuned to be conservative
    if(avg > 12) {
      hadSpeech = true;
      silenceStart = 0;
      micDot.style.background = "#34c759";
      micText.textContent = "Mic: speaking";
    } else {
      if(hadSpeech && !silenceStart) silenceStart = Date.now();
      micDot.style.background = "#ffcc00";
      micText.textContent = "Mic: listening";
    }
    if(hadSpeech && silenceStart && (Date.now() - silenceStart) > 600) {
      // end of utterance
      hadSpeech = false;
      silenceStart = 0;
      pauseListening(); // avoid feedback
      onUtteranceDetected(null).finally(() => {
        resumeListening();
      });
    }
    vadRaf = requestAnimationFrame(vadStep);
  }
}

/* setup SpeechRecognition if available */
function setupRecognition() {
  const S = window.SpeechRecognition || window.webkitSpeechRecognition || null;
  if(!S) return null;
  try {
    const r = new S();
    r.continuous = true;
    r.interimResults = false;
    r.lang = "en-US";
    r.onstart = () => { console.log("recognition started"); }
    r.onend = () => { console.log("recognition ended"); if(inCall) try { r.start(); } catch(e){} }
    r.onerror = (e) => { console.warn("recognition error", e); }
    r.onresult = async (ev) => {
      let transcript = "";
      for(let i = ev.resultIndex; i < ev.results.length; i++){
        if(ev.results[i].isFinal) transcript += ev.results[i][0].transcript + " ";
      }
      transcript = transcript.trim();
      if(transcript) {
        pauseListening();
        await onUtteranceDetected(transcript);
        resumeListening();
      }
    };
    return r;
  } catch(e) {
    console.warn("recognition setup failed", e);
    return null;
  }
}

/* pause/resume listening utilities */
function pauseListening() {
  if(recognition && recognition.abort) try { recognition.abort(); } catch(e) {}
  if(vadRaf) { cancelAnimationFrame(vadRaf); vadRaf = null; }
}
function resumeListening() {
  if(recognition && recognition.start) try { recognition.start(); } catch(e) {}
  if(analyser && !vadRaf) startVADLoop();
}

/* handle utterance: transcript may be null */
async function onUtteranceDetected(transcript) {
  // display user's utterance
  if(transcript) appendBubble("You", transcript);
  else appendBubble("You", "[voice message]");
  const tnode = showTyping("Evelyn is thinking...");
  const prompt = transcript || "[user spoke - no transcript]";
  const reply = await callGemini(prompt);
  if(tnode && tnode.parentNode) tnode.parentNode.removeChild(tnode);
  appendBubble("Evelyn", reply);
  // TTS speak with mood
  await speakWithMood(reply, evelynMood);
}

/* start call */
async function startCall(mode) {
  callMode = mode; inCall = true;
  callModal.style.display = "flex";
  callHint.textContent = mode === "video" ? "Video call — Evelyn will listen and reply." : "Audio call — Evelyn will listen and reply.";
  const ok = await acquireMedia(mode === "video");
  if(!ok) { alert("Microphone / camera access required."); endCall(); return; }
  micDot.style.background = "#ffcc00"; micText.textContent = "Mic: listening";
  startTimer();
  recognition = setupRecognition();
  if(recognition) {
    try { recognition.start(); } catch(e){}
  } else {
    startVADLoop();
  }
}

/* end call */
function endCall() {
  pauseListening();
  if(localStream) {
    try { localStream.getTracks().forEach(t=>t.stop()); } catch(e){}
    localStream = null;
  }
  if(audioCtx) { try{ audioCtx.close(); }catch(e){} audioCtx = null; analyser = null; dataArray = null; }
  if(recognition && recognition.abort) try { recognition.abort(); } catch(e){}
  inCall = false;
  callModal.style.display = "none";
  stopTimer();
  emojiEl.style.transform = "scaleY(1) rotate(0deg)";
  micDot.style.background = "#ffcc00"; micText.textContent = "Mic: off";
}

/* toggle mute */
function toggleMute() {
  if(!localStream) return;
  isMuted = !isMuted;
  localStream.getAudioTracks().forEach(t => t.enabled = !isMuted);
  muteBtn.textContent = isMuted ? "Unmute" : "Mute";
}

/* bindings */
audioBtn.addEventListener('click', () => startCall('audio'));
videoBtn.addEventListener('click', () => startCall('video'));
endBtn.addEventListener('click', endCall);
muteBtn.addEventListener('click', toggleMute);

/* small quick audio-button behavior */
audioBtn.addEventListener('dblclick', async () => {
  // double-click -> single-turn TTS message without full call
  const reply = "Hello! This is Evelyn. You can speak now and I will respond.";
  appendBubble("Evelyn", reply);
  await speakWithMood(reply, evelynMood);
});

/* show recognition capability in console */
(function checkRecog(){
  const S = window.SpeechRecognition || window.webkitSpeechRecognition || null;
  if(S) console.log("SpeechRecognition available (may require permissions)");
  else console.log("SpeechRecognition not available; using VAD fallback (offline).");
})();

/* helper: when page unload, stop audio */
window.addEventListener('beforeunload', () => {
  if(localStream) { try { localStream.getTracks().forEach(t=>t.stop()); } catch(e){} }
});

/* small UX: focus input */
txt.focus();

</script>
</body>
</html>
