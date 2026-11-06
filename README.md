
"""
BHARMASTRA AI — single-file starter application
Run:
  pip install flask
  # Optional: set your model API key before starting
  # Windows PowerShell: $env:OPENAI_API_KEY="your-key"
  # macOS/Linux: export OPENAI_API_KEY="your-key"
  python bharmastra_ai.py

Open http://127.0.0.1:5000

Without an API key, the app runs in DEMO mode. This is a learning starter,
not a production-ready service. Add authentication, rate limits, monitoring,
and secure secret management before public deployment.
"""
import json
import os
import urllib.request
import urllib.error
from flask import Flask, request, jsonify, render_template_string

app = Flask(__name__)
app.config["MAX_CONTENT_LENGTH"] = 1 * 1024 * 1024

MODEL = os.getenv("OPENAI_MODEL", "gpt-4.1-mini")
API_KEY = os.getenv("OPENAI_API_KEY", "")
API_URL = os.getenv("OPENAI_API_URL", "https://api.openai.com/v1/chat/completions")

SYSTEM_PROMPT = """You are BHARMASTRA AI, a helpful, honest, safe general-purpose assistant.
Be clear and practical. If you are unsure, say so. Do not claim to have performed
actions outside this chat. Keep answers age-appropriate. Refuse requests that
would facilitate harm, crime, or dangerous activity, and offer safe alternatives."""

PAGE = r"""
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>BHARMASTRA AI</title>
<style>
:root{color-scheme:dark;--bg:#090b12;--panel:#111522;--line:#252c40;--gold:#e8b84c;--blue:#6ca8ff;--text:#edf1fa;--muted:#9ba6bb}
*{box-sizing:border-box}body{margin:0;background:radial-gradient(ellipse at 50% -20%,#1c2945 0,transparent 55%),var(--bg);color:var(--text);font:15px/1.5 Inter,system-ui,Arial}
header{height:76px;border-bottom:1px solid var(--line);display:flex;align-items:center;justify-content:space-between;padding:0 5%;background:#090b12dd;position:sticky;top:0;backdrop-filter:blur(12px)}
.brand{font-weight:900;letter-spacing:2px;font-size:20px}.brand span{color:var(--gold)}.tag{color:var(--muted);font-size:12px}
main{max-width:900px;margin:45px auto;padding:0 18px}.hero{text-align:center;margin-bottom:28px}.hero h1{font-size:clamp(30px,6vw,48px);margin:0 0 8px}.hero h1 span{color:var(--gold)}.hero p{color:var(--muted);margin:0}
.status{display:inline-block;margin-top:14px;border:1px solid var(--line);border-radius:99px;padding:5px 12px;color:var(--muted);font-size:12px}
.chat{border:1px solid var(--line);border-radius:18px;background:#0e1320eF;overflow:hidden;box-shadow:0 20px 80px #0006}
.messages{height:min(54vh,520px);min-height:300px;overflow:auto;padding:22px;display:flex;flex-direction:column;gap:16px}
.msg{max-width:85%;padding:13px 16px;border-radius:15px;white-space:pre-wrap;overflow-wrap:anywhere}.user{align-self:flex-end;background:#1c3151;border:1px solid #294b77}.assistant{align-self:flex-start;background:var(--panel);border:1px solid var(--line)}
.label{font-size:10px;color:var(--gold);font-weight:800;letter-spacing:1px;margin-bottom:5px}
form{border-top:1px solid var(--line);padding:14px;display:flex;gap:10px;background:#0c101a}
textarea{flex:1;resize:vertical;min-height:52px;max-height:150px;border:1px solid var(--line);border-radius:12px;background:#090d16;color:var(--text);padding:14px;font:inherit;outline:none}
textarea:focus{border-color:var(--blue)}button{border:0;border-radius:12px;padding:0 22px;background:linear-gradient(135deg,var(--gold),#f4d68a);color:#17130a;font-weight:800;cursor:pointer}button:disabled{opacity:.5;cursor:wait}
.actions{display:flex;justify-content:space-between;align-items:center;padding:10px 16px;color:var(--muted);font-size:12px}.clear{background:none;color:var(--muted);padding:4px 8px;border:1px solid var(--line)}
footer{text-align:center;color:#778197;font-size:12px;margin:18px}
@media(max-width:600px){main{margin:25px auto}.messages{padding:14px}.msg{max-width:95%}form{padding:10px}button{padding:0 15px}}
</style>
</head>
<body>
<header><div class="brand">BHARMASTRA <span>AI</span></div><div class="tag">YOUR AI WORKSPACE</div></header>
<main>
 <section class="hero"><h1>Think bigger with <span>BHARMASTRA AI</span></h1><p>Ask questions. Learn. Create. Build.</p><div class="status" id="status">Checking system…</div></section>
 <section class="chat">
  <div class="messages" id="messages">
   <div class="msg assistant"><div class="label">BHARMASTRA AI</div>Hello! I’m BHARMASTRA AI. What would you like to work on today?</div>
  </div>
  <form id="chatForm"><textarea id="prompt" placeholder="Message BHARMASTRA AI…" rows="1" maxlength="8000" required></textarea><button id="send" type="submit">Send ➤</button></form>
  <div class="actions"><span>Enter to send · Shift + Enter for a new line</span><button class="clear" id="clear" type="button">Clear chat</button></div>
 </section>
 <footer>Starter edition · Do not enter passwords, payment details, or private information.</footer>
</main>
<script>
const messages=document.getElementById('messages'), form=document.getElementById('chatForm');
const promptBox=document.getElementById('prompt'), send=document.getElementById('send');
let history=[];
function addMessage(role,text){const box=document.createElement('div');box.className='msg '+role;const label=document.createElement('div');label.className='label';label.textContent=role==='user'?'YOU':'BHARMASTRA AI';const body=document.createElement('div');body.textContent=text;box.append(label,body);messages.appendChild(box);messages.scrollTop=messages.scrollHeight;return box;}
fetch('/api/status').then(r=>r.json()).then(d=>document.getElementById('status').textContent=d.mode==='live'?'● AI connected':'● Demo mode — add API key for live AI').catch(()=>document.getElementById('status').textContent='● Offline');
form.addEventListener('submit',async e=>{e.preventDefault();const text=promptBox.value.trim();if(!text)return;addMessage('user',text);history.push({role:'user',content:text});promptBox.value='';send.disabled=true;send.textContent='Thinking…';const pending=addMessage('assistant','Thinking…');
try{const res=await fetch('/api/chat',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({messages:history})});const data=await res.json();if(!res.ok)throw new Error(data.error||'Request failed');pending.lastChild.textContent=data.reply;history.push({role:'assistant',content:data.reply});}
catch(err){pending.lastChild.textContent='Sorry — '+err.message;history.pop();}
finally{send.disabled=false;send.textContent='Send ➤';promptBox.focus();}
});
promptBox.addEventListener('keydown',e=>{if(e.key==='Enter'&&!e.shiftKey){e.preventDefault();form.requestSubmit();}});
document.getElementById('clear').addEventListener('click',()=>{history=[];messages.replaceChildren();addMessage('assistant','Chat cleared. What shall we explore next?');});
</script>
</body>
</html>
"""

def demo_reply(messages):
    last = messages[-1]["content"].strip() if messages else ""
    return (
        "BHARMASTRA AI is currently running in DEMO mode because no API key is configured.\n\n"
        f"You asked: “{last[:500]}”\n\n"
        "The interface and chat endpoint are working. To enable live AI responses, set "
        "OPENAI_API_KEY in your environment and restart the app. Keep the key private."
    )

def live_reply(messages):
    payload = {
        "model": MODEL,
        "messages": [{"role": "system", "content": SYSTEM_PROMPT}] + messages[-20:],
        "temperature": 0.7
    }
    req = urllib.request.Request(
        API_URL,
        data=json.dumps(payload).encode("utf-8"),
        headers={"Authorization": f"Bearer {API_KEY}", "Content-Type": "application/json"},
        method="POST"
    )
    try:
        with urllib.request.urlopen(req, timeout=60) as response:
            data = json.loads(response.read().decode("utf-8"))
        return data["choices"][0]["message"]["content"]
    except urllib.error.HTTPError as exc:
        detail = exc.read().decode("utf-8", errors="replace")[:500]
        raise RuntimeError(f"AI provider returned HTTP {exc.code}: {detail}")
    except (KeyError, IndexError, ValueError) as exc:
        raise RuntimeError("Unexpected response from AI provider.") from exc
    except urllib.error.URLError as exc:
        raise RuntimeError(f"Could not reach AI provider: {exc.reason}") from exc

@app.get("/")
def home():
    return render_template_string(PAGE)

@app.get("/api/status")
def status():
    return jsonify({"mode": "live" if API_KEY else "demo", "model": MODEL if API_KEY else None})

@app.post("/api/chat")
def chat():
    body = request.get_json(silent=True) or {}
    messages = body.get("messages")
    if not isinstance(messages, list) or not messages:
        return jsonify({"error": "Send at least one message."}), 400
    if len(messages) > 40:
        return jsonify({"error": "Conversation is too long. Clear the chat and try again."}), 400
    clean = []
    for item in messages:
        if not isinstance(item, dict) or item.get("role") not in ("user", "assistant"):
            return jsonify({"error": "Invalid message format."}), 400
        content = item.get("content")
        if not isinstance(content, str) or len(content) > 8000:
            return jsonify({"error": "Each message must be text under 8,000 characters."}), 400
        clean.append({"role": item["role"], "content": content})
    try:
        reply = live_reply(clean) if API_KEY else demo_reply(clean)
        return jsonify({"reply": reply})
    except RuntimeError as exc:
        return jsonify({"error": str(exc)}), 502

if __name__ == "__main__":
    # Local development only. Use a production WSGI server for deployment.
    app.run(host="127.0.0.1", port=int(os.getenv("PORT", "5000")), debug=False)
