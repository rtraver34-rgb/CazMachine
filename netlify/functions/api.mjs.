import crypto from 'node:crypto';
import {getStore} from '@netlify/blobs';

const PACKS={
  p1:{stars:25,coins:2500,name:'Pochette de jetons'},
  p2:{stars:100,coins:12000,name:'Sac de jetons'},
  p3:{stars:500,coins:70000,name:'Coffre de jetons'}
};
const json=(o,s=200)=>new Response(JSON.stringify(o),{status:s,headers:{'Content-Type':'application/json'}});
const tg=(m,b)=>fetch('https://api.telegram.org/bot'+process.env.BOT_TOKEN+'/'+m,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(b)}).then(r=>r.json());
const clean=t=>String(t||'').replace(/[<>"'&`]/g,'').trim();

function auth(initData){
  const p=new URLSearchParams(initData||''),hash=p.get('hash');
  if(!hash||!process.env.BOT_TOKEN)return null;
  p.delete('hash');
  const str=[...p.entries()].sort((a,b)=>a[0]<b[0]?-1:1).map(([k,v])=>k+'='+v).join('\n');
  const key=crypto.createHmac('sha256','WebAppData').update(process.env.BOT_TOKEN).digest();
  const h=crypto.createHmac('sha256',key).update(str).digest('hex');
  if(h.length!==hash.length||!crypto.timingSafeEqual(Buffer.from(h),Buffer.from(hash)))return null;
  if(Date.now()/1000-Number(p.get('auth_date')||0)>86400)return null;
  try{return JSON.parse(p.get('user')).id}catch(e){return null}
}

export default async (req)=>{
  const path=new URL(req.url).pathname,store=getStore('cazmachine');
  const b=await req.json().catch(()=>({}));

  if(path==='/api/webhook'){
    if(req.headers.get('x-telegram-bot-api-secret-token')!==process.env.WEBHOOK_SECRET)return json({ok:false},401);
    if(b.pre_checkout_query){
      const q=b.pre_checkout_query;let ok=false;
      try{const d=JSON.parse(q.invoice_payload);ok=!!PACKS[d.p]&&d.u===q.from.id&&q.currency==='XTR'}catch(e){}
      await tg('answerPreCheckoutQuery',{pre_checkout_query_id:q.id,ok,...(ok?{}:{error_message:'Achat invalide'})});
      return json({ok:true});
    }
    const m=b.message;
    if(m&&m.successful_payment){
      const sp=m.successful_payment,cid='charge-'+sp.telegram_payment_charge_id;
      if(!(await store.get(cid))){
        try{
          const d=JSON.parse(sp.invoice_payload),P=PACKS[d.p];
          if(P&&d.u===m.from.id){
            const k='credits-'+d.u;
            await store.set(k,String((Number(await store.get(k))||0)+P.coins));
            await store.set(cid,'1');
          }
        }catch(e){}
      }
      return json({ok:true});
    }
    if(m&&m.text&&m.text.startsWith('/paysupport'))
      await tg('sendMessage',{chat_id:m.chat.id,text:'Un souci avec un achat ? Décris ton problème ici et nous te répondrons rapidement.'});
    return json({ok:true});
  }

  const uid=auth(b.initData);
  if(!uid)return json({error:'bad'},400);

  if(path==='/api/invoice'){
    const P=PACKS[b.pack];if(!P)return json({error:'bad'},400);
    const r=await tg('createInvoiceLink',{title:P.name,description:P.coins.toLocaleString('fr-FR')+' jetons CazMachine',payload:JSON.stringify({u:uid,p:b.pack}),provider_token:'',currency:'XTR',prices:[{label:P.name,amount:P.stars}]});
    return r.ok?json({link:r.result}):json({error:'telegram'},502);
  }

  if(path==='/api/claim'){
    const k='credits-'+uid,n=Number(await store.get(k))||0;
    if(n>0)await store.set(k,'0');
    return json({coins:n});
  }

  if(path==='/api/score'){
    const c=Math.floor(Number(b.coins));
    if(!Number.isFinite(c)||c<0||c>50000000)return json({error:'bad'},400);
    const k='lb-'+uid,prev=await store.get(k,{type:'json'}).catch(()=>null),now=Date.now();
    if(!prev&&c>500000)return json({error:'implausible'},400);
    if(prev&&c>prev.c+150000+Math.max(1,(now-prev.t)/60000)*50000)return json({error:'implausible'},400);
    await store.setJSON(k,{n:clean(b.name).slice(0,16)||'Joueur',a:Array.from(clean(b.av)).slice(0,2).join(''),c,t:now});
    return json({ok:true});
  }

  if(path==='/api/leaderboard'){
    let cache=await store.get('top-cache',{type:'json'}).catch(()=>null);
    if(!cache||Date.now()-cache.t>30000){
      const {blobs}=await store.list({prefix:'lb-'});
      const keys=blobs.map(x=>x.key).slice(0,3000),rows=[];
      for(let i=0;i<keys.length;i+=50){
        const part=await Promise.all(keys.slice(i,i+50).map(async k=>{const v=await store.get(k,{type:'json'}).catch(()=>null);return v?{id:k.slice(3),...v}:null}));
        rows.push(...part.filter(Boolean));
      }
      rows.sort((x,y)=>y.c-x.c);
      const ranks={};rows.forEach((r,i)=>ranks[r.id]=i+1);
      cache={t:Date.now(),total:rows.length,ranks,top:rows.slice(0,50).map(r=>({id:r.id,n:r.n,a:r.a,c:r.c}))};
      await store.setJSON('top-cache',cache);
    }
    const id=String(uid);
    return json({total:cache.total,me:cache.ranks[id]||null,top:cache.top.map(({id:x,...r})=>({...r,me:x===id}))});
  }

  return json({error:'not found'},404);
};

export const config={path:['/api/invoice','/api/webhook','/api/claim','/api/score','/api/leaderboard']};
