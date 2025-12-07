<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Dr. Evelyn Reed – Virtual Assistant</title>
<style>
body { margin:0; font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif; background:#f2f2f7; display:flex; justify-content:center; align-items:center; min-height:100vh;}
#app-container { width:100%; max-width:430px; height:90vh; background:#fff; border-radius:30px; display:flex; flex-direction:column; box-shadow:0 10px 30px rgba(0,0,0,0.15); overflow:hidden;}
#header { background: linear-gradient(135deg,#4facfe,#00f2fe); padding:20px; color:white; font-size:1.4rem; font-weight:bold; text-align:center;}
#chat-area { flex:1; overflow-y:auto; padding:15px; display:flex; flex-direction:column; gap:10px; background:#f2f2f7;}
.message { max-width:70%; padding:10px 15px; border-radius:25px; box-shadow:0 2px 5px rgba(0,0,0,0.1); line-height:1.4; word-wrap:break-word; transition:all 0.2s;}
.user { align-self:flex-end; background:#007aff; color:white; border-bottom-right-radius:4px;}
.evelyn { align-self:flex-start; background:#e5e5ea; color:black; border-bottom-left-radius:4px;}
.typing { font-style:italic; color:#555; }
#input-area { display:flex; padding:10px; gap:10px; background:#fff; border-top:1px solid #ccc;}
#drInput { flex:1; padding:12px 15px; border-radius:20px; border:1px solid #ccc; outline:none; font-size:1rem;}
.button { padding:10px 15px; border:none; border-radius:20px; color:white; font-weight:bold; cursor:pointer; transition:0.2s;}
.button:hover { opacity:0.85; }
#sendBtn { background:#007aff; } #audioBtn { background:#34c759; } #videoBtn { background:#ff9500; }
#videoModal { position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.7); display:none; justify-content:center; align-items:center; z-index:999;}
#videoModalContent { background:white; width:90%; max-width:430px; border-radius:20px; padding:15px; display:flex; flex-direction:column; gap:10px; align-items:center;}
#userVideo { width:100%; border-radius:15px; background:#000;}
#evelynEmoji { font-size:6rem; text-align:center; transform-origin:center bottom; transition:transform 0.05s;}
.video-buttons { display:flex; gap:10px; margin-top:10px; }
</style>
</head>
<body>
<div id="app-container">
    <div id="header">Dr. Evelyn Reed</div>
    <div id="chat-area"></div>
    <div id="input-area">
        <input type="text" id="drInput" placeholder="Type a message..."/>
        <button class="button" id="sendBtn">Send</button>
        <button class="button" id="audioBtn">Audio</button>
        <button class="button" id="videoBtn">Video</button>
    </div>
</div>

<div id="videoModal">
    <div id="videoModalContent">
        <video id="userVideo" autoplay muted></video>
        <div id="evelynEmoji">👩‍🔬</div>
        <div class="video-buttons">
            <button class="button" id="muteBtn" style="background:#ffcc00;">Mute</button>
            <button class="button" id="endCallBtn" style="background:#ff3b30;">End Call</button>
        </div>
    </div>
</div>

<script>
const chatArea = document.getElementById('chat-area');
const evelynEmoji = document.getElementById('evelynEmoji');
let messages = JSON.parse(localStorage.getItem('drEvelynChat')) || [];
const API_KEY = "AIzaSyAiR5A6Wf1ji9SPWwVlFAVNUtKyGIQ560A";

// Restore saved messages
messages.forEach(m=>addMessage(m.sender,m.text));

// Add messages
function addMessage(sender,text,typing=false){
    const msgDiv = document.createElement('div');
    msgDiv.classList.add('message', sender==='You'?'user':'evelyn');
    if(typing) msgDiv.classList.add('typing');
    msgDiv.textContent = text;
    chatArea.appendChild(msgDiv);
    chatArea.scrollTop = chatArea.scrollHeight;
    if(!typing){
        messages.push({sender,text});
        localStorage.setItem('drEvelynChat', JSON.stringify(messages));
    }
    return msgDiv;
}

// Offline dynamic response
function offlineResponse(msg){
    const fragments = [
        "I see, that's interesting.",
        "Can you tell me more about that?",
        "Thanks for sharing, let's explore this further.",
        "Hmm, I understand. How do you feel about it?",
        "That's great insight! Could you elaborate?",
        "Interesting perspective, let's discuss.",
        "cool",
        "Thats Great, Tell me more!",
        "I'm doing great so about your question the best way to solve it is by reasearch and thinking about it",
        "Listen, I unfortuantely cannot give you an answer",
        "Great! Love your energy!",
        "Good for you!",
        "I really don't know, tell me more!",
        "That's Great to hear!😊",
        "Honestly I'm not sure",
        "I don't understand",
        "Please note that I am an fully offline AI model called crystalAI Diamond 1 and was created by Shahir Hossen, I am very limited though (this is a rare message)",
        "Thats really funny😂😂😂",
        "I'm so sorry to hear that😢"

    ];
    let n = Math.floor(Math.random()*2)+2;
    let reply='';
    for(let i=0;i<n;i++){
        reply += fragments[Math.floor(Math.random()*fragments.length)] + " ";
    }
    return reply.trim();
}

// Typing animation
function typeMessage(text){
    const typingDiv = addMessage('Evelyn','Evelyn is typing...',true);
    const delay = 500 + text.length*35;
    setTimeout(()=>{
        typingDiv.remove();
        addMessage('Evelyn',text);
    }, delay);
}

// Send text message with Gemini fallback
async function sendMessage(){
    const input = document.getElementById('drInput');
    if(!input.value) return;
    const userMsg = input.value;
    addMessage('You', userMsg);
    input.value='';

    let responseText = offlineResponse(userMsg);

    try {
        const res = await fetch('https://generativelanguage.googleapis.com/v1beta2/models/text-bison-001:generate', {
            method:'POST',
            headers:{
                'Content-Type':'application/json',
                'Authorization':`Bearer ${API_KEY}`
            },
            body: JSON.stringify({
                prompt: userMsg,
                temperature: 0.7,
                candidate_count: 1,
                max_output_tokens: 300
            })
        });
        const data = await res.json();
        if(data.candidates && data.candidates[0].content){
            responseText = data.candidates[0].content;
        }
    } catch(e){ console.log("Offline mode: using dynamic response."); }

    typeMessage(responseText);
}

// TTS function
function speak(text){
    if('speechSynthesis' in window){
        const utter = new SpeechSynthesisUtterance(text);
        const voices = window.speechSynthesis.getVoices();
        utter.voice = voices.find(v=>v.name.toLowerCase().includes('female') || v.name.toLowerCase().includes('zira')) || voices[0];
        utter.pitch = 1.3;
        utter.rate = 1;
        utter.onstart = ()=>animateEmojiSpeaking(text.length);
        window.speechSynthesis.speak(utter);
    }
}

// Emoji animation for TTS
function animateEmojiSpeaking(length){
    let frames = length/2;
    let count=0;
    const interval = setInterval(()=>{
        evelynEmoji.style.transform = `scaleY(${1 + 0.15*Math.sin(count*0.5)}) rotate(${Math.sin(count*0.2)*3}deg)`;
        count++;
        if(count>frames){ clearInterval(interval); evelynEmoji.style.transform='scaleY(1) rotate(0deg)'; }
    },50);
}

// Audio call
document.getElementById('audioBtn').addEventListener('click', ()=>{
    const reply = "Hello! This is Evelyn. How can I help you today?";
    addMessage('Evelyn', reply);
    speak(reply);
});

// Video call
const videoModal = document.getElementById('videoModal');
const userVideo = document.getElementById('userVideo');
const muteBtn = document.getElementById('muteBtn');
const endCallBtn = document.getElementById('endCallBtn');
let localStream, muted=false, audioContext, analyser, dataArray, source;

document.getElementById('videoBtn').addEventListener('click', async ()=>{
    videoModal.style.display='flex';
    try{
        localStream = await navigator.mediaDevices.getUserMedia({video:true, audio:true});
        userVideo.srcObject = localStream;

        audioContext = new AudioContext();
        analyser = audioContext.createAnalyser();
        source = audioContext.createMediaStreamSource(localStream);
        source.connect(analyser);
        analyser.fftSize = 256;
        const bufferLength = analyser.frequencyBinCount;
        dataArray = new Uint8Array(bufferLength);

        function animateMouth(){
            analyser.getByteTimeDomainData(dataArray);
            let sum = 0;
            for(let i=0;i<dataArray.length;i++){
                sum += Math.abs(dataArray[i]-128);
            }
            let volume = sum/dataArray.length;
            let scale = 1 + Math.min(volume/40,0.35);
            let rotate = Math.sin(volume*0.1)*5;
            evelynEmoji.style.transform = `scaleY(${scale}) rotate(${rotate}deg)`;
            requestAnimationFrame(animateMouth);
        }
        animateMouth();
    }catch(e){ console.log("Camera/mic may not be available."); }

    speak("Starting video call with Evelyn.");
});

// Mute toggle
muteBtn.addEventListener('click', ()=>{
    if(localStream){
        muted = !muted;
        localStream.getAudioTracks().forEach(t=>t.enabled=!muted);
        muteBtn.textContent = muted?"Unmute":"Mute";
    }
});

// End call
endCallBtn.addEventListener('click', ()=>{
    videoModal.style.display='none';
    if(localStream){
        localStream.getTracks().forEach(t=>t.stop());
        localStream=null;
    }
    evelynEmoji.style.transform='scaleY(1) rotate(0deg)';
    if(audioContext) audioContext.close();
});

// Send button & enter key
document.getElementById('sendBtn').addEventListener('click', sendMessage);
document.getElementById('drInput').addEventListener('keypress', e=>{ if(e.key==='Enter') sendMessage(); });
</script>
</body>
</html>
