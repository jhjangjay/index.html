<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>화성 AI 대화봇</title>
<style>
  html,body{
    margin:0;padding:0;background:transparent;
    font-family:-apple-system,BlinkMacSystemFont,"Noto Sans KR",sans-serif;
  }
  #app{
    width:100%;max-width:900px;margin:0 auto;
    display:flex;flex-direction:column;align-items:center;
  }
  h1{
    color:#fff;font-size:34px;text-align:center;margin:40px 0 10px;font-weight:900;
    text-shadow:0 2px 12px rgba(0,0,0,.4);
  }
  small{
    color:#fff;opacity:.85;margin-bottom:20px;display:block;text-align:center;
  }
  #chat{
    width:100%;
    background:rgba(255,255,255,.95);
    border-radius:16px;
    padding:16px;
    box-shadow:0 0 20px rgba(0,0,0,.2);
    height:400px;              /* 고정 높이 */
    overflow-y:auto;           /* 자동 스크롤 */
    display:flex;flex-direction:column;gap:10px;
  }
  .bubble{
    padding:12px 16px;border-radius:14px;line-height:1.6;white-space:pre-wrap;
    box-shadow:0 3px 10px rgba(0,0,0,.1);
    max-width:80%;
  }
  .user{align-self:flex-end;background:#111;color:#fff;}
  .ai{align-self:flex-start;background:#f5f5f5;color:#000;position:relative;}
  .copy-btn{
    position:absolute;top:4px;right:8px;
    font-size:11px;border:none;background:#ddd;border-radius:10px;padding:2px 6px;cursor:pointer;
  }
  .input-wrap{
    display:flex;gap:8px;margin-top:16px;width:100%;
  }
  #q{
    flex:1;padding:12px 16px;border-radius:999px;border:none;font-size:15px;
  }
  #send{
    background:#111;color:#fff;border:none;border-radius:999px;padding:12px 20px;cursor:pointer;font-weight:700;
  }
</style>
</head>
<body>
<div id="app">
  <h1>화성에 대해 무엇이든 물어보세요</h1>
  <small>Ask anything about Mars • 일상 대화도 가능</small>
  <div id="chat"></div>
  <div class="input-wrap">
    <input id="q" placeholder="예) 화성의 대기는 어때?" />
    <button id="send">보내기</button>
  </div>
</div>

<script>
const chat=document.getElementById('chat');
const qEl=document.getElementById('q');
const send=document.getElementById('send');

const GEMINI_API_KEY="";  // ← 여기에 Google API 키 입력

function addBubble(text,who){
  const div=document.createElement('div');
  div.className='bubble '+who;
  div.textContent=text;
  if(who==='ai'){
    const copy=document.createElement('button');
    copy.textContent='복사';
    copy.className='copy-btn';
    copy.onclick=()=>{navigator.clipboard.writeText(div.textContent);copy.textContent='완료';setTimeout(()=>copy.textContent='복사',1000)};
    div.appendChild(copy);
  }
  chat.appendChild(div);
  chat.scrollTop=chat.scrollHeight;
}

async function ask(){
  const q=qEl.value.trim();
  if(!q)return;
  addBubble(q,'user');
  qEl.value='';
  addBubble('답변 중...','ai');
  chat.scrollTop=chat.scrollHeight;

  try{
    if(!GEMINI_API_KEY){
      chat.lastChild.textContent='⚠️ API 키가 설정되지 않았습니다. Google Makersuite 키를 추가해주세요.';
      return;
    }
    const r=await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash-latest:generateContent?key=${GEMINI_API_KEY}`,{
      method:'POST',headers:{'Content-Type':'application/json'},
      body:JSON.stringify({
        contents:[{parts:[{text:`너는 한국어로 대화하는 AI야. 과학, 우주, 일상에 대해 간결하고 따뜻하게 대답해줘.\n질문:${q}`}]}]
      })
    });
    const j=await r.json();
    const a=j?.candidates?.[0]?.content?.parts?.[0]?.text||'응답을 받을 수 없습니다.';
    chat.lastChild.textContent=a;
  }catch(e){
    chat.lastChild.textContent='⚠️ 네트워크 오류 혹은 CORS 제한으로 응답 실패';
  }
}

send.onclick=ask;
qEl.addEventListener('keydown',e=>{if(e.key==='Enter'){ask();}});
</script>
</body>
</html>
