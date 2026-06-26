import { useState, useEffect } from "react";

const PHONE = "5542988408840";
const SVCS = ["Corte de Grama ✂️","Poda de Árvore 🌳","Irrigação Inteligente 💧","Iluminação de Jardim 💡","Limpeza Geral 🧹","Adubação 🌱","Controle de Pragas 🐛","Paisagismo 🌸"];
const FREQS = ["Semanal","Quinzenal","Mensal"];
const DS = ["Dom","Seg","Ter","Qua","Qui","Sex","Sáb"];
const EJS = { key:"lqI0oJGFLIs8N6eq_", svc:"service_u2qllop", tpl:"template_4fdj8fo" };

const uid = () => Math.random().toString(36).slice(2,9);
const mesKey = () => new Date().toISOString().slice(0,7);
const mesLabel = () => new Date().toLocaleDateString("pt-BR",{month:"long",year:"numeric"}).replace(/^\w/,c=>c.toUpperCase());
const dateKey = d => new Date(d).toISOString().slice(0,10);
const inp = { background:"#f0faf3", border:"1.5px solid #d8f3dc", borderRadius:"9px", padding:".65rem .9rem", fontSize:".88rem", fontFamily:"inherit", color:"#1f2d26", outline:"none", width:"100%", boxSizing:"border-box" };
const lbl = { display:"block", fontSize:".72rem", fontWeight:700, color:"#2d6a4f", marginBottom:".3rem", textTransform:"uppercase", letterSpacing:".05em" };

function getISOWeek(date) {
  const d = new Date(date); d.setHours(0,0,0,0);
  d.setDate(d.getDate() + 4 - (d.getDay()||7));
  return Math.ceil((((d - new Date(d.getFullYear(),0,1))/86400000)+1)/7);
}

function isScheduledOn(svc, date) {
  const ag = svc.agendamento; if(!ag) return false;
  if(svc.freq==="Mensal") return ag.diaMes===date.getDate();
  if(!ag.diasSemana||!ag.diasSemana.includes(date.getDay())) return false;
  if(svc.freq==="Semanal") return true;
  if(svc.freq==="Quinzenal") {
    const w = getISOWeek(date);
    return ag.semPar==="par" ? w%2===0 : w%2===1;
  }
  return false;
}

function getScheduleText(svc) {
  const ag = svc.agendamento; if(!ag) return "";
  if(svc.freq==="Mensal") return `Dia ${ag.diaMes}`;
  if(ag.diasSemana&&ag.diasSemana.length>0) {
    const dias = ag.diasSemana.map(d=>DS[d]).join(", ");
    return svc.freq==="Quinzenal" ? `${dias} · ${ag.semPar==="par"?"sem. pares":"sem. ímpares"}` : dias;
  }
  return "";
}

function getVisits(conds, hoje, days=7) {
  const list = [];
  conds.filter(c=>c.ativo!==false).forEach(cond=>{
    for(let i=1;i<=days;i++){
      const d=new Date(hoje); d.setDate(hoje.getDate()+i); d.setHours(0,0,0,0);
      cond.servicos.forEach(svc=>{
        if(isScheduledOn(svc,d)) list.push({cond,svc,date:new Date(d)});
      });
    }
  });
  return list.sort((a,b)=>a.date-b.date);
}

async function sendEmail(params) {
  return fetch("https://api.emailjs.com/api/v1.0/email/send",{
    method:"POST", headers:{"Content-Type":"application/json"},
    body:JSON.stringify({service_id:EJS.svc,template_id:EJS.tpl,user_id:EJS.key,template_params:params})
  });
}

export default function App() {
  const [tab,setTab] = useState("agenda");
  const [conds,setConds] = useState([]);
  const [feitos,setFeitos] = useState({});
  const [notifs,setNotifs] = useState({});
  const [loading,setLoading] = useState(true);
  const [form,setForm] = useState(null);
  const [toast,setToast] = useState(null);
  const [sending,setSending] = useState(false);
  const hoje = new Date(); hoje.setHours(0,0,0,0);

  useEffect(()=>{
    (async()=>{
      try{const r=await window.storage.get("echo-conds");if(r)setConds(JSON.parse(r.value));}catch{}
      try{const r=await window.storage.get(`echo-feitos:${mesKey()}`);if(r)setFeitos(JSON.parse(r.value));}catch{}
      try{const r=await window.storage.get("echo-notifs");if(r)setNotifs(JSON.parse(r.value));}catch{}
      setLoading(false);
    })();
  },[]);

  useEffect(()=>{
    if(!loading&&conds.length>0) checkNotifs(conds,notifs,false);
  },[loading]);

  async function saveConds(data){ setConds(data); try{await window.storage.set("echo-conds",JSON.stringify(data));}catch{} }
  async function saveFeitos(data){ setFeitos(data); try{await window.storage.set(`echo-feitos:${mesKey()}`,JSON.stringify(data));}catch{} }
  async function saveNotifs(data){ setNotifs(data); try{await window.storage.set("echo-notifs",JSON.stringify(data));}catch{} }

  function getPending(c,n) {
    const pending=[];
    getVisits(c,hoje,3).forEach(v=>{
      const dias=Math.round((v.date-hoje)/86400000);
      if(dias>=1&&dias<=3){
        const nk=`${v.cond.id}|${v.svc.nome}|${dateKey(v.date)}`;
        if(!n[nk]) pending.push({...v,nk,dias});
      }
    });
    return pending;
  }

  async function checkNotifs(c,n,manual=true) {
    const pending=getPending(c,n);
    if(pending.length===0){ if(manual) showToast("✅ Nenhuma notificação pendente",false); return; }
    if(manual) setSending(true);
    let sent=0,failed=0; const newN={...n};
    for(const {cond,svc,date,nk,dias} of pending){
      const ds=date.toLocaleDateString("pt-BR",{weekday:"long",day:"numeric",month:"long"});
      try{
        await sendEmail({
          servico:`⚠️ Visita em ${dias} dia${dias>1?"s":""}`,
          data:ds,
          horario:`${cond.nome} — ${svc.nome}`,
          nome:"Echo Jardinagem",
          telefone:cond.telefone||"—",
          endereco:cond.endereco||"—",
          observacoes:`Frequência: ${svc.freq} | Agendamento: ${getScheduleText(svc)}`
        });
        newN[nk]={at:new Date().toISOString(),ok:true};
        sent++;
      }catch{
        newN[nk]={at:new Date().toISOString(),ok:false};
        failed++;
      }
    }
    saveNotifs(newN);
    if(manual){ setSending(false); showToast(`✅ ${sent} lembrete${sent!==1?"s":""} enviado${sent!==1?"s":""}${failed>0?` · ${failed} falhou`:""}`,failed>0); }
  }

  function showToast(msg,warn=false){ setToast({msg,warn}); setTimeout(()=>setToast(null),4000); }

  function enviarRelatorio(){
    const ativos=conds.filter(c=>c.ativo!==false&&c.servicos.length>0);
    let msg=`🌿 *Echo Jardinagem — ${mesLabel()}*\n\n`;
    ativos.forEach(c=>{
      const done=c.servicos.filter(s=>feitos[`${c.id}|${s.nome}`]).length;
      msg+=`🏢 *${c.nome}* (${done}/${c.servicos.length})\n`;
      c.servicos.forEach(s=>{ msg+=`${feitos[`${c.id}|${s.nome}`]?"✅":"⏳"} ${s.nome} · ${s.freq}\n`; });
      msg+="\n";
    });
    const pend=ativos.flatMap(c=>c.servicos.filter(s=>!feitos[`${c.id}|${s.nome}`]).map(s=>`${c.nome} — ${s.nome}`));
    msg+=pend.length===0?"🎉 *Todos os serviços concluídos!*":`⚠️ *${pend.length} pendente${pend.length>1?"s":""}:*\n${pend.map(p=>`• ${p}`).join("\n")}`;
    window.open(`https://wa.me/${PHONE}?text=${encodeURIComponent(msg)}`,"_blank");
  }

  if(loading) return <div style={{display:"flex",alignItems:"center",justifyContent:"center",height:"100vh",background:"#0d2b1a",color:"#74c69d",fontFamily:"Inter,sans-serif"}}>Carregando...</div>;

  const ativos=conds.filter(c=>c.ativo!==false);
  const totSvcs=ativos.reduce((a,c)=>a+c.servicos.length,0);
  const totFeitos=ativos.reduce((a,c)=>a+c.servicos.filter(s=>feitos[`${c.id}|${s.nome}`]).length,0);
  const pct=totSvcs>0?Math.round(totFeitos/totSvcs*100):0;
  const upcoming=getVisits(conds,hoje,7);
  const pendCount=getPending(conds,notifs).length;

  return (
    <div style={{fontFamily:"Inter,sans-serif",background:"#f0faf3",minHeight:"100vh",color:"#1f2d26"}}>
      {toast&&<div style={{position:"fixed",top:"1rem",right:"1rem",background:toast.warn?"#92400e":"#1a4a2e",color:"#fff",padding:".75rem 1.2rem",borderRadius:"12px",fontSize:".85rem",fontWeight:600,zIndex:2000,boxShadow:"0 4px 16px rgba(0,0,0,.3)",maxWidth:"280px"}}>{toast.msg}</div>}

      {/* Header */}
      <div style={{background:"linear-gradient(135deg,#0d2b1a,#1a4a2e)",padding:"1.2rem 1.5rem 1rem",boxShadow:"0 2px 16px rgba(0,0,0,.3)"}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:".9rem",gap:".7rem"}}>
          <div>
            <div style={{color:"#74c69d",fontSize:".68rem",fontWeight:700,letterSpacing:".09em",textTransform:"uppercase"}}>🌿 Echo Jardinagem</div>
            <div style={{color:"#fff",fontSize:"1.05rem",fontWeight:800,marginTop:".15rem"}}>Gestão de Condomínios</div>
          </div>
          <div style={{display:"flex",gap:".45rem",flexShrink:0}}>
            <button onClick={()=>checkNotifs(conds,notifs,true)} disabled={sending} style={{background:pendCount>0?"#f59e0b":"rgba(255,255,255,.15)",color:"#fff",border:`1px solid ${pendCount>0?"#f59e0b":"rgba(255,255,255,.25)"}`,borderRadius:"100px",padding:".5rem .85rem",fontSize:".74rem",fontWeight:700,cursor:"pointer",position:"relative"}}>
              {sending?"Enviando...":"🔔 "+( pendCount>0?`${pendCount} pendente${pendCount>1?"s":""}`:"Lembretes")}
            </button>
            <button onClick={enviarRelatorio} style={{background:"#25D366",color:"#fff",border:"none",borderRadius:"100px",padding:".5rem .85rem",fontSize:".74rem",fontWeight:700,cursor:"pointer",boxShadow:"0 2px 8px rgba(37,211,102,.3)"}}>📲 Relatório</button>
          </div>
        </div>
        {totSvcs>0&&<div>
          <div style={{display:"flex",justifyContent:"space-between",marginBottom:".35rem"}}>
            <span style={{fontSize:".72rem",color:"#95d5b2"}}>{mesLabel()}</span>
            <span style={{fontSize:".72rem",color:"#74c69d",fontWeight:700}}>{totFeitos}/{totSvcs} · {pct}%</span>
          </div>
          <div style={{background:"rgba(255,255,255,.15)",borderRadius:"100px",height:"6px",overflow:"hidden"}}>
            <div style={{background:"linear-gradient(90deg,#52b788,#74c69d)",height:"100%",width:`${pct}%`,borderRadius:"100px",transition:"width .5s"}}/>
          </div>
        </div>}
      </div>

      {/* Tabs */}
      <div style={{background:"#fff",borderBottom:"2px solid #e2ebe4",display:"flex",padding:"0 1rem",overflowX:"auto"}}>
        {[["agenda","📋 Agenda"],["proximas",`🗓️ Próximas${upcoming.length>0?` (${upcoming.length})`:""}`],["condominios","🏢 Condomínios"]].map(([id,lb])=>(
          <button key={id} onClick={()=>setTab(id)} style={{background:"none",border:"none",borderBottom:tab===id?"2.5px solid #52b788":"2.5px solid transparent",marginBottom:"-2px",padding:".8rem .8rem",fontSize:".82rem",fontWeight:tab===id?700:500,color:tab===id?"#2d6a4f":"#8a9a8f",cursor:"pointer",whiteSpace:"nowrap",flexShrink:0}}>
            {lb}
          </button>
        ))}
      </div>

      <div style={{padding:"1.2rem",maxWidth:"760px",margin:"0 auto"}}>
        {tab==="agenda"&&<Agenda conds={ativos} feitos={feitos} toggle={(cid,svc)=>{const k=`${cid}|${svc}`;saveFeitos({...feitos,[k]:!feitos[k]});}}/>}
        {tab==="proximas"&&<Proximas visits={upcoming} notifs={notifs} hoje={hoje}/>}
        {tab==="condominios"&&<Condominios conds={conds} onEdit={c=>setForm(c)} onDelete={id=>{if(confirm("Remover?"))saveConds(conds.filter(c=>c.id!==id));}} onNew={()=>setForm({nome:"",endereco:"",contato:"",telefone:"",servicos:[],ativo:true})}/>}
      </div>

      {form&&<FormModal cond={form} onSave={c=>{saveConds(c.id?conds.map(x=>x.id===c.id?c:x):[...conds,{...c,id:uid()}]);setForm(null);}} onClose={()=>setForm(null)}/>}
    </div>
  );
}

function Agenda({conds,feitos,toggle}) {
  if(!conds.length) return <Empty icon="🏢" title="Nenhum condomínio cadastrado" sub='Vá em "Condomínios" para adicionar'/>;
  return <div style={{display:"flex",flexDirection:"column",gap:".9rem"}}>
    {conds.map(c=>{
      const done=c.servicos.filter(s=>feitos[`${c.id}|${s.nome}`]).length;
      const all=done===c.servicos.length&&c.servicos.length>0;
      return <div key={c.id} style={{background:"#fff",borderRadius:"14px",border:`1.5px solid ${all?"#74c69d":"#e2ebe4"}`,overflow:"hidden",boxShadow:"0 2px 8px rgba(0,0,0,.06)"}}>
        <div style={{background:all?"linear-gradient(135deg,#2d6a4f,#1a4a2e)":"linear-gradient(135deg,#1a4a2e,#0d2b1a)",padding:".85rem 1.2rem",display:"flex",alignItems:"center",justifyContent:"space-between"}}>
          <div>
            <div style={{color:"#fff",fontWeight:700,fontSize:".9rem"}}>{c.nome}</div>
            {c.contato&&<div style={{color:"#95d5b2",fontSize:".7rem",marginTop:".1rem"}}>👤 {c.contato}{c.telefone?` · ${c.telefone}`:""}</div>}
          </div>
          <div style={{background:all?"#52b788":"rgba(255,255,255,.15)",color:"#fff",borderRadius:"100px",padding:".22rem .7rem",fontSize:".73rem",fontWeight:700,flexShrink:0}}>{done}/{c.servicos.length}{all?" ✅":""}</div>
        </div>
        {!c.servicos.length?<div style={{padding:"1rem 1.2rem",color:"#8a9a8f",fontSize:".83rem"}}>Nenhum serviço</div>:
        <div style={{padding:"0 1.2rem"}}>
          {c.servicos.map((s,i)=>{
            const k=`${c.id}|${s.nome}`;const ck=!!feitos[k];
            return <div key={i} onClick={()=>toggle(c.id,s.nome)} style={{display:"flex",alignItems:"center",gap:".8rem",padding:".78rem 0",borderBottom:i<c.servicos.length-1?"1px solid #f0faf3":"none",cursor:"pointer",userSelect:"none"}}>
              <div style={{width:"22px",height:"22px",borderRadius:"6px",border:`2px solid ${ck?"#52b788":"#d8f3dc"}`,background:ck?"#52b788":"#fff",display:"flex",alignItems:"center",justifyContent:"center",flexShrink:0,transition:"all .2s"}}>
                {ck&&<span style={{color:"#fff",fontSize:".72rem",fontWeight:900}}>✓</span>}
              </div>
              <div style={{flex:1}}>
                <div style={{fontSize:".85rem",fontWeight:600,color:ck?"#8a9a8f":"#1f2d26",textDecoration:ck?"line-through":"none"}}>{s.nome}</div>
                <div style={{fontSize:".71rem",color:"#8a9a8f",marginTop:".08rem"}}>🔄 {s.freq}{s.agendamento&&getScheduleText(s)?" · "+getScheduleText(s):""}</div>
              </div>
            </div>;
          })}
        </div>}
      </div>;
    })}
  </div>;
}

function Proximas({visits,notifs,hoje}) {
  if(!visits.length) return <Empty icon="🗓️" title="Nenhuma visita nos próximos 7 dias" sub="Configure os dias nos serviços de cada condomínio"/>;
  const grouped={};
  visits.forEach(v=>{ const dk=dateKey(v.date); if(!grouped[dk]) grouped[dk]=[]; grouped[dk].push(v); });
  return <div style={{display:"flex",flexDirection:"column",gap:".9rem"}}>
    {Object.entries(grouped).map(([dk,vs])=>{
      const d=new Date(dk+"T12:00:00");
      const dias=Math.round((d-hoje)/86400000);
      const urgent=dias<=2;
      const dlabel=dias===1?"Amanhã ⚡":dias===2?"Em 2 dias ⚠️":d.toLocaleDateString("pt-BR",{weekday:"long",day:"numeric",month:"long"});
      return <div key={dk} style={{background:"#fff",borderRadius:"14px",border:`1.5px solid ${urgent?"#fbbf24":"#e2ebe4"}`,overflow:"hidden",boxShadow:"0 2px 8px rgba(0,0,0,.06)"}}>
        <div style={{background:urgent?"linear-gradient(135deg,#92400e,#78350f)":"linear-gradient(135deg,#1a4a2e,#0d2b1a)",padding:".85rem 1.2rem",display:"flex",alignItems:"center",justifyContent:"space-between"}}>
          <div>
            <div style={{color:"#fff",fontWeight:700,fontSize:".88rem"}}>{dlabel.charAt(0).toUpperCase()+dlabel.slice(1)}</div>
            <div style={{color:urgent?"#fcd34d":"#95d5b2",fontSize:".7rem",marginTop:".1rem"}}>{d.toLocaleDateString("pt-BR",{weekday:"long",day:"numeric",month:"long"})}</div>
          </div>
          <div style={{background:urgent?"#f59e0b":"rgba(255,255,255,.15)",color:"#fff",borderRadius:"100px",padding:".22rem .7rem",fontSize:".72rem",fontWeight:700,flexShrink:0}}>{dias} dia{dias>1?"s":""}</div>
        </div>
        <div style={{padding:"0 1.2rem"}}>
          {vs.map((v,i)=>{
            const nk=`${v.cond.id}|${v.svc.nome}|${dk}`;const n=notifs[nk];
            return <div key={i} style={{display:"flex",alignItems:"center",gap:".8rem",padding:".78rem 0",borderBottom:i<vs.length-1?"1px solid #f0faf3":"none"}}>
              <div style={{flex:1}}>
                <div style={{fontSize:".86rem",fontWeight:700,color:"#1a4a2e"}}>{v.cond.nome}</div>
                <div style={{fontSize:".74rem",color:"#4a5a50",marginTop:".12rem"}}>{v.svc.nome} · {v.svc.freq}</div>
                {v.cond.endereco&&<div style={{fontSize:".7rem",color:"#8a9a8f",marginTop:".08rem"}}>📍 {v.cond.endereco}</div>}
              </div>
              <span style={{fontSize:".72rem",fontWeight:700,padding:".22rem .65rem",borderRadius:"100px",flexShrink:0,...(n?n.ok?{color:"#52b788",background:"#d8f3dc"}:{color:"#ef4444",background:"#fef2f2"}:{color:"#f59e0b",background:"#fef3c7"})}}>
                {n?n.ok?"✉️ Enviado":"❌ Falhou":"⏳ Pendente"}
              </span>
            </div>;
          })}
        </div>
      </div>;
    })}
  </div>;
}

function Condominios({conds,onEdit,onDelete,onNew}) {
  return <div>
    <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:"1.1rem"}}>
      <div style={{fontSize:".83rem",color:"#4a5a50"}}>{conds.length} condomínio{conds.length!==1?"s":""}</div>
      <button onClick={onNew} style={{background:"#1a4a2e",color:"#fff",border:"none",borderRadius:"100px",padding:".55rem 1.1rem",fontSize:".82rem",fontWeight:700,cursor:"pointer"}}>+ Novo</button>
    </div>
    {!conds.length&&<Empty icon="🏢" title="Nenhum condomínio ainda" sub='Clique em "+ Novo" para começar' dashed/>}
    <div style={{display:"flex",flexDirection:"column",gap:".75rem"}}>
      {conds.map(c=><div key={c.id} style={{background:"#fff",borderRadius:"12px",border:"1.5px solid #e2ebe4",padding:"1rem 1.2rem",display:"flex",alignItems:"center",gap:"1rem",boxShadow:"0 1px 6px rgba(0,0,0,.04)"}}>
        <div style={{flex:1,minWidth:0}}>
          <div style={{fontWeight:700,fontSize:".9rem",color:"#1a4a2e"}}>{c.nome}</div>
          {c.endereco&&<div style={{fontSize:".73rem",color:"#8a9a8f",marginTop:".1rem"}}>📍 {c.endereco}</div>}
          <div style={{fontSize:".73rem",marginTop:".28rem",display:"flex",alignItems:"center",gap:".45rem"}}>
            <span style={{color:"#52b788",fontWeight:600}}>{c.servicos.length} serviço{c.servicos.length!==1?"s":""}</span>
            <span style={{width:"3px",height:"3px",borderRadius:"50%",background:"#d8f3dc",display:"inline-block"}}/>
            <span style={{color:c.ativo!==false?"#2d6a4f":"#8a9a8f",fontWeight:600}}>{c.ativo!==false?"Ativo":"Inativo"}</span>
          </div>
        </div>
        <div style={{display:"flex",gap:".4rem",flexShrink:0}}>
          <button onClick={()=>onEdit(c)} style={{background:"#f0faf3",border:"1.5px solid #b7e4c7",color:"#2d6a4f",borderRadius:"8px",padding:".38rem .7rem",fontSize:".78rem",fontWeight:600,cursor:"pointer"}}>✏️</button>
          <button onClick={()=>onDelete(c.id)} style={{background:"#fff5f5",border:"1.5px solid #fecaca",color:"#ef4444",borderRadius:"8px",padding:".38rem .7rem",cursor:"pointer"}}>🗑️</button>
        </div>
      </div>)}
    </div>
  </div>;
}

function FormModal({cond,onSave,onClose}) {
  const [f,setF]=useState({...cond});
  const [ns,setNs]=useState({nome:SVCS[0],freq:"Semanal",agendamento:{diasSemana:[],semPar:"impar",diaMes:1}});

  function toggleDia(d){ const dias=ns.agendamento.diasSemana||[];setNs(p=>({...p,agendamento:{...p.agendamento,diasSemana:dias.includes(d)?dias.filter(x=>x!==d):[...dias,d].sort((a,b)=>a-b)}})); }
  function addSvc(){ if(f.servicos.find(s=>s.nome===ns.nome))return;setF(p=>({...p,servicos:[...p.servicos,{nome:ns.nome,freq:ns.freq,agendamento:{...ns.agendamento}}]})); }
  function remSvc(nome){ setF(p=>({...p,servicos:p.servicos.filter(s=>s.nome!==nome)})); }
  function salvar(){ if(!f.nome.trim()){alert("Digite o nome");return;} onSave(f); }

  return <div onClick={e=>{if(e.target===e.currentTarget)onClose();}} style={{position:"fixed",inset:0,background:"rgba(0,0,0,.55)",zIndex:999,display:"flex",alignItems:"flex-end",justifyContent:"center"}}>
    <div style={{background:"#fff",borderRadius:"20px 20px 0 0",padding:"1.5rem",width:"100%",maxWidth:"520px",maxHeight:"93vh",overflowY:"auto"}}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:"1.2rem"}}>
        <h3 style={{fontSize:"1rem",fontWeight:800,color:"#1a4a2e"}}>{cond.id?"✏️ Editar":"🏢 Novo"} Condomínio</h3>
        <button onClick={onClose} style={{background:"none",border:"none",fontSize:"1.4rem",cursor:"pointer",color:"#aaa",lineHeight:1}}>✕</button>
      </div>
      <div style={{display:"flex",flexDirection:"column",gap:".85rem"}}>
        <div><label style={lbl}>Nome *</label><input style={inp} value={f.nome} onChange={e=>setF(p=>({...p,nome:e.target.value}))} placeholder="Condomínio das Flores"/></div>
        <div><label style={lbl}>Endereço</label><input style={inp} value={f.endereco||""} onChange={e=>setF(p=>({...p,endereco:e.target.value}))} placeholder="Rua, número, bairro — Guarapuava"/></div>
        <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:".7rem"}}>
          <div><label style={lbl}>Contato</label><input style={inp} value={f.contato||""} onChange={e=>setF(p=>({...p,contato:e.target.value}))} placeholder="Síndico"/></div>
          <div><label style={lbl}>Telefone</label><input style={inp} value={f.telefone||""} onChange={e=>setF(p=>({...p,telefone:e.target.value}))} placeholder="(42) 9..."/></div>
        </div>

        {/* Services */}
        <div style={{borderTop:"1px solid #e2ebe4",paddingTop:".85rem"}}>
          <label style={lbl}>Serviços Contratados</label>
          {!f.servicos.length&&<div style={{fontSize:".8rem",color:"#8a9a8f",marginBottom:".6rem"}}>Nenhum serviço adicionado</div>}
          {f.servicos.map((s,i)=><div key={i} style={{display:"flex",alignItems:"flex-start",gap:".6rem",marginBottom:".4rem",background:"#f0faf3",borderRadius:"9px",padding:".6rem .9rem",border:"1px solid #d8f3dc"}}>
            <div style={{flex:1}}>
              <div style={{fontSize:".84rem",fontWeight:700,color:"#1a4a2e"}}>{s.nome}</div>
              <div style={{fontSize:".73rem",color:"#52b788",marginTop:".1rem"}}>{s.freq}{s.agendamento&&getScheduleText(s)?" · "+getScheduleText(s):""}</div>
            </div>
            <button onClick={()=>remSvc(s.nome)} style={{background:"none",border:"none",color:"#ef4444",cursor:"pointer",fontSize:"1rem",lineHeight:1,flexShrink:0}}>✕</button>
          </div>)}

          {/* Add service */}
          <div style={{background:"#f9fffe",border:"1.5px solid #d8f3dc",borderRadius:"12px",padding:".9rem",marginTop:".5rem"}}>
            <div style={{fontSize:".72rem",fontWeight:700,color:"#2d6a4f",marginBottom:".6rem",textTransform:"uppercase",letterSpacing:".04em"}}>Adicionar serviço</div>
            <div style={{display:"flex",gap:".4rem",marginBottom:".6rem"}}>
              <select value={ns.nome} onChange={e=>setNs(p=>({...p,nome:e.target.value}))} style={{...inp,flex:2}}>
                {SVCS.map(s=><option key={s} value={s}>{s}</option>)}
              </select>
              <select value={ns.freq} onChange={e=>setNs(p=>({...p,freq:e.target.value,agendamento:{diasSemana:[],semPar:"impar",diaMes:1}}))} style={{...inp,flex:1}}>
                {FREQS.map(f=><option key={f}>{f}</option>)}
              </select>
            </div>

            {(ns.freq==="Semanal"||ns.freq==="Quinzenal")&&<div style={{marginBottom:".6rem"}}>
              <div style={{fontSize:".72rem",color:"#4a5a50",marginBottom:".4rem",fontWeight:600}}>Dias da semana:</div>
              <div style={{display:"flex",gap:".28rem",flexWrap:"wrap",marginBottom:ns.freq==="Quinzenal"?".5rem":"0"}}>
                {DS.map((d,i)=>{const sel=ns.agendamento.diasSemana?.includes(i);return <button key={i} onClick={()=>toggleDia(i)} type="button" style={{padding:".32rem .45rem",borderRadius:"7px",border:`1.5px solid ${sel?"#52b788":"#e2ebe4"}`,background:sel?"#52b788":"#fff",color:sel?"#fff":"#4a5a50",fontSize:".73rem",fontWeight:sel?700:500,cursor:"pointer",minWidth:"36px"}}>{d}</button>;})}
              </div>
              {ns.freq==="Quinzenal"&&<div style={{display:"flex",gap:".4rem"}}>
                {[["impar","Semanas ímpares (1ª, 3ª...)"],["par","Semanas pares (2ª, 4ª...)"]].map(([v,lb])=><button key={v} type="button" onClick={()=>setNs(p=>({...p,agendamento:{...p.agendamento,semPar:v}}))} style={{flex:1,padding:".38rem",borderRadius:"7px",border:`1.5px solid ${ns.agendamento.semPar===v?"#52b788":"#e2ebe4"}`,background:ns.agendamento.semPar===v?"#52b788":"#fff",color:ns.agendamento.semPar===v?"#fff":"#4a5a50",fontSize:".72rem",fontWeight:600,cursor:"pointer"}}>{lb}</button>)}
              </div>}
            </div>}

            {ns.freq==="Mensal"&&<div style={{marginBottom:".6rem"}}>
              <div style={{fontSize:".72rem",color:"#4a5a50",marginBottom:".4rem",fontWeight:600}}>Dia do mês:</div>
              <select value={ns.agendamento.diaMes} onChange={e=>setNs(p=>({...p,agendamento:{...p.agendamento,diaMes:parseInt(e.target.value)}}))} style={{...inp,width:"auto"}}>
                {Array.from({length:28},(_,i)=><option key={i+1} value={i+1}>Dia {i+1}</option>)}
              </select>
            </div>}

            <button onClick={addSvc} style={{width:"100%",background:"#1a4a2e",color:"#fff",border:"none",borderRadius:"9px",padding:".65rem",cursor:"pointer",fontWeight:700,fontSize:".85rem"}}>+ Adicionar serviço</button>
          </div>
        </div>

        <label style={{display:"flex",alignItems:"center",gap:".6rem",cursor:"pointer"}}>
          <input type="checkbox" checked={f.ativo!==false} onChange={e=>setF(p=>({...p,ativo:e.target.checked}))} style={{width:"16px",height:"16px",cursor:"pointer",accentColor:"#52b788"}}/>
          <span style={{fontSize:".85rem",fontWeight:600,color:"#4a5a50"}}>Condomínio ativo</span>
        </label>
        <div style={{display:"flex",gap:".7rem",paddingTop:".3rem"}}>
          <button onClick={onClose} style={{flex:1,background:"none",border:"1.5px solid #e2ebe4",color:"#4a5a50",borderRadius:"100px",padding:".75rem",fontSize:".88rem",fontWeight:600,cursor:"pointer"}}>Cancelar</button>
          <button onClick={salvar} style={{flex:2,background:"#1a4a2e",color:"#fff",border:"none",borderRadius:"100px",padding:".75rem",fontSize:".88rem",fontWeight:700,cursor:"pointer"}}>💾 Salvar</button>
        </div>
      </div>
    </div>
  </div>;
}

function Empty({icon,title,sub,dashed}) {
  return <div style={{textAlign:"center",padding:"3.5rem 1rem",color:"#8a9a8f",background:"#fff",borderRadius:"14px",border:`1.5px ${dashed?"dashed":"solid"} #e2ebe4`}}>
    <div style={{fontSize:"3rem",marginBottom:"1rem"}}>{icon}</div>
    <div style={{fontWeight:700,color:"#4a5a50",marginBottom:".4rem"}}>{title}</div>
    <div style={{fontSize:".83rem"}}>{sub}</div>
  </div>;
}
