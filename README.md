<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Ultimate AI Chatbot</title>
<style>
  body {
    font-family: Arial, sans-serif;
    margin: 20px;
    background: #121212;
    color: #eee;
  }
  #chatbox {
    width: 100%;
    max-width: 700px;
    height: 450px;
    background: #222;
    border-radius: 10px;
    padding: 15px;
    overflow-y: auto;
    box-shadow: 0 0 10px #000;
    margin-bottom: 10px;
  }
  .message {
    margin: 10px 0;
    padding: 10px 15px;
    border-radius: 20px;
    max-width: 75%;
    line-height: 1.4;
    font-size: 1em;
  }
  .user {
    background: #007bff;
    color: white;
    margin-left: auto;
    text-align: right;
  }
  .bot {
    background: #444;
    color: #eee;
    margin-right: auto;
    text-align: left;
  }
  #input-area {
    max-width: 700px;
    display: flex;
  }
  #input {
    flex: 1;
    padding: 12px;
    font-size: 1em;
    border-radius: 25px 0 0 25px;
    border: none;
    outline: none;
  }
  #sendBtn, #voiceBtn {
    background: #007bff;
    border: none;
    color: white;
    padding: 12px 18px;
    cursor: pointer;
    font-size: 1em;
  }
  #sendBtn:hover, #voiceBtn:hover {
    background: #0056b3;
  }
  #sendBtn {
    border-radius: 0 25px 25px 0;
  }
  #voiceBtn {
    border-radius: 25px;
    margin-left: 10px;
  }
  #status {
    max-width: 700px;
    margin: 8px 0;
    font-style: italic;
    color: #999;
  }
  #theme-select {
    margin-bottom: 10px;
    max-width: 700px;
  }
</style>
</head>
<body>

<h1>Ultimate AI Chatbot</h1>

<label for="theme-select">Choose Bot Personality / Theme:</label>
<select id="theme-select">
  <option value="friendly">Friendly Assistant</option>
  <option value="sarcastic">Sarcastic Bot</option>
  <option value="motivational">Motivational Coach</option>
  <option value="professional">Professional Advisor</option>
</select>

<div id="chatbox"></div>
<div id="status"></div>

<div id="input-area">
  <input id="input" type="text" placeholder="Type your message here..." autocomplete="off" />
  <button id="sendBtn">Send</button>
  <button id="voiceBtn" title="Toggle Voice Input/Output">🎤</button>
</div>

<script>
  // =================
  // CONFIGURATION
  // =================

  const OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY_HERE'; // <-- Replace this with your OpenAI API key

  // Personalities prompt prefixes
  const personalities = {
    friendly: "You are a friendly and helpful assistant. Always polite and supportive.",
    sarcastic: "You are a sarcastic and witty assistant. Use humor and sarcasm in your replies.",
    motivational: "You are a motivational coach. Inspire and encourage the user.",
    professional: "You are a professional advisor. Speak formally and clearly."
  };

  // =================
  // DOM ELEMENTS
  // =================

  const chatbox = document.getElementById('chatbox');
  const input = document.getElementById('input');
  const sendBtn = document.getElementById('sendBtn');
  const voiceBtn = document.getElementById('voiceBtn');
  const status = document.getElementById('status');
  const themeSelect = document.getElementById('theme-select');

  // =================
  // CHAT HISTORY & STATE
  // =================

  let chatHistory = []; // store messages for context [{role:"user"/"assistant", content:"text"}]

  // =================
  // SPEECH RECOGNITION SETUP
  // =================

  let recognizing = false;
  let recognition;

  if ('webkitSpeechRecognition' in window) {
    recognition = new webkitSpeechRecognition();
  } else if ('SpeechRecognition' in window) {
    recognition = new SpeechRecognition();
  } else {
    recognition = null;
    voiceBtn.disabled = true;
    voiceBtn.title = 'Voice input not supported in this browser';
  }

  if (recognition) {
    recognition.continuous = false;
    recognition.lang = 'en-US';
    recognition.interimResults = false;
    recognition.maxAlternatives = 1;

    recognition.onstart = () => {
      recognizing = true;
      status.textContent = 'Listening... Speak now.';
      voiceBtn.textContent = '🎙️';
    };

    recognition.onend = () => {
      recognizing = false;
      status.textContent = '';
      voiceBtn.textContent = '🎤';
    };

    recognition.onerror = (event) => {
      status.textContent = 'Speech recognition error: ' + event.error;
      recognizing = false;
      voiceBtn.textContent = '🎤';
    };

    recognition.onresult = (event) => {
      const transcript = event.results[0][0].transcript;
      input.value = transcript;
      sendMessage();
    };
  }

  // =================
  // FUNCTIONS
  // =================

  function appendMessage(sender, text) {
    const msg = document.createElement('div');
    msg.classList.add('message', sender);
    msg.textContent = text;
    chatbox.appendChild(msg);
    chatbox.scrollTop = chatbox.scrollHeight;
  }

  function speak(text) {
    if (!('speechSynthesis' in window)) return;
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.lang = 'en-US';
    speechSynthesis.speak(utterance);
  }

  async function callOpenAI(messages) {
    status.textContent = 'Thinking...';

    try {
      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${OPENAI_API_KEY}`
        },
        body: JSON.stringify({
          model: 'gpt-4o-mini',
          messages: messages,
          temperature: 0.7,
          max_tokens: 500,
        })
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error.message || 'OpenAI API error');
      }

      const data = await response.json();
      const botMessage = data.choices[0].message.content.trim();
      status.textContent = '';
      return botMessage;

    } catch (error) {
      status.textContent = 'Error: ' + error.message;
      return "Sorry, I couldn't process that right now.";
    }
  }

  async function sendMessage() {
    const userText = input.value.trim();
    if (!userText) return;

    appendMessage('user', userText);
    chatHistory.push({ role: 'user', content: userText });
    input.value = '';

    // Build messages with personality prefix
    const personality = personalities[themeSelect.value];
    let systemMessage = { role: 'system', content: personality };

    // Compose full messages for OpenAI
    const messages = [systemMessage, ...chatHistory];

    const botReply = await callOpenAI(messages);
    appendMessage('bot', botReply);
    chatHistory.push({ role: 'assistant', content: botReply });

    // Speak bot reply aloud
    speak(botReply);
  }

  sendBtn.addEventListener('click', sendMessage);

  input.addEventListener('keydown', e => {
    if (e.key === 'Enter') sendMessage();
  });

  voiceBtn.addEventListener('click', () => {
    if (!recognition) return;
    if (recognizing) {
      recognition.stop();
      recognizing = false;
      voiceBtn.textContent = '🎤';
    } else {
      recognition.start();
    }
  });
</script>

</body>
</html>
