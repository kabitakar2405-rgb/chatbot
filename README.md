<!DOCTYPE html>
<html lang="bn">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>ChatLike — ChatGPT-মতো চ্যাট UI</title>
  <style>
    :root{
      --bg:#0b1220; --panel:#071428; --user:#1867ff; --bot:#2dd4bf;
      --muted:#9aa6b2; --glass: rgba(255,255,255,0.03);
      --radius:14px;
      color-scheme: dark;
    }
    *{box-sizing:border-box;margin:0;padding:0}
    html,body{height:100%}
    body{
      font-family: Inter, "Noto Sans Bengali", system-ui, -apple-system, Roboto, Arial;
      background: linear-gradient(180deg,#051025,#071428 60%);
      color:#e6eef6;
      display:flex;align-items:center;justify-content:center;padding:18px;
    }

    .app{
      width:100%;max-width:920px;height:90vh;border-radius:16px;
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      box-shadow:0 15px 50px rgba(2,6,23,0.7);display:grid;grid-template-columns:320px 1fr;overflow:hidden;
      border:1px solid rgba(255,255,255,0.04);
    }

    /* LEFT - sidebar */
    .sidebar{background:var(--panel);padding:16px;display:flex;flex-direction:column;gap:12px}
    .brand{font-weight:800;font-size:1.05rem;display:flex;gap:10px;align-items:center}
    .new-btn{margin-top:6px;padding:10px;border-radius:10px;background:linear-gradient(90deg,var(--user),#5a9cff);border:none;color:white;font-weight:700;cursor:pointer}
    .convos{margin-top:6px;overflow:auto;padding-right:6px;flex:1}
    .conv{padding:10px;border-radius:10px;margin-bottom:8px;background:transparent;border:1px solid rgba(255,255,255,0.02);cursor:pointer}
    .conv.active{background:rgba(255,255,255,0.02)}

    /* RIGHT - chat area */
    .chat-area{display:flex;flex-direction:column;gap:12px;padding:12px 18px;background:linear-gradient(180deg, rgba(255,255,255,0.01), transparent)}
    .topbar{display:flex;align-items:center;justify-content:space-between}
    .controls{display:flex;gap:8px;align-items:center}
    .toggle{display:flex;gap:6px;align-items:center;font-size:0.9rem;color:var(--muted)}
    .messages{flex:1;overflow:auto;padding:12px;display:flex;flex-direction:column;gap:12px}
    .msg{max-width:78%;padding:12px;border-radius:12px;line-height:1.4}
    .msg.user{margin-left:auto;background:linear-gradient(90deg,var(--user),#5a9cff);color:white;border-bottom-right-radius:2px}
    .msg.bot{margin-right:auto;background:rgba(255,255,255,0.03);color: #eaf6f2;border-bottom-left-radius:2px;border:1px solid rgba(255,255,255,0.02)}
    .meta{font-size:0.85rem;color:var(--muted);margin-top:6px}

    .composer{display:flex;gap:8px;align-items:end;padding:12px;border-top:1px solid rgba(255,255,255,0.02);background:linear-gradient(180deg, transparent, rgba(255,255,255,0.01))}
    .input{flex:1;border-radius:10px;padding:10px;border:1px solid rgba(255,255,255,0.03);background:var(--glass);color:inherit;min-height:44px;resize:none;outline:none}
    .send{padding:10px 14px;border-radius:10px;background:linear-gradient(90deg,var(--bot),#3be5c4);border:none;color:#022a27;font-weight:800;cursor:pointer}

    .typing{font-style:italic;color:var(--muted)}
    .small{font-size:0.9rem;color:var(--muted)}

    /* responsive */
    @media(max-width:880px){
      .app{grid-template-columns:1fr; height:92vh}
      .sidebar{display:none}
    }
  </style>
</head>
<body>
  <div class="app" role="application" aria-label="ChatLike app">
    <!-- SIDEBAR -->
    <aside class="sidebar" aria-label="conversations">
      <div class="brand">💬 ChatLike <span style="font-weight:600;color:var(--muted);font-size:0.9rem;margin-left:8px">বাংলা/ইংরেজি</span></div>
      <button class="new-btn" id="newConvBtn">+ New Chat</button>
      <div class="small">Mode: <span id="modeLabel">Mock (local)</span></div>
      <div class="convos" id="convos"></div>
      <div class="small" style="text-align:center;margin-top:6px;color:var(--muted)">Conversation history saved locally</div>
    </aside>

    <!-- CHAT AREA -->
    <section class="chat-area" aria-live="polite">
      <div class="topbar">
        <div>
          <strong id="convTitle">New chat</strong>
          <div class="small meta" id="convMeta">No messages yet</div>
        </div>
        <div class="controls">
          <label class="toggle"><input type="checkbox" id="realToggle"> Use real API</label>
          <button class="small" id="clearBtn">Clear All</button>
        </div>
      </div>

      <div class="messages" id="messages" tabindex="0"></div>

      <form id="composer" class="composer" autocomplete="off">
        <textarea id="input" class="input" placeholder="এখানে টাইপ করো... (Enter পাঠাবে, Shift+Enter নতুন লাইন)" rows="1"></textarea>
        <button class="send" type="submit">Send</button>
      </form>
    </section>
  </div>

  <script>
    // Simple Chat-like UI with local mock mode + optional server integration
    (function(){
      // DOM
      const convosEl = document.getElementById('convos');
      const newConvBtn = document.getElementById('newConvBtn');
      const messagesEl = document.getElementById('messages');
      const composer = document.getElementById('composer');
      const inputEl = document.getElementById('input');
      const convTitle = document.getElementById('convTitle');
      const convMeta = document.getElementById('convMeta');
      const realToggle = document.getElementById('realToggle');
      const modeLabel = document.getElementById('modeLabel');
      const clearBtn = document.getElementById('clearBtn');

      // state
      let conversations = JSON.parse(localStorage.getItem('chatlike.convos')||'[]');
      let active = localStorage.getItem('chatlike.active') || null;

      // helpers
      function save(){
        localStorage.setItem('chatlike.convos', JSON.stringify(conversations));
      }
      function setActive(id){
        active = id;
        localStorage.setItem('chatlike.active', id);
        render();
      }
      function newConversation(){
        const id = 'c_'+Date.now();
        conversations.unshift({ id, title:'New chat', created:Date.now(), messages:[] });
        save();
        setActive(id);
      }
      function clearAll(){
        if(!confirm('সব কনভারসেশন মুছে ফেলতে চান?')) return;
        conversations = []; active = null;
        save(); localStorage.removeItem('chatlike.active'); render();
      }

      // UI rendering
      function renderConvos(){
        convosEl.innerHTML = '';
        conversations.forEach(c=>{
          const d = document.createElement('div');
          d.className = 'conv' + (c.id===active ? ' active':'');
          d.innerHTML = <div style="font-weight:700">${c.title}</div><div class="small">${c.messages.length} messages</div>;
          d.onclick = ()=> setActive(c.id);
          convosEl.appendChild(d);
        });
      }

      function renderMessages(){
        messagesEl.innerHTML = '';
        const conv = conversations.find(c=>c.id===active);
        if(!conv){
          convTitle.textContent = 'New chat';
          convMeta.textContent = 'No messages yet';
          return;
        }
        convTitle.textContent = conv.title || 'Chat';
        convMeta.textContent = new Date(conv.created).toLocaleString();

        conv.messages.forEach(m=>{
          const el = document.createElement('div');
          el.className = 'msg ' + (m.role==='user' ? 'user':'bot');
          el.innerHTML = <div>${escapeHtml(m.content)}</div>;
          messagesEl.appendChild(el);
        });
        // scroll to bottom
        messagesEl.scrollTop = messagesEl.scrollHeight;
      }

      function render(){
        renderConvos();
        renderMessages();
        modeLabel.textContent = realToggle.checked ? 'Real API (proxy required)' : 'Mock (local)';
      }

      function escapeHtml(s){
        return s.replaceAll('&','&amp;').replaceAll('<','&lt;').replaceAll('>','&gt;').replaceAll('\n','<br>');
      }

      // send message flow
      async function sendMessage(text){
        if(!text.trim()) return;
        let conv = conversations.find(c=>c.id===active);
        if(!conv){ newConversation(); conv = conversations[0]; }
        // push user
        conv.messages.push({role:'user', content:text, ts:Date.now()});
        conv.title = text.slice(0,40) + (text.length>40? '...':'');
        save(); render();

        // push typing indicator
        const typing = {role:'bot', content:'Typing…', temp:true};
        conv.messages.push(typing); render(); messagesEl.scrollTop = messagesEl.scrollHeight;

        try{
          if(realToggle.checked){
            // send to /api/chat (you must create server-side endpoint that calls OpenAI or similar)
            // Request body format: { message: text, convId: conv.id }
            // Response expected JSON: { reply: "AI answer text" }
            const res = await fetch('/api/chat', {
              method:'POST',
              headers:{'Content-Type':'application/json'},
              body: JSON.stringify({ message: text, convId: conv.id })
            });
            if(!res.ok) throw new Error('Network response not ok: '+res.status);
            const data = await res.json();
            // replace typing with real reply
            conv.messages.pop();
            conv.messages.push({role:'bot', content: data.reply||'No reply from server', ts:Date.now()});
          } else {
            // MOCK local reply (simple rule-based / random)
            await delay(700 + Math.random()*900);
            const reply = mockReply(text);
            conv.messages.pop(); // remove typing
            conv.messages.push({role:'bot', content:reply, ts:Date.now()});
          }
        }catch(err){
          conv.messages.pop();
          conv.messages.push({role:'bot', content:'Error: ' + String(err), ts:Date.now()});
        }
        save(); render();
      }

      function delay(ms){ return new Promise(r=>setTimeout(r, ms)); }

      function mockReply(userText){
        // very simple mock — you can expand rules
        const txt = userText.toLowerCase();
        if(txt.includes('হ্যালো') || txt.includes('hello')) return 'হ্যালো! কীভাবে সাহায্য করতে পারি?';
        if(txt.includes('কে তুমি') || txt.includes('who are you')) return 'আমি একটি লোকাল সিমুলেটেড চ্যাটবট — তুমি চাইলে রিয়েল এপিআই যুক্ত করতে পারো।';
        if(txt.includes('কেমন') || txt.includes('how are')) return 'ভালো আছি, ধন্যবাদ! তুমি বলো কেমন আছো?';
        // fallback: echo + small paraphrase
        const echoes = [
          'দেখি—তুমি বলেছো: "'+userText.slice(0,80)+'" ...আর কোনো প্রশ্ন আছে?',
          'ভালো প্রশ্ন! একটু সময় দিন, আমি ব্যাখ্যা করছি...',
          'এই বিষয়ে আরো কিছুর প্রয়োজন হলে বলো।'
        ];
        return echoes[Math.floor(Math.random()*echoes.length)];
      }

      // events
      composer.addEventListener('submit', async function(e){
        e.preventDefault();
        const text = inputEl.value;
        inputEl.value = '';
        inputEl.style.height = '44px';
        await sendMessage(text);
      });

      // send on Enter, allow Shift+Enter newline
      inputEl.addEventListener('keydown', function(e){
        if(e.key === 'Enter' && !e.shiftKey){
          e.preventDefault();
          composer.requestSubmit();
        }
        // auto-resize
        setTimeout(()=> inputEl.style.height = Math.min(200, inputEl.scrollHeight) + 'px',0);
      });

      newConvBtn.addEventListener('click', newConversation);
      clearBtn.addEventListener('click', clearAll);
      realToggle.addEventListener('change', render);

      // init: if no conv exists create one
      if(conversations.length===0) newConversation();
      if(!active) active = conversations[0].id;
      render();

      // expose small helper for debugging
      window.chatLike = {conversations, save, newConversation, setActive, sendMessage};
    })();
  </script>

  <!--
    === Notes for adding REAL AI backend ===

    1) Browser should NOT call OpenAI directly with your secret key.
       Instead create a server endpoint (example below) that receives POST /api/chat
       and forwards the message to OpenAI (or other model), then returns { reply: "..." } to the browser.

    2) Minimal Node/Express example (save as server.js, run with NODE, install express & node-fetch/openai libs):

    // --- server.js (example, DO NOT run in browser) ---
    const express = require('express');
    const fetch = require('node-fetch'); // or use OpenAI official SDK
    const app = express();
    app.use(express.json());
    app.post('/api/chat', async (req,res) => {
      const userMsg = req.body.message || '';
      // Example using OpenAI Chat completions (pseudo)
      try {
        const openaiRes = await fetch('https://api.openai.com/v1/chat/completions',{
          method:'POST',
          headers:{
            'Content-Type':'application/json',
            'Authorization':'Bearer YOUR_OPENAI_KEY'
          },
          body: JSON.stringify({
            model: 'gpt-4o-mini', // or appropriate model
            messages: [{role:'user',content:userMsg}],
            max_tokens: 512
          })
        });
        const j = await openaiRes.json();
        const reply = j.choices?.[0]?.message?.content || 'No reply';
        res.json({ reply });
      } catch(e){
        res.status(500).json({ error: String(e) });
      }
    });
    app.listen(3000, ()=> console.log('Server running on :3000'));
    // ---------------------------------------------------

    3) Deploy this server somewhere (Railway, Render, Heroku, Vercel Serverless, etc.)
       Then set the browser UI toggle "Use real API" and ensure the endpoint path matches (/api/chat).
  -->

</body>
</html>
