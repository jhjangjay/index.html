<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>화성 지식 챗봇 – Free (WebLLM 3B)</title>
<style>
  body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,'Apple SD Gothic Neo','Noto Sans KR',sans-serif;background:#0b0b0b url('https://images.unsplash.com/photo-1531045535792-9c7cdd8f3c6e?q=80&w=1600&auto=format&fit=crop') center/cover fixed}
  #wrap{max-width:900px;margin:60px auto;padding:0 16px;text-align:center}
  h1{margin:0 0 12px;color:#fff;font-weight:800;font-size:40px;letter-spacing:-.3px;text-shadow:0 2px 18px rgba(0,0,0,.35)}
  h1 small{display:block;margin-top:6px;font:700 16px/1.4 system-ui;opacity:.9}
  .capsule{display:inline-flex;align-items:center;gap:10px;border:1px solid rgba(255,255,255,.35);background:#fff;border-radius:999px;padding:10px 14px;box-shadow:0 8px 30px rgba(0,0,0,.25)}
  .capsule input{width:560px;border:none;outline:none;font-size:16px;color:#111}
  .capsule button{border:1px solid #e5e7eb;background:#111;color:#fff;padding:10px 16px;border-radius:999px;cursor:pointer;font-weight:700}
  #card{display:none;margin:16px auto 0;border:1px solid #e5e7eb;background:#fff;border-radius:16px;padding:18px;box-shadow:0 8px 30px rgba(0,0,0,.10);text-align:left;max-width:900px}
  #title{font-weight:800;font-size:18px;margin-bottom:8px;color:#111}
  #body{white-space:pre-wrap;font-size:15px;line-height:1.75;color:#111}
  #status{font-size:12px;color:#6b7280;margin-left:8px}
  .sub{font-weight:700;margin:16px 0 6px}
  .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(140px,1fr));gap:10px}
  .thumb{border:1px solid #e5e7eb;border-radius:12px;overflow:hidden}
  .thumb img{display:block;width:100%;height:120px;object-fit:cover}
  .thumb a{display:block;padding:6px 8px;font-size:12px;color:#2563eb;text-decoration:none}
  .muted{color:#6b7280}
  #note{margin-top:8px;color:#fff;font-size:12px;text-shadow:0 1px 10px rgba(0,0,0,.35)}
</style>
</head>
<body>
<div id="wrap">
  <h1>화성에 대해 무엇이든 물어보세요 <small>Ask anything about Mars</small></h1>
  <div class="capsule">
    <input id="q" placeholder="질문을 입력하세요…" />
    <button id="ask">보내기</button>
  </div>

  <div id="card">
    <div style="display:flex;align-items:center;gap:8px">
      <div id="title">AI 답변</div>
      <div id="status"></div>
    </div>
    <div id="body"></div>
  </div>
  <div id="note"></div>
</div>

<!-- WebLLM (온디바이스 LLM) -->
<script src="https://cdn.jsdelivr.net/npm/webllm/dist/webllm.min.js"></script>
<script>
(function(){
  const qEl = document.getElementById('q');
  const ask = document.getElementById('ask');
  const card = document.getElementById('card');
  const body = document.getElementById('body');
  const title = document.getElementById('title');
  const note = document.getElementById('note');
  const statusEl = document.getElementById('status');

  // 안전 필터
  const danger=/자살|극단|해치|무기|폭발|폭탄|응급 수술|정확한 단계|단계별|구체적 절차|수리 단계/i;

  // ===== 무료 공개 API =====
  async function wikiSearch(q,lang='ko',limit=4){
    try{const r=await fetch(`https://${lang}.wikipedia.org/w/rest.php/v1/search/title?q=${encodeURIComponent(q)}&limit=${limit}`);if(!r.ok)return[];const j=await r.json();return(j.pages||[]).map(p=>({title:p.title,lang}))}catch(_){return[]}
  }
  async function wikiSummary(title,lang='ko'){
    try{const r=await fetch(`https://${lang}.wikipedia.org/api/rest_v1/page/summary/${encodeURIComponent(title)}`);if(!r.ok)return null;const j=await r.json();return{title:j.title||title,extract:(j.extract||'').trim(),url:j?.content_urls?.desktop?.page}}catch(_){return null}
  }
  async function wikidata(q){
    try{const r=await fetch(`https://www.wikidata.org/w/api.php?action=wbsearchentities&search=${encodeURIComponent(q)}&language=ko&format=json&limit=3&origin=*`);if(!r.ok)return[];const j=await r.json();return(j.search||[]).map(x=>({id:x.id,label:x.label,desc:x.description||''}))}catch(_){return[]}
  }
  async function nasaImages(q,limit=6){
    try{const r=await fetch(`https://images-api.nasa.gov/search?q=${encodeURIComponent(q)}&media_type=image`);if(!r.ok)return[];const j=await r.json();return(j.collection?.items||[]).slice(0,limit).map(it=>{const d=it.data?.[0]||{};const img=(it.links?.[0]?.href)||'';return{title:d.title||'NASA Image',thumb:img,url:(it.href||img)}})}catch(_){return[]}
  }
  async function nasaAPOD(){try{const r=await fetch('https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY');if(!r.ok)return null;return await r.json()}catch(_){return null}}
  async function crossref(q,rows=5){
    try{const r=await fetch(`https://api.crossref.org/works?query=${encodeURIComponent(q)}&rows=${rows}`);if(!r.ok)return[];const j=await r.json();return(j.message?.items||[]).map(x=>({title:(x.title||['Untitled'])[0],year:x.issued?.['date-parts']?.[0]?.[0],url:x.URL,doi:x.DOI}))}catch(_){return[]}
  }
  async function openlib(q,limit=5){
    try{const r=await fetch(`https://openlibrary.org/search.json?q=${encodeURIComponent(q)}&limit=${limit}`);if(!r.ok)return[];const j=await r.json();return(j.docs||[]).map(d=>({title:d.title,author:(d.author_name||[])[0]||'',year:d.first_publish_year||'',url:(d.key?`https://openlibrary.org${d.key}`:'' )}))}catch(_){return[]}
  }
  const WEATHER=/날씨|기온|강수|비|눈|예보|풍속|습도|체감|자외선/i;
  async function geocode(place){const r=await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(place)}&count=1&language=ko&format=json`);if(!r.ok)return null;const j=await r.json();return j.results?.[0]||null}
  async function forecast(lat,lon,tz='auto'){const u=`https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,apparent_temperature,relative_humidity_2m,wind_speed_10m,wind_direction_10m,weather_code,precipitation&daily=temperature_2m_max,temperature_2m_min,precipitation_sum,precipitation_probability_max,weather_code&timezone=${tz}`;const r=await fetch(u);if(!r.ok)return null;return await r.json()}
  const wmo=(c)=>({0:'맑음',1:'대체로 맑음',2:'부분적 흐림',3:'흐림',45:'안개',51:'이슬비',61:'비',71:'눈',95:'뇌우'})[c]||'기상';

  // ===== WebLLM (3B로 품질 ↑, 미지원 브라우저는 자동 폴백) =====
  let engine=null, ready=false, loading=false, modelTried=false;
  async function ensureLLM(){
    if(ready||loading||modelTried) return; loading=true; modelTried=true;
    if(!('gpu' in navigator)){ note.textContent='⚠️ WebGPU 미지원 브라우저입니다. (최신 Chrome/Edge 권장)'; loading=false; return; }
    // 먼저 3B 시도, 실패 시 1B 폴백
    const models=[
      'Llama-3.2-3B-Instruct-q4f16_1-MLC',
      'Llama-3.2-1B-Instruct-q4f16_1-MLC'
    ];
    for(const m of models){
      try{
        await (engine = await webllm.CreateMLCEngine(m,{
          initProgressCallback:(p)=>{
            const pct = typeof p.progress==='number' ? Math.round(p.progress*100) : null;
            note.textContent = pct!=null? `모델 로딩 중… ${pct}% (${m})` : (p.text||`모델 로딩 중… (${m})`);
          }
        }));
        ready=true; loading=false; note.textContent='모델 준비 완료!'; return;
      }catch(e){
        console.warn('모델 로딩 실패:', m, e);
      }
    }
    loading=false; note.textContent='온디바이스 LLM 로딩 실패 – 검색 요약 모드로 동작합니다.';
  }
  async function llmAnswer(context,question){
    await ensureLLM();
    if(!engine) return ''; // LLM 없이도 결과 제공
    const sys='너는 화성/우주/기상 분야 공개 자료를 토대로 사실 기반 한국어 답변을 주는 조수다. 근거가 불충분하면 모른다고 말해라. 실제 생존/의료/정비의 단계별 절차는 제공하지 마라.';
    const prompt=`[질문]\n${question}\n\n[참고 요약]\n${context}\n\n[응답 지침]\n- 핵심만 간결히, 과학적 사실 위주\n- 숫자/단위 포함\n- 불확실시 "확실치 않음" 표기\n\n[최종 답변]`;
    const out=await engine.chat.completions.create({messages:[{role:'system',content:sys},{role:'user',content:prompt}],temperature:0.25,max_tokens:320});
    return out?.choices?.[0]?.message?.content||'';
  }

  function take(t){return (t||'').split(/[。\.\n]/).filter(Boolean).slice(0,2).join(' ').slice(0,300)}
  function section(h,html){return `<div class="sub">${h}</div><div>${html}</div>`}
  function bullets(a){return `<ul style="margin:6px 0 0 18px">${a.map(x=>`<li>${x}</li>`).join('')}</ul>`}

  async function answer(){
    const q=qEl.value.trim(); if(!q) return;
    card.style.display='block'; title.textContent='AI 답변'; body.textContent='자료 수집 중…'; statusEl.textContent='';

    if(danger.test(q)){ body.textContent='⚠️ 안전상의 이유로 실제 생존·의료·정비의 단계별 절차는 제공할 수 없습니다. 개념 수준의 설명만 가능합니다.'; return; }

    // 1) 날씨(지구) 질문 즉답
    if(WEATHER.test(q)){
      const place=q.replace(WEATHER,'').replace(/날씨|어때|예보|오늘|내일/gi,'').trim()||'Seoul';
      const g=await geocode(place); if(!g){ body.textContent='지명을 찾지 못했습니다. 예: 서울 날씨'; return; }
      const fc=await forecast(g.latitude,g.longitude,g.timezone||'auto'); if(!fc){ body.textContent='예보를 불러오지 못했습니다.'; return; }
      const cur=fc.current,d=fc.daily,i=0;
      const now=`<div class="grid" style="grid-template-columns:140px 1fr">
        <div class="muted">위치</div><div>${g.name}${g.admin1? ', '+g.admin1:''} (${g.country_code})</div>
        <div class="muted">현재</div><div>${cur.temperature_2m}°C (체감 ${cur.apparent_temperature}°C), ${wmo(cur.weather_code)}</div>
        <div class="muted">습도</div><div>${cur.relative_humidity_2m}%</div>
        <div class="muted">풍속/풍향</div><div>${cur.wind_speed_10m} m/s, ${cur.wind_direction_10m}°</div>
      </div>`;
      const today=bullets([
        `오늘: 최고 ${d.temperature_2m_max[i]}°C / 최저 ${d.temperature_2m_min[i]}°C, ${wmo(d.weather_code[i])}`,
        d.precipitation_probability_max?.[i]!=null?`강수확률 최대 ${d.precipitation_probability_max[i]}%`:null,
        d.precipitation_sum?.[i]!=null?`강수량 ${d.precipitation_sum[i]} mm`:null
      ].filter(Boolean));
      body.innerHTML=section('현재 상황',now)+section('오늘 요약',today);
      return;
    }

    try{
      statusEl.textContent='위키·NASA·Wikidata 검색 중…';
      const ko=await wikiSearch(q,'ko',4), en=await wikiSearch(q,'en',3);
      const picks=[...ko,...en].slice(0,6);
      const wikis=[]; for(const p of picks){ const s=await wikiSummary(p.title,p.lang); if(s&&s.extract) wikis.push(s); }
      const wd=await wikidata(q);
      let thumbs=[],apod=null;
      if(/(nasa|우주|space|화성|mars|사진|image)/i.test(q)){ thumbs=await nasaImages(q,6); apod=await nasaAPOD(); }
      const papers=/논문|paper|연구|학술|doi/i.test(q)? await crossref(q,5):[];
      const books=/책|도서|book/i.test(q)? await openlib(q,5):[];

      // 컨텍스트 만들기 → LLM 요약
      const context=[
        ...wikis.slice(0,4).map(s=>`${s.title}: ${take(s.extract)}`),
        ...wd.slice(0,3).map(x=>`${x.label}(${x.id}): ${x.desc}`)
      ].join('\n');
      let llmOut=''; if(context){ statusEl.textContent='요약/정리 중…'; llmOut=await llmAnswer(context,q); }

      const secs=[];
      if(llmOut) secs.push(section('요약 답변', llmOut));
      if(wikis.length) secs.push(section('백과사전 요약',
        bullets(wikis.slice(0,4).map(s=>`${s.title}: ${take(s.extract)}  —  ${s.url?`<a class="muted" href="${s.url}" target="_blank">원문</a>`:''}`))
      ));
      if(wd.length) secs.push(section('Wikidata',
        bullets(wd.slice(0,3).map(x=>`${x.label}: ${x.desc} <span class="muted">[${x.id}]</span>`))
      ));
      if(thumbs.length||apod){
        const grid=thumbs.length? `<div class="grid">${thumbs.map(t=>`<div class="thumb"><img src="${t.thumb||t.url}" alt=""><a href="${t.url}" target="_blank">${t.title}</a></div>`).join('')}</div>` : '';
        const ap = apod? `<div class="thumb" style="margin-top:10px"><img src="${apod.url}" alt=""><a href="${apod.hdurl||apod.url}" target="_blank">APOD: ${apod.title} (${apod.date})</a></div>` : '';
        secs.push(section('NASA 이미지', grid+ap));
      }
      if(papers.length) secs.push(section('논문(Top)',
        bullets(papers.map(p=>`${p.title} (${p.year||''}) — <a href="${p.url}" target="_blank">${p.doi||'링크'}</a>`))
      ));
      if(books.length) secs.push(section('도서(Top)',
        bullets(books.map(b=>`${b.title}${b.author? ' — '+b.author:''}${b.year? ' ('+b.year+')':''} — <a href="${b.url}" target="_blank">OpenLibrary</a>`))
      ));

      body.innerHTML = secs.length? secs.join('') : '관련 공개 자료를 찾지 못했습니다. 질문을 더 구체화해 주세요.';
      statusEl.textContent='';
    }catch(e){
      console.error(e);
      body.textContent='무료 출처를 불러오는 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.';
      statusEl.textContent='';
    }
  }

  ask.addEventListener('click', answer);
  qEl.addEventListener('keydown', e=>{ if(e.key==='Enter'){ e.preventDefault(); answer(); }});

  // 최초 LLM 준비(비동기)
  ensureLLM();
})();
</script>
</body>
</html>
