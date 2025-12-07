<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Dr. Evelyn Reed — Call + Chat (Offline-first, Gemini optional)</title>
<style>
  :root{--bg:#f2f2f7; --card:#fff; --accent:#007aff; --muted:#e5e5ea}
  body{margin:0;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,Arial,sans-serif;background:var(--bg);display:flex;align-items:center;justify-content:center;min-height:100vh}
  #app{width:100%;max-width:430px;height:90vh;background:var(--card);border-radius:24px;box-shadow:0 12px 30px rgba(0,0,0,.12);display:flex;flex-direction:column;overflow:hidden}
  header{background:linear-gradient(135deg,#4facfe,#00f2fe);color:#fff;padding:18px;text-align:center;font-weight:700;font-size:1.2rem}
  #chat{flex:1;overflow:auto;padding:16px;display:flex;flex-direction:column;gap:10px;background:var(--bg)}
  .msg{max-width:72%;padding:10px 14px;border-radius:18px;box-shadow:0 2px 6px rgba(0,0,0,.06)}
  .you{align-self:flex-end;background:var(--accent);color:#fff;border-bottom-right-radius:6px}
  .ev{align-self:flex-start;background:var(--muted);color:#000;border-bottom-left-radius:6px}
  .typing{font-style:italic;color:#555}
  #composer{display:flex;padding:12px;gap:8px;border-top:1px solid #e6e6e6;background:#fff}
  input[type="text"]{flex:1;padding:12px 14px;border-radius:20px;border:1px solid #d0d0d0;outline:none;font-size:15px}
  .btn{padding:10px 12px;border-radius:18px;border:none;color:#fff;font-weight:700;cursor:pointer}
  .btn:active{transform:translateY(1px)}
  .send{background:var(--accent)}
  .audio{background:#34c759}
  .video{background:#ff9500}
  /* modal */
  #callModal{position:fixed;inset:0;display:none;align-items:center;justify-content:center;background:rgba(0,0,0,.6);z-index:999}
  .callCard{width:95%;max-width:520px;background:#fff;border-radius:16px;padding:12px;display:flex;flex-direction:column;gap:10px;align-items:center}
  #userVideo{width:100%;max-height:280px;border-radius:12px;background:#000;object-fit:cover}
  #evelynEmoji{font-size:6rem;transform-origin:center bottom;transition:transform .08s}
  .callControls{display:flex;gap:8px}
  .mutebtn{background:#ffcc00;color:#000}
  .endbtn{background:#ff3b30}
  .timer{font-weight:700;margin-left:8px}
  .mic-ind{font-size:.9rem;color:#333;padding:6px 8px;border-radius:12px;background:#f5f5f7}
  .small{font-size:13px;color:#666}
</style>
</head>
<body>

<div id="app">
  <header>Dr. Evelyn Reed</header>

  <div id="chat"></div>

  <div id="composer">
    <input id="txt" type="text" placeholder="Type a message..." autocomplete="off"/>
    <button id="sendBtn" class="btn send">Send</button>
    <button id="audioBtn" class="btn audio" title="Start audio call">Audio</button>
    <button id="videoBtn" class="btn video" title="Start video call">Video</button>
  </div>
</div>

<!-- Call modal (used for both audio and video calls) -->
<div id="callModal" aria-hidden="true">
  <div class="callCard">
    <div style="width:100%;display:flex;justify-content:space-between;align-items:center">
      <div style="display:flex;align-items:center;gap:8px">
        <div class="mic-ind" id="micStatus">Mic: off</div>
        <div class="small" id="callTimer">00:00</div>
      </div>
      <div class="small" id="callMode">Call</div>
    </div>

    <video id="userVideo" autoplay muted playsinline></video>
    <div id="evelynEmoji">👩‍🔬</div>

    <div class="callControls">
      <button id="muteBtn" class="btn mutebtn">Mute</button>
      <button id="endBtn" class="btn endbtn">End Call</button>
    </div>
    <div class="small" id="callHint">Evelyn will listen while you speak and reply with TTS.</div>
  </div>
</div>

<script>
/* ===========================
   Offline-first assistant
   Gemini key (Google) used when available
   =========================== */
const GEMINI_KEY = "AIzaSyAiR5A6Wf1ji9SPWwVlFAVNUtKyGIQ560A"; // your key (Gemini)
const chatEl = document.getElementById('chat');
const txt = document.getElementById('txt');
const sendBtn = document.getElementById('sendBtn');
const audioBtn = document.getElementById('audioBtn');
const videoBtn = document.getElementById('videoBtn');
const callModal = document.getElementById('callModal');
const userVideo = document.getElementById('userVideo');
const evelynEmoji = document.getElementById('evelynEmoji');
const muteBtn = document.getElementById('muteBtn');
const endBtn = document.getElementById('endBtn');
const micStatus = document.getElementById('micStatus');
const callTimer = document.getElementById('callTimer');
const callModeLabel = document.getElementById('callMode');

let saved = JSON.parse(localStorage.getItem('drEvelynChat') || '[]');
if(Array.isArray(saved)) saved.forEach(m=>renderMsg(m.sender,m.text));

// helper render
function renderMsg(sender,text,typing=false){
  const div = document.createElement('div');
  div.className = 'msg ' + (sender === 'You' ? 'you' : 'ev');
  if(typing) div.classList.add('typing');
  div.textContent = text;
  chatEl.appendChild(div);
  chatEl.scrollTop = chatEl.scrollHeight;
  if(!typing) {
    saved.push({sender,text});
    localStorage.setItem('drEvelynChat', JSON.stringify(saved));
  }
  return div;
}

// offline local responder (longer, dynamic)
function offlineResponse(userMsg){
  const fragments = [
    "I see, that's interesting.",
    "Can you tell me more about that?",
    "Thanks for sharing — let's explore it.",
    "Hmm, I understand. How does that make you feel?",
    "That's useful info. Could you elaborate a bit more?",
    "I appreciate you telling me that. Let's continue.",
    "Thats really cool!",
    "6 7",
    "ohio',
    "Tell me more about it",
    "idk bro",
    "unfortanutely I can't give an answer sry",
    "4 1",
    "I'm busy right now please wait",
    "I'm so sorry to hear that😢",
    "That's great to hear😊',
    "Awesome, can you talk more about it?",
    "W😎",
    "Bro don't expect me to always give an answer",
    "Did you know that I am a offline AI model called crystalAI and was deveolped by Shahir Hossen (this is a rare message),
    "Error, please try agagin",
    "I am a message",
    "Cool, Tell me more!",
    "Lets talk about this in a video chat"
  ];
  const n = 2 + Math.floor(Math.random()*2);
  let out = [];
  for(let i=0;i<n;i++) out.push(fragments[Math.floor(Math.random()*fragments.length)]);
  return out.join(' ');
}

// try Gemini (Google) via REST; fallback to offlineResponse
async function askGemini(userMsg){
  if(!GEMINI_KEY) return offlineResponse(userMsg);
  try{
    const resp = await fetch('https://generativelanguage.googleapis.com/v1beta2/models/text-bison-001:generate', {
      method:'POST',
      headers:{'Content-Type':'application/json','Authorization':`Bearer ${GEMINI_KEY}`},
      body: JSON.stringify({
        prompt: userMsg,
        temperature: 0.6,
        candidate_count: 1,
        max_output_tokens: 300
      })
    });
    const data = await resp.json();
    // parse robustly
    if(data && data.candidates && data.candidates[0] && data.candidates[0].content) {
      return data.candidates[0].content.trim();
    }
    // some APIs return different shape
    if(data && data.result && data.result.output) return data.result.output;
    return offlineResponse(userMsg);
  }catch(e){
    console.log('Gemini request failed, offline fallback', e);
    return offlineResponse(userMsg);
  }
}

/* ----------------------------
   Text chat (no TTS)
   ---------------------------- */
sendBtn.addEventListener('click', async ()=>{
  const v = txt.value.trim(); if(!v) return; txt.value='';
  renderMsg('You', v);
  // show typing indicator
  const typingEl = renderMsg('Evelyn','Evelyn is typing...', true);
  // request
  const reply = await askGemini(v);
  typingEl.remove();
  renderMsg('Evelyn', reply);
});

// Enter to send
txt.addEventListener('keypress', e=>{ if(e.key === 'Enter') sendBtn.click(); });

/* ===========================
   Audio / Video call logic
   Offline-first:
   - Use SpeechRecognition if available (may require network)
   - Otherwise use VAD (analyser) to detect voice segments, then send that "utterance" to Gemini/offline responder
   - While TTS plays, pause recognition/VAD to avoid feedback
   =========================== */

let localStream = null;
let audioCtx=null, analyser=null, dataArray=null, mediaSource=null;
let vadLoopRunning = false;
let recognition = null;
let callStart = null;
let callTimerInterval = null;
let inCallMode = null; // 'audio' or 'video'
let muted = false;

// helper timer
function startTimer(){
  callStart = Date.now();
  callTimerInterval = setInterval(()=>{
    const s = Math.floor((Date.now()-callStart)/1000);
    const mm = String(Math.floor(s/60)).padStart(2,'0');
    const ss = String(s%60).padStart(2,'0');
    callTimer.textContent = `${mm}:${ss}`;
  },200);
}
function stopTimer(){
  clearInterval(callTimerInterval); callTimer.textContent='00:00';
}

/* Utility: pause STT/VAD */
function pauseListening(){
  if(recognition && recognition.abort) try{ recognition.abort(); }catch(e){}
  vadLoopRunning = false;
}
/* Utility: resume STT/VAD */
function resumeListening(){
  if(recognition && recognition.start && !recognition.active){
    try{ recognition.start(); }catch(e){}
  }
  if(!vadLoopRunning && analyser) { startVADLoop(); }
}

/* Start local audio capture + analyser (VAD) */
async function startLocalMedia(requireVideo=false){
  try{
    localStream = await navigator.mediaDevices.getUserMedia({audio:true, video: requireVideo});
    if(requireVideo && userVideo){
      userVideo.srcObject = localStream;
    }
    // audio analyser
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    mediaSource = audioCtx.createMediaStreamSource(localStream);
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048;
    mediaSource.connect(analyser);
    const bufferLength = analyser.fftSize;
    dataArray = new Uint8Array(bufferLength);
    return true;
  }catch(err){
    console.warn('startLocalMedia failed', err);
    return false;
  }
}

/* VAD loop — detect speech segments and call handleUtterance */
let vadSilenceStart = null;
let hadSpeech = false;
function startVADLoop(){
  if(!analyser) return;
  vadLoopRunning = true;
  vadSilenceStart = null; hadSpeech = false;
  async function loop(){
    if(!vadLoopRunning) return;
    analyser.getByteTimeDomainData(dataArray);
    // compute RMS-ish
    let sum=0;
    for(let i=0;i<dataArray.length;i++){ const v = dataArray[i]-128; sum += Math.abs(v); }
    const avg = sum / dataArray.length;
    const threshold = 12; // tuned; may vary by device
    if(avg > threshold){
      hadSpeech = true;
      vadSilenceStart = null;
      micStatus.textContent = 'Mic: speaking';
    } else {
      if(hadSpeech && !vadSilenceStart){
        vadSilenceStart = Date.now();
      }
      micStatus.textContent = 'Mic: listening';
    }
    // if we had speech and detected a period of silence -> treat as end of utterance
    if(hadSpeech && vadSilenceStart && (Date.now() - vadSilenceStart) > 600){
      // capture a short audio blob (optional) - but offline we don't transcribe audio
      // handle utterance with no transcript
      hadSpeech = false;
      vadSilenceStart = null;
      // pause listening before generating reply to avoid feedback
      pauseListening();
      await handleUtterance(null); // null = no text transcript
      // resume after reply finishes (handled in handleUtterance)
    }
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
}

/* SpeechRecognition (if available) wrapper.
   We'll use it when present; for iOS Safari use webkitSpeechRecognition.
   Behavior:
   - continuous:true
   - onresult -> handleUtterance(transcript)
*/
function setupRecognition(){
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition || null;
  if(!SpeechRecognition) return null;
  try{
    recognition = new SpeechRecognition();
    recognition.continuous = true;
    recognition.interimResults = false;
    recognition.lang = 'en-US';
    recognition.onstart = ()=>{ console.log('recognition started'); recognition.active=true; };
    recognition.onend = ()=>{ console.log('recognition ended'); recognition.active=false; /* auto-restart? optional */ };
    recognition.onerror = (e)=>{ console.log('recognition error',e); };
    recognition.onresult = async (ev)=>{
      // collect the final transcript
      let transcript = '';
      for(let i=ev.resultIndex;i<ev.results.length;i++){
        if(ev.results[i].isFinal) transcript += ev.results[i][0].transcript + ' ';
      }
      if(transcript.trim()){
        // temporarily pause recognition to avoid feedback while TTS speaks
        try{ recognition.stop(); }catch(e){}
        await handleUtterance(transcript.trim());
        // restart recognition
        try{ recognition.start(); }catch(e){}
      }
    };
    return recognition;
  }catch(e){
    console.warn('recognition setup failed', e);
    return null;
  }
}

/* handle utterance: transcript may be null (no STT), otherwise contains text.
   We call Gemini if possible, else offlineResponse; then TTS reply and resume listening.
*/
async function handleUtterance(transcript){
  // show user placeholder in chat if we have transcript
  if(transcript){
    renderMsg('You', transcript);
  } else {
    // If no transcript, show a small indicator (we captured audio)
    renderMsg('You', '[voice message — transcription unavailable offline]');
  }
  // Show typing indicator for Evelyn
  const typingEl = renderMsg('Evelyn','Evelyn is thinking...', true);

  // Build prompt: if we have transcript prefer that
  const promptText = transcript || '[user spoke but transcription unavailable]';
  // get response
  const reply = await askGemini(promptText);
  typingEl.remove();
  renderMsg('Evelyn', reply);

  // speak via TTS and animate
  await speakWithPromise(reply);

  // resume listening
  if(recognition) { try{ recognition.start(); }catch(e){} }
  if(analyser) { startVADLoop(); }
}

/* speak that returns a promise which resolves when speech ends; also pauses recognition/VAD */
function speakWithPromise(text){
  return new Promise((resolve)=>{
    if(!('speechSynthesis' in window)){
      resolve();
      return;
    }
    // pause listening to avoid feedback
    try{ if(recognition) recognition.abort(); }catch(e){}
    vadLoopRunning = false;
    const utt = new SpeechSynthesisUtterance(text);
    const voices = window.speechSynthesis.getVoices();
    utt.voice = voices.find(v=>v.name.toLowerCase().includes('female') || v.name.toLowerCase().includes('zira')) || voices[0];
    utt.pitch = 1.25; utt.rate = 1;
    utt.onstart = ()=>{ animateEmojiSpeaking(text.length); };
    utt.onend = ()=>{ 
      // small delay to let audio settle
      setTimeout(()=>{
        // resume audio analyser listening
        if(analyser) startVADLoop();
        // restart recognition if available
        if(recognition) try{ recognition.start(); }catch(e){}
        resolve();
      }, 120);
    };
    window.speechSynthesis.speak(utt);
  });
}

function animateEmojiSpeaking(length){
  let frames = Math.max(6, Math.floor(length/6));
  let count=0;
  const id = setInterval(()=>{
    evelynEmoji.style.transform = `scaleY(${1 + 0.16*Math.sin(count*0.6)}) rotate(${Math.sin(count*0.13)*3}deg)`;
    count++;
    if(count>frames){ clearInterval(id); evelynEmoji.style.transform='scaleY(1) rotate(0deg)'; }
  },50);
}

/* Start a call (mode 'audio' or 'video') */
async function startCall(mode='audio'){
  inCallMode = mode;
  callModeLabel.textContent = mode === 'audio' ? 'Audio call' : 'Video call';
  callModal.style.display = 'flex';
  // start local media
  const ok = await startLocalMedia(mode === 'video');
  if(!ok) {
    alert('Could not access camera/mic. Check permissions.');
    closeCall();
    return;
  }
  micStatus.textContent = 'Mic: on';
  startTimer();

  // Setup STT: recognition if available
  const r = setupRecognition();
  if(r){
    try{ r.start(); }catch(e){ console.warn('recognition start failed'); }
  } else {
    // fallback to VAD only
    startVADLoop();
  }
}

/* End / close call */
function closeCall(){
  if(localStream){
    localStream.getTracks().forEach(t=>t.stop());
    localStream=null;
  }
  if(audioCtx) { try{ audioCtx.close(); }catch(e){} audioCtx=null; analyser=null; dataArray=null;}
  if(recognition && recognition.abort) try{ recognition.abort(); }catch(e){}
  vadLoopRunning=false;
  callModal.style.display='none';
  stopTimer();
  evelynEmoji.style.transform='scaleY(1) rotate(0deg)';
  micStatus.textContent = 'Mic: off';
}

/* Toggle mute */
function toggleMute(){
  if(!localStream) return;
  muted = !muted;
  localStream.getAudioTracks().forEach(t=>t.enabled = !muted);
  muteBtn.textContent = muted ? 'Unmute' : 'Mute';
}

/* Bind buttons */
audioBtn.addEventListener('click', ()=> startCall('audio'));
videoBtn.addEventListener('click', ()=> startCall('video'));
endBtn.addEventListener('click', closeCall);
muteBtn.addEventListener('click', toggleMute);

/* Helper: ask Gemini wrapper already defined earlier as askGemini() */

/* TTS promise version for audio button quick message */
async function handleAudioButton(){
  const reply = "Hello! This is Evelyn. You can speak now and I will respond.";
  renderMsg('Evelyn', reply);
  await speakWithPromise(reply);
}

/* audioBtn already starts call; but also keep plain audio single-turn behavior
   if user clicks audioBtn while not wanting the full call, they can use send message.
*/

// small UX: announce recognition availability
(function checkRecognitionAvailable(){
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition || null;
  if(SpeechRecognition) {
    console.log('SpeechRecognition available (may require network in some browsers)');
  } else {
    console.log('SpeechRecognition not available — using VAD fallback (offline).');
  }
})();

</script>
</body>
</html>
