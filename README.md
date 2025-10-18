"""
Simple functional AI Chatbot with Gemini API (local key version)
---------------------------------------------------------------
Features:
- Flask backend with /api/chat endpoint
- Single-file frontend for chatting
- Uses Google Generative AI (Gemini)
- In-memory conversation history

Requirements:
    pip install flask flask-cors google-generativeai
Run:
    python ai_chatbot_api_flask_gemini.py
Then open http://127.0.0.1:5000
"""

from flask import Flask, request, jsonify, render_template_string
from flask_cors import CORS
import google.generativeai as genai
import uuid

app = Flask(__name__)
CORS(app)

# --- ⚠️ Insert your Gemini API key here (local only) ---
GEMINI_API_KEY = "AIzaSyAQZuaUCNsgo3FDhOLzQR-z8bddbDpzHYs"

# Configure Gemini
genai.configure(api_key=GEMINI_API_KEY)

# --- Conversation store ---
conversations = {}
SYSTEM_PROMPT = (
    "You are a helpful, concise AI assistant. Keep answers short unless the user asks for more details."
)


@app.route("/api/chat", methods=["POST"])
def chat_api():
    """POST JSON body:
    { "session_id": "optional", "message": "user message" }
    """
    data = request.get_json(force=True)
    user_message = data.get("message", "").strip()
    if not user_message:
        return jsonify({"error": "`message` is required"}), 400

    # Create or continue a session
    session_id = data.get("session_id") or str(uuid.uuid4())
    if session_id not in conversations:
        conversations[session_id] = [{"role": "system", "content": SYSTEM_PROMPT}]

    conversations[session_id].append({"role": "user", "content": user_message})

    # Prepare prompt from conversation history
    chat_history = "\n".join(
        f"{m['role'].capitalize()}: {m['content']}"
        for m in conversations[session_id]
        if m["role"] != "system"
    )
    prompt = f"{SYSTEM_PROMPT}\n\n{chat_history}\nAssistant:"

    try:
        model = genai.GenerativeModel("gemini-1.5-flash")
        response = model.generate_content(prompt)
        assistant_message = response.text or "(no reply from model)"
    except Exception as e:
        return jsonify({"error": str(e)}), 500

    conversations[session_id].append({"role": "assistant", "content": assistant_message})

    # Limit memory
    MAX_HISTORY = 20
    if len(conversations[session_id]) > MAX_HISTORY:
        conversations[session_id] = [conversations[session_id][0]] + conversations[
            session_id
        ][-(MAX_HISTORY - 1) :]

    return jsonify({"session_id": session_id, "reply": assistant_message})


# --- Frontend for chat ---
INDEX_HTML = """
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>AI Chatbot (Gemini)</title>
    <style>
      body { font-family: Arial, sans-serif; max-width:800px; margin:40px auto; }
      #chat { border:1px solid #ddd; padding:16px; height:400px; overflow:auto; }
      .msg { margin:8px 0; }
      .user { text-align:right; }
      .assistant { text-align:left; color:#0b5394; }
      #controls { margin-top:12px; display:flex; gap:8px }
      textarea { flex:1; height:60px }
    </style>
  </head>
  <body>
    <h2>Simple AI Chatbot (Gemini API)</h2>
    <div id="chat"></div>
    <div id="controls">
      <textarea id="input" placeholder="Type a message..."></textarea>
      <button id="send">Send</button>
    </div>

    <script>
      let sessionId = null;
      const chatEl = document.getElementById('chat');
      const inputEl = document.getElementById('input');
      const sendBtn = document.getElementById('send');

      function appendMessage(text, cls){
        const d = document.createElement('div');
        d.className = 'msg ' + cls;
        d.textContent = text;
        chatEl.appendChild(d);
        chatEl.scrollTop = chatEl.scrollHeight;
      }

      sendBtn.onclick = async () => {
        const text = inputEl.value.trim();
        if(!text) return;
        appendMessage(text, 'user');
        inputEl.value = '';

        const payload = { message: text };
        if(sessionId) payload.session_id = sessionId;

        try{
          const res = await fetch('/api/chat', {
            method:'POST',
            headers:{'Content-Type':'application/json'},
            body: JSON.stringify(payload)
          });
          const data = await res.json();
          if(data.error){
            appendMessage('Error: ' + data.error, 'assistant');
          } else {
            sessionId = data.session_id;
            appendMessage(data.reply, 'assistant');
          }
        } catch(err){
          appendMessage('Network error: ' + err.message, 'assistant');
        }
      }
    </script>
  </body>
</html>
"""

@app.route("/", methods=["GET"])
def index():
    return render_template_string(INDEX_HTML)


if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
# AI-chatbot-2
