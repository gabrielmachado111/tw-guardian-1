// guardian-main.js - coloque este arquivo no seu GitHub (https://raw.githubusercontent.com/gabrielmachado111/Guardian/main/guardian-main.js)
(async function(){
  // URL do arquivo de licenças
  const LICENSE_URL = "https://raw.githubusercontent.com/gabrielmachado111/Guardian/main/licenses.json";

  // Função para extrair o nick logado no topo do Tribal Wars
  function getCurrentNick(){
    let el=document.querySelector('#menu_row2 a[href*="screen=info_player"]');
    if(el) return el.textContent.trim();
    el=document.querySelector('.menu_column a[href*="screen=info_player"]');
    if(el) return el.textContent.trim();
    return null;
  }

  // Checa se o nick possui licença válida (data igual ou maior que hoje)
  async function checkLicense(nick){
    try{
      const resp = await fetch(LICENSE_URL + "?t=" + Date.now());
      if(!resp.ok) return false;
      const json = await resp.json();
      if(!json[nick]) return false;
      const expiry = new Date(json[nick]+"T23:59:59");
      return new Date() <= expiry;
    }catch(e){ return false; }
  }

  // Espera DOM
  function domReady(){
    return new Promise(res=>{
      if(document.readyState==="complete"||document.readyState==="interactive") res();
      else document.addEventListener("DOMContentLoaded",res,{once:true});
    });
  }

  await domReady();

  const nick = getCurrentNick();
  if(!nick){
    alert("Não foi possível identificar seu nick no topo da página!\nAcesse pelo perfil da conta Tribal Wars.");
    return;
  }

  // Verificação da licença
  const ok = await checkLicense(nick);
  if(!ok){
    alert("Seu nick '" + nick + "' não possui licença válida ou está vencida.\nContate o administrador para liberar acesso.");
    return;
  }

  // === TODO O SEU SCRIPT GUARDIAN VAI DAQUI PARA BAIXO ===
  // --- segue o script v1.5.2 com interface arrastável, exatamente como fornecido antes ---

  // ---------- Config ----------
  const POLL_MS=2000, GAP_SECONDS=30, RECENT_WINDOW_SECONDS=60, MAX_LOOKBACK_ROWS=80, RETRIGGER_COOLDOWN_MS=30000;

  // ---------- Keys ----------
  const K_ENABLED='tw_guard_enabled', K_SUSPECT='tw_guard_suspect', K_SUSPECT_ID='tw_guard_suspect_id',
        K_ACTION_FLAG='tw_guard_action_flag', K_LAST_TRIGGER_TS='tw_guard_last_trigger',
        K_UI_POS='tw_guard_ui_pos';

  // ---------- Context ----------
  const url=new URL(location.href), screen=url.searchParams.get('screen'), mode=url.searchParams.get('mode'),
        village=url.searchParams.get('village')||'';

  // ---------- Utils ----------
  function enabled(){ return GM_getValue(K_ENABLED,true); }
  function setEnabled(v){ GM_setValue(K_ENABLED,!!v); updateUi(); }
  function nowTs(){ return Date.now(); }
  function parsePtBrDateTime(diaMesStr,horaStr){
    const [monStrRaw, diaStr]=(diaMesStr||'').replace('.','').trim().split(/\s+/);
    const ptMon=['jan','fev','mar','abr','mai','jun','jul','ago','set','out','nov','dez'];
    const mi=ptMon.indexOf((monStrRaw||'').toLowerCase()); if(mi<0)return NaN;
    const [hh,mm]=(horaStr||'00:00').split(':').map(x=>parseInt(x,10)||0);
    const now=new Date(); return new Date(now.getFullYear(),mi,parseInt(diaStr,10)||1,hh,mm,0,0).getTime();
  }
  const reExp=/^(.+?)\s*foi expulso\/retirado da tribo por\s+(.+?)[\.\s]*$/i;
  function parseRow(tr){
    const tds=tr.querySelectorAll('td'); if(tds.length<2) return null;
    const parts=tds[0].innerText.replace(/\r/g,'').trim().split('\n').map(s=>s.trim()).filter(Boolean);
    if(parts.length<2) return null;
    const m=tds[1].innerText.trim().match(reExp); if(!m) return null;
    const ts=parsePtBrDateTime(parts[0],parts[1]); if(!isFinite(ts)) return null;
    return { ts, victim:m[1].replace(/\s*\.$/,'').trim(), author:m[2].replace(/\s*\.$/,'').trim() };
  }

  // ---------- UI (arrastável) ----------
  let uiRoot=null, statusSpan=null, toggleBtn=null;
  function ensureUi(){
    if (uiRoot && document.body.contains(uiRoot)) return;

    // container
    uiRoot=document.createElement('div');
    uiRoot.id='tw-guard-ui';
    uiRoot.style.cssText='position:fixed;right:16px;bottom:16px;z-index:2147483647;background:#101010;color:#eee;border:1px solid #444;border-radius:8px;min-width:220px;box-shadow:0 2px 8px rgba(0,0,0,.4);font:12px Arial';

    // header (área de arraste)
    const header=document.createElement('div');
    header.textContent='Tribe Guard v1.5.2';
    header.style.cssText='cursor:move;font-weight:bold;padding:8px 12px;border-bottom:1px solid #333;background:#181818;border-top-left-radius:8px;border-top-right-radius:8px;';

    // corpo
    const body=document.createElement('div');
    body.style.cssText='padding:10px 12px;';
    statusSpan=document.createElement('div'); statusSpan.style.cssText='margin-bottom:8px;';
    toggleBtn=document.createElement('button'); toggleBtn.style.cssText='padding:6px 10px;border:none;border-radius:5px;color:#fff;cursor:pointer;';
    toggleBtn.onclick=()=>setEnabled(!enabled());
    const hint=document.createElement('div'); hint.textContent='Arraste pelo topo para mover.'; hint.style.cssText='margin-top:6px;color:#bbb;';

    body.append(statusSpan,toggleBtn,hint);
    uiRoot.append(header, body);
    document.body.appendChild(uiRoot);

    // posição persistida
    const pos = GM_getValue(K_UI_POS, null);
    if (pos && typeof pos==='object') {
      if (pos.left!=null && pos.top!=null) {
        uiRoot.style.left = pos.left+'px';
        uiRoot.style.top  = pos.top+'px';
        uiRoot.style.right='auto';
        uiRoot.style.bottom='auto';
      }
    }

    // drag handlers
    let dragging=false, startX=0, startY=0, startLeft=0, startTop=0;
    header.addEventListener('mousedown', (e)=>{
      dragging=true;
      startX=e.clientX; startY=e.clientY;
      const rect=uiRoot.getBoundingClientRect();
      startLeft=rect.left; startTop=rect.top;
      document.addEventListener('mousemove', onMove);
      document.addEventListener('mouseup', onUp, { once:true });
      e.preventDefault();
    });
    function onMove(e){
      if(!dragging) return;
      const dx=e.clientX-startX, dy=e.clientY-startY;
      const nl = Math.min(window.innerWidth-40, Math.max(0, startLeft+dx));
      const nt = Math.min(window.innerHeight-40, Math.max(0, startTop+dy));
      uiRoot.style.left=nl+'px';
      uiRoot.style.top=nt+'px';
      uiRoot.style.right='auto';
      uiRoot.style.bottom='auto';
    }
    function onUp(){
      dragging=false;
      document.removeEventListener('mousemove', onMove);
      const rect=uiRoot.getBoundingClientRect();
      GM_setValue(K_UI_POS, { left: Math.round(rect.left), top: Math.round(rect.top) });
    }

    updateUi();
  }
  function updateUi(){
    if(!uiRoot) return;
    const on=enabled();
    statusSpan.textContent=on?`Status: LIGADO (atuais ≤ ${60}s)`: 'Status: DESLIGADO';
    toggleBtn.textContent=on?'Desligar':'Ligar';
    toggleBtn.style.background=on?'#d9534f':'#5cb85c';
  }

  // Recria UI ao navegar via Ajax
  function mountUiOnOverview(){
    const isOverview = (screen==='ally'&&mode==='overview');
    if(!isOverview) return;
    const mount=()=>ensureUi();
    if(document.readyState==='loading') document.addEventListener('DOMContentLoaded', mount, { once:true });
    else mount();
    const root=document.querySelector('#content_value')||document.body;
    const mo=new MutationObserver(()=>ensureUi());
    mo.observe(root,{childList:true,subtree:true});
  }

  // ---------- Overview detector ----------
  function runOverview(){
    if(!(screen==='ally'&&mode==='overview')) return;
    function trigger(author){
      if(!enabled()) return;
      const now=Date.now(), last=GM_getValue('tw_guard_last_trigger',0);
      if(now-last<RETRIGGER_COOLDOWN_MS) return;
      GM_setValue('tw_guard_last_trigger', now);
      GM_setValue(K_SUSPECT, author);
      GM_setValue(K_SUSPECT_ID, '');
      GM_setValue(K_ACTION_FLAG, 1);
      const membersUrl=location.origin+`/game.php?village=${url.searchParams.get('village')||''}&screen=ally&mode=members`;
      GM_openInTab(membersUrl,{active:true,insert:true});
    }
    function scan(){
      if(!enabled()) return;
      const container=document.querySelector('#content_value')||document.body;
      const rows=Array.from(container.querySelectorAll('table tr, .vis tr, .content-border tr')).slice(0,MAX_LOOKBACK_ROWS);
      const exps=[]; for(const tr of rows){ const e=parseRow(tr); if(e) exps.push(e); }
      exps.sort((a,b)=>a.ts-b.ts);
      const cutoff=Date.now()-RECENT_WINDOW_SECONDS*1000;
      const recent=exps.filter(e=>e.ts>=cutoff);
      for(let i=1;i<recent.length;i++){
        const p=recent[i-1], c=recent[i];
        if(c.author===p.author && (c.ts-p.ts)<=GAP_SECONDS*1000){ trigger(c.author); return; }
      }
    }
    setInterval(scan, POLL_MS);
    const target=document.querySelector('#content_value')||document.body;
    const mo=new MutationObserver(()=>scan());
    mo.observe(target,{childList:true,subtree:true});
  }

  // ---------- Members executor ----------
  function runMembers(){
    if(!(screen==='ally'&&mode==='members')) return;
    const sleep=ms=>new Promise(r=>setTimeout(r,ms));
    async function act(){
      if(!enabled()) return;
      if(!GM_getValue(K_ACTION_FLAG,0)) return;
      const suspectName=GM_getValue(K_SUSPECT,''); if(!suspectName){ GM_setValue(K_ACTION_FLAG,0); return; }
      const select=document.querySelector('select[name="ally_action"]'); if(select){ select.value='rights'; select.dispatchEvent(new Event('change',{bubbles:true})); }
      await sleep(100);
      const links=Array.from(document.querySelectorAll('a[href*="screen=info_player"][href*="id="]'));
      const a=links.find(x=>x.textContent.trim().localeCompare(suspectName,undefined,{sensitivity:'accent'})===0);
      if(!a){ GM_setValue(K_ACTION_FLAG,0); return; }
      const href=new URL(a.getAttribute('href'),location.origin); const id=href.searchParams.get('id'); if(!id){ GM_setValue(K_ACTION_FLAG,0); return; }
      const radio=document.querySelector(`input[type="radio"][name="player"][value="${id}"]`); if(radio && !radio.checked) radio.click();
      const editBtn=Array.from(document.querySelectorAll('a.btn, a.show_toggle.btn')).find(x=>/Editar permiss/i.test(x.textContent)); if(editBtn) editBtn.click();
      await sleep(120);
      const lead=document.querySelector(`input[type="checkbox"][name="player_id[${id}][lead]"]`);
      const found=document.querySelector(`input[type="checkbox"][name="player_id[${id}][found]"]`);
      if(lead && lead.checked) lead.click(); if(found && found.checked) found.click();
      await sleep(80);
      if(lead && lead.checked) lead.click(); if(found && found.checked) found.click();
      const saveBtn=Array.from(document.querySelectorAll('input[type="submit"].btn.show_toggle, input[type="submit"].btn')).find(i=>/Salvar permiss/i.test(i.value));
      if(saveBtn) saveBtn.click();
      GM_setValue(K_ACTION_FLAG,0);
    }
    act();
  }

  // Boot
  mountUiOnOverview();
  runOverview();
  runMembers();
})();
