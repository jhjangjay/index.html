<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>화성 지식 & 일상 대화 – Free On-device AI</title>
<style>
  :root{
    --maxw: 900px; --ring: rgba(0,0,0,.25); --border:#e5e7eb; --muted:#6b7280; --ink:#111; --bgwhite:#fff;
  }
  body{
    margin:0;
    font-family:system-ui,-apple-system,Segoe UI,Roboto,'Apple SD Gothic Neo','Noto Sans KR',sans-serif;
    background:#0b0b0b url('https://images.unsplash.com/photo-1531045535792-9c7cdd8f3c6e?q=80&w=1600&auto=format&fit=crop') center/cover fixed;
  }
  .mw{max-width:var(--maxw);margin:60px auto;padding:0 16px;text-align:center}
  .ttl{margin:0 0 20px;color:#fff;font-weight:800;font-size:40px;letter-spacing:-.3px;text-shadow:0 2px 18px rgba(0,0,0,.35)}
  .ttl small{display:block;margin-top:6px;font:700 16px/1.4 system-ui;opacity:.9}
  .capsule{display:inline-flex;align-items:center;gap:10px;background:var(--bgwhite);border:none;border-radius:999px;padding:12px 14px;box-shadow:0 8px 30px var(--ring);width:100%;max-width:760px}
  .capsule input{flex:1;min-width:0;border:none;outline:none;font-size:16px;line-height:1.4;color:var(--ink);background:transparent}
  .capsule button{border:none;background:#111;color:#fff;padding:12px 18px;border-radius:999px;cursor:pointer;font-weight:700;min-height:44px}
  .card{margin:20px auto 0;border:none;background:#fff;border-radius:16px;padding:18px;box-shadow:0 8px 30px rgba(0,0,0,.15);text-align:left;max-width:var(--maxw)}
  .head{display:flex;align-items:center;gap:8px}
  .title{font-weight:800;font-size:18px;color:var(--ink)}
  .muted{color:var(--muted);font-size:12px}
  .body{white-space:pre-wrap;font-size:15px;line-height:1.75;color:var(--ink)}
  .note{margin-top:10px;color:#fff;font-size:12px;text-shadow:0 1px 10px rgba(0,0,0,.35)}
  @media (max-width: 768px){
    .mw{margin:40px auto; padding:0 12px}
    .ttl{font-size:32px}
    .ttl small{font-size:14px}
    .capsule{padding:10px 12px;box-shadow:0 6px 22px var(--ring);max-width:100%}
    .capsule button{padding:10px 14px}
    .card{padding:14px;border-radius:14px}
    .title{font-size:16px}
    .body{font-size:14.5px;line-height:1.7}
  }
  @supports (padding-bottom: env(safe-area-inset-bottom)){
    .mw{padding-bottom:calc(16px + env(safe-area-inset-bottom))}
  }
  .capsule input:focus,.capsule button:focus{outline:2px solid #111;outline-offset:2px;border-radius:999px}
</style>
</head>
<body>
<div class="mw">
  <h1 class="ttl">
    화성에 대해 무엇이든 물어보세요
    <small>Ask anything about Mars • 일상 대화도 가능</small>
  </h1>

  <div class="capsule">
    <input id="q" placeholder="예) 화성의 대기는? / 오늘 기분이 안좋아…" inputmode="search"/>
    <button id="ask" aria-label="보내기">보내기</button>
  </div>

  <div id="card" class="card" hidden>
    <div class="head">
      <div class="title">AI 답변</div>
      <div id="status" class="muted"></div>
    </div>
    <div id="body" class="body"></div>
  </div>

  <div id="note" class="note"></div>
</div>

<!-- WebGPU 가능 시: WebLLM -->
<script src="https://cdn.jsdelivr.net/npm/webllm/dist/webllm.min.js"></script>
<!-- WebGPU 불가 시: CPU/WASM 대체 -->
<script src="https://cdn.jsdelivr.net/npm/@xenova/transformers/dist/transformers.min.js"></script>

<script>
(function(){
  const qEl = document.getElementById('q');
  const ask = document.getElementById('ask');
  const card = document.getElementById('card');
  const body = document.getElementById('body');
  const statusEl = document.getElementById('status');
  const note = document.getElementById('note');

  // ===== 요청 큐 (연속 질문 가능) =====
  const queue = [];
  let busy = false;
  function enqueue(text){
    const q = text.trim(); if(!q) return;
    queue.push(q);
    if(!busy) runNext();
  }
  async function runNext(){
    if(queue.length===0){ busy=false; return; }
    busy = true;
    const question = queue.shift();
    try{ await handleQuestion(question); }
    catch(e){ console.error(e); body.textContent='오류가 발생했습니다.'; }
    finally{ busy=false; runNext(); }
  }

  // ===== 대화 히스토리 =====
  const KEY='mars_chat_history_v2';
  let history = JSON.parse(localStorage.getItem(KEY)||'[]'); // [{role, content}]
  const save = ()=> localStorage.setItem(KEY, JSON.stringify(history.slice(-12)));

  // ===== 엔진: WebLLM(WebGPU) → 실패 시 Transformers.js(CPU) =====
  let webllmEngine = null;
  let wasmGen = null;

  async function ensureWebLLM(){
    if(webllmEngine || !('gpu' in navigator)) return; // WebGPU 없으면 스킵
    note.textContent='AI 모델(WebGPU) 불러오는 중…';
    try{
      webllmEngine = await webllm.CreateMLCEngine('Llama-3.2-1B-Instruct-q4f16_1-MLC',{
        initProgressCallback:(p)=>{
          if(typeof p.progress==='number') note.textContent = `로딩 중… ${Math.round(p.progress*100)}% (WebGPU)`;
        }
      });
      note.textContent='AI(WebGPU) 준비 완료!';
    }catch(e){
      console.warn('WebLLM 로딩 실패', e);
      webllmEngine = null;
    }
  }

  async function ensureWASM(){
    if(wasmGen) return;
    note.textContent='AI 모델(CPU) 불러오는 중… (최초 1회)';
    try{
      const { pipeline } = window.transformers;
      wasmGen = await pipeline('text-generation','Xenova/TinyLlama-1.1B-Chat-v1.0',{ quantized:true });
      note.textContent='AI(CPU) 준비 완료!';
    }catch(e){
      console.error(e);
      note.textContent='온디바이스 모델 로딩 실패(WebGPU/CPU). 간단 답변으로 대체합니다.';
    }
  }

  function baseSystem(hasContext){
    const a='너는 한국어로 대화하는 친절한 조수다. 질문이 과학적이면 정확한 사실 위주로 간결하게 설명하고, 일상적이면 공감과 간단한 제안을 한다.';
    const b='화성/우주/기상 분야에서는 공개 자료를 우선으로 삼고, 불확실하면 모른다고 말한다.';
    return hasContext ? a+'\n'+b : a;
  }

  // CPU 경량 모델용 ChatML 스타일 프롬프트
  function buildChatPrompt(q, ctx, hist){
    let prompt = "";
    prompt += "<|system|>\n" + baseSystem(!!ctx) + "\n";
    hist.slice(-8).forEach(m=>{
      if(m.role==='user') prompt += "<|user|>\n"+m.content+"\n";
      else prompt += "<|assistant|>\n"+m.content+"\n";
    });
    if(ctx) prompt += "<|user|>\n다음 참고 요약을 반영해서 답해줘:\n"+ctx+"\n";
    prompt += "<|user|>\n"+q+"\n<|assistant|>\n";
    return prompt;
  }

  // 간단 컨텍스트(위키 요약 1개) — 실패해도 무시
  async function contextFromWikipedia(q){
    try{
      const r=await fetch(`https://ko.wikipedia.org/w/rest.php/v1/search/title?q=${encodeURIComponent(q)}&limit=1`,
        {headers:{'Accept':'application/json'}});
      if(!r.ok) return '';
      const j=await r.json();
      const title=j.pages?.[0]?.title; if(!title) return '';
      const s=await fetch(`https://ko.wikipedia.org/api/rest_v1/page/summary/${encodeURIComponent(title)}`,
        {headers:{'Accept':'application/json'}});
      if(!s.ok) return '';
      const js=await s.json();
      return `${js.title}: ${(js.extract||'').slice(0,600)}`;
    }catch{ return ''; }
  }

  async function callLLM(question, context=''){
    // 1) WebGPU
    await ensureWebLLM();
    if(webllmEngine){
      const msgs=[{role:'system',content:baseSystem(!!context)}];
      history.slice(-8).forEach(m=>msgs.push(m));
      if(context) msgs.push({role:'user',content:`참고 요약:\n${context}`});
      msgs.push({role:'user',content:question});
      const out=await webllmEngine.chat.completions.create({messages:msgs,temperature:0.4,max_tokens:300});
      return out?.choices?.[0]?.message?.content||'';
    }
    // 2) CPU/WASM
    await ensureWASM();
    if(wasmGen){
      const prompt=buildChatPrompt(question, context, history);
      const outputs=await wasmGen(prompt,{max_new_tokens:220,temperature:0.7});
      const full=outputs[0].generated_text;
      return full.slice(prompt.length);
    }
    // 3) 최소 폴백
    return context ? '찾은 공개 자료를 요약해 드렸어요.' : '지금 환경에서는 간단한 설명만 가능합니다.';
  }

  async function handleQuestion(q){
    card.hidden=false;
    body.textContent='생각 중…';
    statusEl.textContent='';
    const ctx = await contextFromWikipedia(q); // 있으면 정확도↑
    const answer = await callLLM(q, ctx);
    body.textContent = answer || '응답 생성 실패';
    history.push({role:'user',content:q},{role:'assistant',content:answer});
    save();
  }

  // 이벤트
  ask.addEventListener('click', ()=>enqueue(qEl.value));
  qEl.addEventListener('keydown', e=>{ if(e.key==='Enter'){ e.preventDefault(); enqueue(qEl.value); }});

  // 백그라운드 프리로드
  (async()=>{ await ensureWebLLM(); if(!webllmEngine) await ensureWASM(); })();
})();
</script>
</body>
</html>
