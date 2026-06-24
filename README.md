<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover"/>
  <meta name="apple-mobile-web-app-capable" content="yes"/>
  <meta name="apple-mobile-web-app-status-bar-style" content="default"/>
  <meta name="apple-mobile-web-app-title" content="Stock Tracker"/>
  <meta name="theme-color" content="#ffffff"/>
  <title>Stock Tracker</title>

  <!-- PWA icons (inline SVG as data URIs) -->
  <link rel="apple-touch-icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><rect width='100' height='100' rx='22' fill='%231a56db'/><text y='68' x='50' text-anchor='middle' font-size='58'>📈</text></svg>"/>

  <style>
    *{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent;}
    body{font-family:-apple-system,BlinkMacSystemFont,'SF Pro Text','Segoe UI',sans-serif;background:#f5f6fa;overscroll-behavior:none;}
    input,button,select{font-family:inherit;}
    button{cursor:pointer;}
    /* Safe area support for iPhone notch/home bar */
    .safe-top{padding-top:env(safe-area-inset-top,0px);}
    .safe-bot{padding-bottom:env(safe-area-inset-bottom,16px);}
  </style>

  <!-- React + Babel from CDN -->
  <script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
  <script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
</head>
<body>
<div id="root"></div>

<script type="text/babel">
const { useState, useEffect, useRef, useCallback } = React;

const C = {
  bg:"#f5f6fa", panel:"#ffffff", border:"#e8eaf0",
  green:"#00a86b", red:"#e8334a", amber:"#f5a623",
  dark:"#0d1b2a", muted:"#8a94a6", accent:"#1a56db",
};

const TICKERS = [
  { id:"DHHF", name:"BetaShares Diversified All Growth ETF", exchange:"ASX", currency:"AUD", flag:"🇦🇺", basePrice:38.20, icon:"🌏",
    bull:["Low-cost diversification (MER 0.19%)","100% growth assets — global + Australian equities","Long-run equity premium over 7+ year horizon"],
    bear:["Full equity exposure — no defensive buffer","Global recession would hit all asset classes"],
    dca:"Best suited to monthly DCA given low MER. Consistent contributions over your 7+ year horizon will smooth entry costs.",
    overview:"DHHF is a single-ETF solution providing diversified exposure to global and Australian shares. It holds ~100% in growth assets with zero fixed income, making it ideal for long-term investors comfortable with volatility in exchange for maximum growth potential." },
  { id:"FANG", name:"Global X FANG+ ETF", exchange:"ASX", currency:"AUD", flag:"🇦🇺", basePrice:46.85, icon:"💻",
    bull:["Direct exposure to AI/cloud mega-caps","Strong earnings momentum from top 10 tech stocks","AUD-denominated — no FX account needed"],
    bear:["Highly concentrated — only 10 holdings","Valuation sensitive to US interest rate changes"],
    dca:"Consider staged entries due to higher volatility. Avoid lump-sum purchases near all-time highs.",
    overview:"FANG tracks the NYSE FANG+ Index, giving concentrated exposure to 10 leading global technology companies including Meta, Apple, Amazon, Netflix, Google, NVIDIA, and others. Higher risk/reward than broad ETFs." },
  { id:"PMGOLD", name:"Perth Mint Physical Gold ETF", exchange:"ASX", currency:"AUD", flag:"🇦🇺", basePrice:32.10, icon:"🥇",
    bull:["Backed by physical gold at the Perth Mint","Hedges against AUD weakness and inflation","Safe-haven demand rises in market uncertainty"],
    bear:["No income — zero dividends","Opportunity cost vs equities in bull markets"],
    dca:"Best held as a 5–10% portfolio hedge rather than a core growth position. Add on equity market strength.",
    overview:"PMGOLD provides direct exposure to physical gold bullion held and guaranteed by the Government of Western Australia via the Perth Mint. It trades in AUD on the ASX." },
  { id:"GOLD", name:"Global X Physical Gold ETF", exchange:"ASX", currency:"AUD", flag:"🇦🇺", basePrice:57.04, icon:"🏅",
    bull:["Backed by physical gold held by JP Morgan in London","Each unit = 1/100th troy oz of allocated gold","AUD gold hit all-time highs in 2026"],
    bear:["No income or dividends paid","USD gold strength needed to offset AUD appreciation"],
    dca:"Hold as a 5–15% portfolio hedge. Consider adding during equity market rallies.",
    overview:"ASX:GOLD is the Global X Physical Gold ETF. Each unit represents approximately 1/100th of a troy ounce of gold held in allocated vaults by JP Morgan Chase in London. It tracks the LBMA Gold Price PM in AUD." },
  { id:"NVDA", name:"NVIDIA Corporation", exchange:"NASDAQ", currency:"AUD", flag:"🇺🇸", basePrice:315.43, icon:"🟢",
    bull:["Dominant AI GPU infrastructure provider","CUDA ecosystem creates deep competitive moat","Data centre revenue in multi-year supercycle"],
    bear:["AUD strengthening reduces AUD returns even if USD price rises","Premium valuation leaves little margin for error"],
    dca:"High single-stock volatility — consider staged entries on pullbacks of 5–10% from recent highs.",
    overview:"NVIDIA designs GPUs for gaming, data centres, and AI workloads. Its H100/H200 GPUs are the primary infrastructure behind large language model training. Priced in AUD via live FX conversion — AUD/USD rate updated automatically." },
];

// ── Price simulation + live AUD/USD for NVDA ─────────────────────
const NVDA_USD_BASE = 200.91; // USD base price
function useLivePrices() {
  const [audUsd, setAudUsd] = useState(0.637); // fallback AUD/USD

  // Fetch live AUD/USD rate from Frankfurter (free, no key needed)
  useEffect(()=>{
    fetch("https://api.frankfurter.app/latest?from=USD&to=AUD")
      .then(r=>r.json())
      .then(d=>{ if(d.rates?.AUD) setAudUsd(1/d.rates.AUD); })
      .catch(()=>{}); // keep fallback on failure
  },[]);

  const makeBase = (basePrice) => {
    const mk=(len,amp)=>Array.from({length:len},(_,i)=>+(basePrice*(1+(Math.sin(i/8)+Math.random()-0.5)*amp)).toFixed(3));
    return {current:basePrice,prev:basePrice,open:basePrice,
      high:basePrice*1.012,low:basePrice*0.988,
      volume:Math.floor(Math.random()*800000+200000),
      change:0,changePct:0,
      history:mk(60,.015),history5d:mk(40,.018),
      history1m:mk(30,.025),history6m:mk(26,.04),history1y:mk(52,.06)};
  };

  const [prices,setPrices]=useState(()=>Object.fromEntries(TICKERS.map(t=>[t.id,makeBase(t.basePrice)])));

  // When AUD/USD loads, re-base NVDA price in AUD
  useEffect(()=>{
    if(!audUsd) return;
    const nvdaAUD = +(NVDA_USD_BASE / audUsd).toFixed(3);
    setPrices(prev=>({...prev, NVDA:{...makeBase(nvdaAUD), audUsd}}));
  },[audUsd]);

  useEffect(()=>{
    const id=setInterval(()=>{
      setPrices(prev=>{
        const next={...prev};
        TICKERS.forEach(t=>{
          const old=prev[t.id];
          if(!old) return;
          const np=+(old.current*(1+(Math.random()-0.497)*0.004)).toFixed(3);
          const ch=+(np-old.open).toFixed(3);
          next[t.id]={...old,prev:old.current,current:np,
            high:Math.max(old.high,np),low:Math.min(old.low,np),
            change:ch,changePct:+((ch/old.open)*100).toFixed(2),
            history:[...old.history.slice(1),np],
            history5d:[...old.history5d.slice(1),np]};
        });
        return next;
      });
    },2000);
    return()=>clearInterval(id);
  },[]);

  // Expose audUsd on NVDA price object so UI can show it
  return prices;
}

// ── Analysis engine ───────────────────────────────────────────────
function generateAnalysis(t,p){
  const avg10=p.history.slice(-10).reduce((a,b)=>a+b,0)/10;
  const avg30=p.history.reduce((a,b)=>a+b,0)/p.history.length;
  const momentum=p.current>avg10&&avg10>avg30?"bullish":p.current<avg10&&avg10<avg30?"bearish":"neutral";
  const recent=p.history.slice(-20);
  const mean=recent.reduce((a,b)=>a+b,0)/recent.length;
  const std=Math.sqrt(recent.map(v=>(v-mean)**2).reduce((a,b)=>a+b,0)/recent.length);
  const vol=(std/mean)*100;
  const dayPos=p.high-p.low>0?(p.current-p.low)/(p.high-p.low):0.5;
  let signal,conf;
  if(momentum==="bullish"&&dayPos>0.5){signal="BUY";conf=vol<1.5?"High":"Medium";}
  else if(momentum==="bearish"&&dayPos<0.4){signal="SELL";conf="Medium";}
  else{signal="HOLD";conf="Medium";}
  const isTech=["NVDA","FANG"].includes(t.id);
  const bm=isTech?1.18:t.id==="PMGOLD"?1.10:1.12;
  const dm=isTech?0.82:t.id==="PMGOLD"?0.91:0.88;
  return{signal,conf,
    bull:+(p.current*bm).toFixed(2),base:+(p.current*((bm+dm)/2)).toFixed(2),bear:+(p.current*dm).toFixed(2),
    momentum,vol:vol.toFixed(2),
    rsi:Math.round(40+Math.random()*30),
    macd:momentum==="bullish"?"Bullish crossover":momentum==="bearish"?"Bearish crossover":"Neutral",
    support:+(p.low*0.98).toFixed(2),resistance:+(p.high*1.02).toFixed(2)};
}

// ── SVG Chart ─────────────────────────────────────────────────────
function LineChart({data,color,height=140}){
  const W=340,H=height,min=Math.min(...data),max=Math.max(...data),range=max-min||1;
  const pts=data.map((v,i)=>`${(i/(data.length-1))*W},${H-4-((v-min)/range)*(H-8)}`).join(" ");
  const gid="g"+color.replace("#","");
  return(
    <svg width="100%" viewBox={`0 0 ${W} ${H}`} preserveAspectRatio="none" style={{display:"block"}}>
      <defs><linearGradient id={gid} x1="0" y1="0" x2="0" y2="1">
        <stop offset="0%" stopColor={color} stopOpacity="0.18"/>
        <stop offset="100%" stopColor={color} stopOpacity="0"/>
      </linearGradient></defs>
      <polygon points={`0,${H-4-((data[0]-min)/range)*(H-8)} ${pts} ${W},${H} 0,${H}`} fill={`url(#${gid})`}/>
      <polyline points={pts} fill="none" stroke={color} strokeWidth="2" strokeLinejoin="round" strokeLinecap="round"/>
    </svg>
  );
}

// ── Watchlist ─────────────────────────────────────────────────────
function WatchlistScreen({prices,onSelect}){
  return(
    <div style={{background:C.bg,minHeight:"100%",paddingBottom:20}}>
      <div style={{background:C.panel,padding:"16px 20px 12px",borderBottom:`1px solid ${C.border}`}} className="safe-top">
        <div style={{fontSize:20,fontWeight:700,color:C.dark}}>Watchlist</div>
        <div style={{fontSize:12,color:C.muted,marginTop:2}}>ASX + NASDAQ · Simulated prices</div>
      </div>
      <div style={{padding:"12px 16px",display:"flex",flexDirection:"column",gap:6}}>
        {TICKERS.map(t=>{
          const p=prices[t.id];if(!p)return null;
          const up=p.changePct>=0,sym=t.currency==="AUD"?"A$":"$";
          return(
            <div key={t.id} onClick={()=>onSelect(t.id)}
              style={{background:C.panel,borderRadius:14,padding:"14px 16px",display:"flex",alignItems:"center",gap:12,boxShadow:"0 1px 4px rgba(0,0,0,0.06)"}}>
              <div style={{width:46,height:46,borderRadius:23,background:up?`${C.green}15`:`${C.red}15`,display:"flex",alignItems:"center",justifyContent:"center",fontSize:20,flexShrink:0}}>
                {t.icon}
              </div>
              <div style={{flex:1,minWidth:0}}>
                <div style={{fontWeight:700,fontSize:15,color:C.dark}}>{t.id} <span style={{fontSize:12}}>{t.flag}</span></div>
                <div style={{fontSize:11,color:C.muted,marginTop:1,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"}}>{t.name}</div>
              </div>
              <div style={{width:60,height:28,flexShrink:0}}>
                <LineChart data={p.history.slice(-20)} color={up?C.green:C.red} height={28}/>
              </div>
              <div style={{textAlign:"right",flexShrink:0}}>
                <div style={{fontWeight:700,fontSize:15,color:C.dark,fontFamily:"monospace"}}>{sym}{p.current.toFixed(2)}</div>
                <div style={{fontSize:12,marginTop:2,fontWeight:600,color:up?C.green:C.red}}>{up?"▲":"▼"}{Math.abs(p.changePct)}%</div>
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}

// ── Stock Detail ──────────────────────────────────────────────────
function StockDetail({tickerId,prices,onBack}){
  const [innerTab,setInnerTab]=useState("overview");
  const [chartRange,setChartRange]=useState("1D");
  const [analysis,setAnalysis]=useState(null);
  const t=TICKERS.find(x=>x.id===tickerId);
  const p=prices[tickerId];
  if(!t||!p)return null;
  const up=p.changePct>=0,sym=t.currency==="AUD"?"A$":"$";
  const sigColor=!analysis?C.muted:analysis.signal==="BUY"?C.green:analysis.signal==="SELL"?C.red:C.amber;
  const chartData={
    "1D":p.history,"5D":p.history5d,"1M":p.history1m,"6M":p.history6m,"1Y":p.history1y
  }[chartRange]||p.history;

  return(
    <div style={{background:C.bg,minHeight:"100%",paddingBottom:32,overflowY:"auto"}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,position:"sticky",top:0,zIndex:10}} className="safe-top">
        <div style={{display:"flex",alignItems:"center",justifyContent:"space-between",padding:"12px 20px 0"}}>
          <button onClick={onBack} style={{background:"transparent",border:"none",color:C.accent,fontSize:16,fontWeight:600,padding:"4px 0"}}>‹ Back</button>
          <span style={{fontWeight:600,fontSize:15,color:C.dark}}>Summary</span>
          <div style={{width:48}}/>
        </div>
        <div style={{padding:"12px 20px 0",display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div>
            <div style={{fontSize:22,fontWeight:800,color:C.dark}}>{t.id} <span style={{fontSize:14}}>{t.flag}</span></div>
            <div style={{fontSize:11,color:C.muted,marginTop:1}}>{t.name.toUpperCase()}</div>
          </div>
          <div style={{width:46,height:46,borderRadius:23,background:up?`${C.green}15`:`${C.red}15`,display:"flex",alignItems:"center",justifyContent:"center",fontSize:22}}>
            {t.icon}
          </div>
        </div>
        <div style={{padding:"10px 20px 12px"}}>
          <div style={{display:"flex",alignItems:"baseline",gap:8}}>
            <span style={{fontSize:34,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>{p.current.toFixed(3)}</span>
            <span style={{fontSize:13,color:C.muted}}>{t.currency}</span>
          </div>
          <div style={{fontSize:14,fontWeight:600,color:up?C.green:C.red,marginTop:4}}>
            {up?"▲":"▼"} {Math.abs(p.change).toFixed(3)} ({Math.abs(p.changePct)}%) Today
          </div>
        </div>
        <div style={{padding:"0 20px 12px",display:"grid",gridTemplateColumns:"1fr 1fr",gap:10}}>
          <button style={{background:C.green,border:"none",color:"#fff",padding:"13px",borderRadius:10,fontSize:15,fontWeight:700}}>Buy</button>
          <button style={{background:"transparent",border:`2px solid ${C.red}`,color:C.red,padding:"13px",borderRadius:10,fontSize:15,fontWeight:700}}>Sell</button>
        </div>
        <div style={{display:"flex",borderTop:`1px solid ${C.border}`,paddingLeft:20}}>
          {["overview","analysis"].map(tab=>(
            <button key={tab} onClick={()=>setInnerTab(tab)} style={{background:"transparent",border:"none",borderBottom:innerTab===tab?`2px solid ${C.accent}`:"2px solid transparent",color:innerTab===tab?C.accent:C.muted,padding:"10px 18px",fontWeight:innerTab===tab?600:400,fontSize:13,textTransform:"capitalize"}}>
              {tab}
            </button>
          ))}
        </div>
      </div>

      {innerTab==="overview"&&(
        <div>
          <div style={{background:C.panel,padding:"16px",marginBottom:8,marginTop:8}}>
            <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:12}}>Price History</div>
            <LineChart data={chartData} color={up?C.green:C.red} height={140}/>
            <div style={{display:"flex",justifyContent:"space-around",marginTop:12,paddingTop:10,borderTop:`1px solid ${C.border}`}}>
              {["1D","5D","1M","6M","1Y"].map(r=>(
                <button key={r} onClick={()=>setChartRange(r)} style={{background:chartRange===r?C.dark:"transparent",border:"none",color:chartRange===r?"#fff":C.muted,padding:"5px 12px",borderRadius:20,fontSize:12,fontWeight:chartRange===r?700:400}}>
                  {r}
                </button>
              ))}
            </div>
          </div>
          <div style={{background:C.panel,padding:"16px",marginBottom:8}}>
            <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:12}}>Key Statistics</div>
            {t.id==="NVDA"&&p.audUsd&&<div style={{background:`${C.accent}08`,border:`1px solid ${C.accent}20`,borderRadius:8,padding:"8px 12px",marginBottom:12,fontSize:12,color:C.accent}}>
              💱 AUD/USD rate: <strong>{(1/p.audUsd).toFixed(4)}</strong> · USD price: <strong>US${(p.current*p.audUsd).toFixed(2)}</strong>
            </div>}
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:0}}>
              {[["Open",`${sym}${p.open.toFixed(3)}`],["Prev Close",`${sym}${(p.current-p.change).toFixed(3)}`],
                ["Day High",`${sym}${p.high.toFixed(3)}`],["Day Low",`${sym}${p.low.toFixed(3)}`],
                ["Volume",`${(p.volume/1000).toFixed(0)}K`],["Exchange",t.exchange]].map(([l,v],i)=>(
                <div key={i} style={{padding:"10px 0",borderBottom:`1px solid ${C.border}`,paddingRight:12}}>
                  <div style={{fontSize:11,color:C.muted,marginBottom:3}}>{l}</div>
                  <div style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>{v}</div>
                </div>
              ))}
            </div>
          </div>
          <div style={{background:C.panel,padding:"16px"}}>
            <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:10}}>About</div>
            <div style={{fontSize:13,color:"#4a5568",lineHeight:1.7}}>{t.overview}</div>
          </div>
        </div>
      )}

      {innerTab==="analysis"&&(
        <div style={{padding:"16px"}}>
          {!analysis&&(
            <div style={{background:C.panel,borderRadius:14,padding:"40px 20px",textAlign:"center",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
              <div style={{fontSize:40,marginBottom:12}}>✦</div>
              <div style={{fontSize:16,fontWeight:600,color:C.dark,marginBottom:8}}>AI Stock Analysis</div>
              <div style={{fontSize:13,color:C.muted,marginBottom:24,lineHeight:1.6}}>Signal, price targets, and recommendation for {t.id}</div>
              <button onClick={()=>setAnalysis(generateAnalysis(t,p))} style={{background:C.accent,border:"none",color:"#fff",padding:"14px 36px",borderRadius:24,fontSize:14,fontWeight:700}}>
                ✦ Run Analysis
              </button>
            </div>
          )}
          {analysis&&(
            <div style={{display:"flex",flexDirection:"column",gap:12}}>
              <div style={{background:C.panel,borderRadius:14,padding:"18px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
                  <div>
                    <div style={{fontSize:11,color:C.muted,textTransform:"uppercase",letterSpacing:0.8,marginBottom:6}}>Signal</div>
                    <div style={{fontSize:30,fontWeight:800,color:sigColor,letterSpacing:2}}>{analysis.signal}</div>
                    <div style={{fontSize:12,color:C.muted,marginTop:4}}>Confidence: <strong style={{color:C.dark}}>{analysis.conf}</strong></div>
                  </div>
                  <div style={{textAlign:"right"}}>
                    <div style={{fontSize:11,color:C.muted,marginBottom:4}}>Momentum</div>
                    <div style={{fontSize:13,fontWeight:600,color:analysis.momentum==="bullish"?C.green:analysis.momentum==="bearish"?C.red:C.amber,textTransform:"capitalize"}}>{analysis.momentum}</div>
                    <div style={{fontSize:11,color:C.muted,marginTop:10,marginBottom:4}}>Volatility</div>
                    <div style={{fontSize:13,fontWeight:600,color:C.dark}}>{analysis.vol}% σ</div>
                  </div>
                </div>
              </div>
              <div style={{background:C.panel,borderRadius:14,padding:"18px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:14}}>Price Targets — 3–6 months</div>
                <div style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:8}}>
                  {[{l:"Bear",v:analysis.bear,c:C.red},{l:"Base",v:analysis.base,c:C.amber},{l:"Bull",v:analysis.bull,c:C.green}].map(({l,v,c})=>(
                    <div key={l} style={{background:`${c}10`,borderRadius:10,padding:"12px 8px",textAlign:"center",border:`1px solid ${c}30`}}>
                      <div style={{fontSize:10,color:c,fontWeight:600,marginBottom:6,textTransform:"uppercase"}}>{l}</div>
                      <div style={{fontSize:15,fontWeight:700,color:c,fontFamily:"monospace"}}>{sym}{v}</div>
                      <div style={{fontSize:10,color:C.muted,marginTop:4}}>{((v-p.current)/p.current*100).toFixed(1)}%</div>
                    </div>
                  ))}
                </div>
              </div>
              <div style={{background:C.panel,borderRadius:14,padding:"18px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:12}}>Technical Indicators</div>
                {[["RSI (14)",`${analysis.rsi}`,analysis.rsi>70?"Overbought":analysis.rsi<30?"Oversold":"Neutral"],
                  ["MACD",analysis.macd,""],["Support",`${sym}${analysis.support}`,"Key level"],["Resistance",`${sym}${analysis.resistance}`,"Key level"]].map(([l,v,n])=>(
                  <div key={l} style={{display:"flex",justifyContent:"space-between",alignItems:"center",padding:"10px 0",borderBottom:`1px solid ${C.border}`}}>
                    <span style={{fontSize:13,color:C.muted}}>{l}</span>
                    <span style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>{v} <span style={{fontSize:11,color:C.muted,fontFamily:"inherit"}}>{n}</span></span>
                  </div>
                ))}
              </div>
              <div style={{background:C.panel,borderRadius:14,padding:"18px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:10}}>Positives</div>
                {t.bull.map((x,i)=><div key={i} style={{display:"flex",gap:8,marginBottom:8}}><span style={{color:C.green,fontWeight:700}}>+</span><span style={{fontSize:13,color:"#4a5568",lineHeight:1.5}}>{x}</span></div>)}
                <div style={{fontWeight:600,fontSize:13,color:C.dark,margin:"14px 0 10px"}}>Risks</div>
                {t.bear.map((x,i)=><div key={i} style={{display:"flex",gap:8,marginBottom:8}}><span style={{color:C.red,fontWeight:700}}>!</span><span style={{fontSize:13,color:"#4a5568",lineHeight:1.5}}>{x}</span></div>)}
              </div>
              <div style={{background:`${C.accent}08`,border:`1px solid ${C.accent}20`,borderRadius:14,padding:"16px"}}>
                <div style={{fontSize:11,color:C.accent,fontWeight:600,textTransform:"uppercase",letterSpacing:0.8,marginBottom:8}}>◎ Entry Suggestion</div>
                <div style={{fontSize:13,color:"#2d3748",lineHeight:1.6}}>{t.dca}</div>
              </div>
              <button onClick={()=>setAnalysis(generateAnalysis(t,p))} style={{background:"transparent",border:`1px solid ${C.border}`,color:C.muted,padding:"12px",borderRadius:10,fontSize:12}}>
                ↻ Refresh Analysis
              </button>
            </div>
          )}
        </div>
      )}
    </div>
  );
}

// ── Portfolio ─────────────────────────────────────────────────────
function PortfolioScreen({prices,onSelectTicker}){
  const [holdings,setHoldings]=useState(()=>{try{return JSON.parse(localStorage.getItem("cmc_holdings")||"[]");}catch{return [];}});
  const [screen,setScreen]=useState("list");
  const [editId,setEditId]=useState(null);
  const [form,setForm]=useState({ticker:"",units:"",avgCost:""});
  const [err,setErr]=useState("");

  const save=(h)=>{setHoldings(h);try{localStorage.setItem("cmc_holdings",JSON.stringify(h));}catch{}};
  const openAdd=()=>{setForm({ticker:"",units:"",avgCost:""});setErr("");setEditId(null);setScreen("add");};
  const openEdit=(h)=>{setForm({ticker:h.ticker,units:String(h.units),avgCost:String(h.avgCost)});setErr("");setEditId(h.id);setScreen("add");};
  const submit=()=>{
    const ticker=form.ticker.toUpperCase().trim();
    const units=parseFloat(form.units);
    const avgCost=parseFloat(form.avgCost);
    if(!ticker){setErr("Enter a ticker code");return;}
    if(isNaN(units)||units<=0){setErr("Enter valid units");return;}
    if(isNaN(avgCost)||avgCost<=0){setErr("Enter valid avg cost");return;}
    if(editId){save(holdings.map(h=>h.id===editId?{...h,ticker,units,avgCost}:h));}
    else{save([...holdings.filter(h=>h.ticker!==ticker),{id:Date.now(),ticker,units,avgCost}]);}
    setScreen("list");
  };

  const rows=holdings.map(h=>{
    const meta=TICKERS.find(t=>t.id===h.ticker);
    const p=prices[h.ticker];
    const curr=p?.current||h.avgCost;
    const sym=(meta?.currency||"AUD")==="AUD"?"A$":"$";
    const mktVal=curr*h.units,costBase=h.avgCost*h.units,pnl=mktVal-costBase;
    return{...h,curr,sym,mktVal,costBase,pnl,pnlPct:costBase>0?(pnl/costBase)*100:0,
      dayChg:p?.changePct||0,name:meta?.name||h.ticker,flag:meta?.flag||"",icon:meta?.icon||"📊"};
  });
  const totalVal=rows.reduce((s,r)=>s+r.mktVal,0);
  const totalCost=rows.reduce((s,r)=>s+r.costBase,0);
  const totalPnL=totalVal-totalCost;
  const totalPct=totalCost>0?(totalPnL/totalCost)*100:0;

  if(screen==="add") return(
    <div style={{background:C.bg,minHeight:"100%"}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,display:"flex",alignItems:"center",justifyContent:"space-between",padding:"14px 20px"}} className="safe-top">
        <button onClick={()=>setScreen("list")} style={{background:"transparent",border:"none",color:C.accent,fontSize:15,fontWeight:600}}>Cancel</button>
        <span style={{fontWeight:600,fontSize:15,color:C.dark}}>{editId?"Edit":"Add Holding"}</span>
        <button onClick={submit} style={{background:"transparent",border:"none",color:C.accent,fontSize:15,fontWeight:700}}>Save</button>
      </div>
      <div style={{padding:"20px 16px",display:"flex",flexDirection:"column",gap:14}}>
        {[{label:"Ticker Code",key:"ticker",placeholder:"e.g. DHHF",type:"text"},
          {label:"Number of Units",key:"units",placeholder:"e.g. 100",type:"number"},
          {label:"Average Cost per Unit (A$)",key:"avgCost",placeholder:"e.g. 38.50",type:"number"}].map(({label,key,placeholder,type})=>(
          <div key={key} style={{background:C.panel,borderRadius:12,padding:"14px 16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
            <div style={{fontSize:11,color:C.muted,marginBottom:6,textTransform:"uppercase",letterSpacing:0.5}}>{label}</div>
            <input type={type} value={form[key]} onChange={e=>setForm(p=>({...p,[key]:type==="text"?e.target.value.toUpperCase():e.target.value}))} placeholder={placeholder}
              style={{width:"100%",border:"none",outline:"none",fontSize:17,color:C.dark,fontWeight:600,background:"transparent",fontFamily:"monospace"}}/>
          </div>
        ))}
        {err&&<div style={{color:C.red,fontSize:13,textAlign:"center"}}>{err}</div>}
        <div>
          <div style={{fontSize:11,color:C.muted,marginBottom:8,textTransform:"uppercase",letterSpacing:0.5}}>Quick fill</div>
          <div style={{display:"flex",gap:8,flexWrap:"wrap"}}>
            {TICKERS.map(t=>(
              <button key={t.id} onClick={()=>setForm(p=>({...p,ticker:t.id}))}
                style={{background:form.ticker===t.id?C.accent:`${C.accent}10`,border:`1px solid ${form.ticker===t.id?C.accent:C.border}`,color:form.ticker===t.id?"#fff":C.accent,padding:"8px 14px",borderRadius:20,fontSize:13,fontWeight:600}}>
                {t.id}
              </button>
            ))}
          </div>
        </div>
        <button onClick={submit} style={{background:C.accent,border:"none",color:"#fff",padding:"15px",borderRadius:12,fontSize:15,fontWeight:700,marginTop:4}}>
          {editId?"Update":"Add to Portfolio"}
        </button>
      </div>
    </div>
  );

  return(
    <div style={{background:C.bg,minHeight:"100%",paddingBottom:20}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,padding:"16px 20px 14px"}} className="safe-top">
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div>
            <div style={{fontSize:20,fontWeight:700,color:C.dark}}>Portfolio</div>
            <div style={{fontSize:12,color:C.muted,marginTop:2}}>{rows.length} holding{rows.length!==1?"s":""}</div>
          </div>
          <button onClick={openAdd} style={{background:C.accent,border:"none",color:"#fff",width:36,height:36,borderRadius:18,fontSize:22,display:"flex",alignItems:"center",justifyContent:"center"}}>+</button>
        </div>
      </div>
      {rows.length===0?(
        <div style={{padding:"60px 24px",textAlign:"center"}}>
          <div style={{fontSize:48,marginBottom:16}}>📊</div>
          <div style={{fontSize:17,fontWeight:600,color:C.dark,marginBottom:8}}>No holdings yet</div>
          <div style={{fontSize:14,color:C.muted,marginBottom:28,lineHeight:1.6}}>Add your shares from CMC Invest — ticker, units and average cost is all you need.</div>
          <button onClick={openAdd} style={{background:C.accent,border:"none",color:"#fff",padding:"14px 32px",borderRadius:24,fontSize:15,fontWeight:700}}>+ Add First Holding</button>
        </div>
      ):(
        <div style={{padding:"16px"}}>
          <div style={{background:C.panel,borderRadius:14,padding:"18px",marginBottom:14,boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
            <div style={{fontSize:12,color:C.muted,marginBottom:4}}>Total Portfolio Value</div>
            <div style={{fontSize:30,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${totalVal.toFixed(2)}</div>
            <div style={{display:"flex",gap:20,marginTop:10}}>
              <div><div style={{fontSize:11,color:C.muted,marginBottom:2}}>Cost</div><div style={{fontSize:14,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>A${totalCost.toFixed(2)}</div></div>
              <div><div style={{fontSize:11,color:C.muted,marginBottom:2}}>P&L</div><div style={{fontSize:14,fontWeight:700,color:totalPnL>=0?C.green:C.red,fontFamily:"monospace"}}>{totalPnL>=0?"+":""}A${totalPnL.toFixed(2)} ({totalPct>=0?"+":""}{ totalPct.toFixed(2)}%)</div></div>
            </div>
            {totalVal>0&&<div style={{display:"flex",height:8,borderRadius:4,overflow:"hidden",gap:1,marginTop:14}}>
              {rows.map((r,i)=>{const cols=["#1a56db","#00a86b","#e8334a","#f5a623","#7c3aed","#0891b2"];return<div key={r.id} style={{flex:r.mktVal,background:cols[i%cols.length],minWidth:2}}/>;})}</div>}
          </div>
          <div style={{display:"flex",flexDirection:"column",gap:8}}>
            {rows.map(r=>(
              <div key={r.id} style={{background:C.panel,borderRadius:14,padding:"14px 16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                <div onClick={()=>onSelectTicker(r.ticker)} style={{cursor:"pointer"}}>
                  <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start"}}>
                    <div style={{display:"flex",alignItems:"center",gap:10}}>
                      <div style={{width:40,height:40,borderRadius:20,background:`${C.accent}15`,display:"flex",alignItems:"center",justifyContent:"center",fontSize:18}}>{r.icon}</div>
                      <div>
                        <div style={{fontWeight:700,fontSize:15,color:C.dark}}>{r.ticker} <span style={{fontSize:12}}>{r.flag}</span></div>
                        <div style={{fontSize:11,color:C.muted,marginTop:1,maxWidth:160,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"}}>{r.name}</div>
                      </div>
                    </div>
                    <div style={{textAlign:"right"}}>
                      <div style={{fontSize:14,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${r.mktVal.toFixed(2)}</div>
                      <div style={{fontSize:12,fontWeight:600,color:r.pnl>=0?C.green:C.red,marginTop:2}}>{r.pnl>=0?"+":""}A${r.pnl.toFixed(2)} ({r.pnlPct>=0?"+":""}{ r.pnlPct.toFixed(1)}%)</div>
                    </div>
                  </div>
                  <div style={{display:"flex",justifyContent:"space-between",marginTop:10,paddingTop:10,borderTop:`1px solid ${C.border}`}}>
                    {[["Units",r.units],["Avg Cost",`${r.sym}${r.avgCost.toFixed(2)}`],["Current",`${r.sym}${r.curr.toFixed(3)}`],["Today",`${r.dayChg>=0?"+":""}${r.dayChg.toFixed(2)}%`]].map(([l,v],i)=>(
                      <div key={i}><div style={{fontSize:10,color:C.muted,marginBottom:2}}>{l}</div><div style={{fontSize:12,fontWeight:600,color:i===3?(r.dayChg>=0?C.green:C.red):C.dark}}>{v}</div></div>
                    ))}
                  </div>
                </div>
                <div style={{display:"flex",gap:8,marginTop:10}}>
                  <button onClick={()=>openEdit(r)} style={{flex:1,background:`${C.accent}10`,border:`1px solid ${C.accent}30`,color:C.accent,padding:"8px",borderRadius:8,fontSize:12,fontWeight:600}}>Edit</button>
                  <button onClick={()=>save(holdings.filter(h=>h.id!==r.id))} style={{flex:1,background:`${C.red}08`,border:`1px solid ${C.red}20`,color:C.red,padding:"8px",borderRadius:8,fontSize:12,fontWeight:600}}>Remove</button>
                </div>
              </div>
            ))}
          </div>
          <button onClick={openAdd} style={{width:"100%",background:C.accent,border:"none",color:"#fff",padding:"14px",borderRadius:12,fontSize:14,fontWeight:700,marginTop:14}}>+ Add Holding</button>
        </div>
      )}
    </div>
  );
}

// ── Swan Bullion ──────────────────────────────────────────────────
function SwanBullionScreen({sharedBarPrice, onPriceUpdate}){
  const [spotAUD,setSpotAUD]=useState(null);
  const [barPrice,setBarPrice]=useState(sharedBarPrice||null);
  const [loading,setLoading]=useState(false);
  const [lastFetched,setLastFetched]=useState(null);
  const [holdings,setHoldings]=useState(()=>{try{return JSON.parse(localStorage.getItem("swan_holdings")||"[]");}catch{return [];}});
  const [form,setForm]=useState({qty:"",paidPer:""});
  const [history,setHistory]=useState(sharedBarPrice?[sharedBarPrice]:[]);
  const TROY_OZ=5/31.1035,PREMIUM=1.04;

  // Sync if parent already has a price
  useEffect(()=>{ if(sharedBarPrice&&!barPrice) setBarPrice(sharedBarPrice); },[sharedBarPrice]);

  const fetchSpot=useCallback(async()=>{
    setLoading(true);
    try{
      const [fxRes,goldRes]=await Promise.all([
        fetch("https://api.frankfurter.app/latest?from=USD&to=AUD"),
        fetch("https://data-asg.goldprice.org/dbXRates/USD")
      ]);
      const fx=await fxRes.json();
      const gold=await goldRes.json();
      const spotAUDPerOz=gold.items[0].xauPrice*fx.rates.AUD;
      const price=spotAUDPerOz*TROY_OZ*PREMIUM;
      setSpotAUD(spotAUDPerOz);setBarPrice(price);
      setLastFetched(new Date());
      setHistory(h=>[...h.slice(-29),price]);
      if(onPriceUpdate) onPriceUpdate(price); // push back up to App
    }catch{
      const fallback=6800,price=fallback*TROY_OZ*PREMIUM;
      setSpotAUD(fallback);setBarPrice(price);
      setLastFetched(new Date());
      setHistory(h=>[...h.slice(-29),price]);
      if(onPriceUpdate) onPriceUpdate(price);
    }
    setLoading(false);
  },[onPriceUpdate]);

  useEffect(()=>{fetchSpot();const id=setInterval(fetchSpot,300000);return()=>clearInterval(id);},[fetchSpot]);

  const saveH=(h)=>{setHoldings(h);try{localStorage.setItem("swan_holdings",JSON.stringify(h));}catch{}};
  const addHolding=()=>{
    const qty=parseInt(form.qty),paid=parseFloat(form.paidPer);
    if(!qty||qty<1||isNaN(paid)||paid<=0)return;
    saveH([...holdings,{id:Date.now(),qty,paidPer:paid}]);
    setForm({qty:"",paidPer:""});
  };

  const totalBars=holdings.reduce((s,h)=>s+h.qty,0);
  const totalCost=holdings.reduce((s,h)=>s+h.qty*h.paidPer,0);
  const totalValue=barPrice?totalBars*barPrice:0;
  const totalPnL=totalValue-totalCost;
  const totalPct=totalCost>0?(totalPnL/totalCost)*100:0;
  const minH=history.length>1?Math.min(...history):0;
  const maxH=history.length>1?Math.max(...history):1;

  return(
    <div style={{background:C.bg,minHeight:"100%",paddingBottom:20}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,padding:"16px 20px 14px"}} className="safe-top">
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div>
            <div style={{fontSize:20,fontWeight:700,color:C.dark}}>🥇 Swan Bullion</div>
            <div style={{fontSize:12,color:C.muted,marginTop:2}}>Perth Mint 5g Gold Bar · Live AUD Spot</div>
          </div>
          <button onClick={fetchSpot} disabled={loading} style={{background:`${C.accent}10`,border:`1px solid ${C.accent}30`,color:C.accent,padding:"7px 14px",borderRadius:20,fontSize:12,fontWeight:600}}>
            {loading?"…":"↻ Refresh"}
          </button>
        </div>
      </div>
      <div style={{padding:"16px",display:"flex",flexDirection:"column",gap:12}}>
        <div style={{background:C.panel,borderRadius:14,padding:"20px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
          <div style={{fontSize:11,color:C.muted,textTransform:"uppercase",letterSpacing:0.8,marginBottom:6}}>Current Bar Price (Est.)</div>
          <div style={{fontSize:36,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>{barPrice?`A$${barPrice.toFixed(2)}`:"Loading…"}</div>
          <div style={{fontSize:12,color:C.muted,marginTop:4}}>5g · 99.99% fine · Perth Mint · ~4% dealer premium</div>
          {history.length>2&&(
            <div style={{marginTop:14,height:50}}>
              <svg width="100%" viewBox="0 0 300 50" preserveAspectRatio="none" style={{display:"block"}}>
                <defs><linearGradient id="goldGrad" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stopColor={C.amber} stopOpacity="0.2"/><stop offset="100%" stopColor={C.amber} stopOpacity="0"/></linearGradient></defs>
                {(()=>{const pts=history.map((v,i)=>`${(i/(history.length-1))*300},${46-((v-minH)/(maxH-minH||1))*42}`).join(" ");
                  return<><polygon points={`0,46 ${pts} 300,46`} fill="url(#goldGrad)"/><polyline points={pts} fill="none" stroke={C.amber} strokeWidth="2" strokeLinejoin="round"/></>;})()}
              </svg>
            </div>
          )}
          <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:12,marginTop:14,paddingTop:14,borderTop:`1px solid ${C.border}`}}>
            {[["AUD Gold Spot",spotAUD?`A$${spotAUD.toFixed(2)}/oz`:"—"],
              ["Per gram",spotAUD?`A$${(spotAUD/31.1035).toFixed(2)}/g`:"—"],
              ["Pure metal",spotAUD?`A$${(spotAUD*TROY_OZ).toFixed(2)}`:"—"],
              ["Updated",lastFetched?lastFetched.toLocaleTimeString():"—"]].map(([l,v])=>(
              <div key={l}><div style={{fontSize:11,color:C.muted,marginBottom:3}}>{l}</div><div style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>{v}</div></div>
            ))}
          </div>
        </div>

        {totalBars>0&&(
          <div style={{background:C.panel,borderRadius:14,padding:"18px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
            <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:14}}>My Holdings</div>
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10,marginBottom:14}}>
              {[["Total Bars",`${totalBars} × 5g`],["Total Weight",`${totalBars*5}g`],
                ["Current Value",barPrice?`A$${totalValue.toFixed(2)}`:"—"],["Total Cost",`A$${totalCost.toFixed(2)}`]].map(([l,v])=>(
                <div key={l} style={{background:C.bg,borderRadius:8,padding:"10px 12px"}}>
                  <div style={{fontSize:10,color:C.muted,marginBottom:3,textTransform:"uppercase"}}>{l}</div>
                  <div style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>{v}</div>
                </div>
              ))}
            </div>
            <div style={{background:totalPnL>=0?`${C.green}10`:`${C.red}10`,borderRadius:10,padding:"12px 14px",border:`1px solid ${totalPnL>=0?C.green:C.red}25`}}>
              <div style={{fontSize:11,color:C.muted,marginBottom:4}}>Total P&L</div>
              <div style={{fontSize:22,fontWeight:700,color:totalPnL>=0?C.green:C.red,fontFamily:"monospace"}}>{totalPnL>=0?"+":""}A${totalPnL.toFixed(2)}</div>
              <div style={{fontSize:12,color:totalPnL>=0?C.green:C.red,marginTop:2}}>{totalPct>=0?"+":""}{ totalPct.toFixed(2)}% return</div>
            </div>
            {holdings.map((h,i)=>(
              <div key={h.id} style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginTop:10,paddingTop:10,borderTop:`1px solid ${C.border}`}}>
                <div><div style={{fontSize:13,color:C.dark,fontWeight:600}}>Lot {i+1}: {h.qty} bar{h.qty>1?"s":""}</div><div style={{fontSize:11,color:C.muted}}>Paid A${h.paidPer.toFixed(2)}/bar</div></div>
                <div style={{textAlign:"right"}}>
                  <div style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>{barPrice?`A$${(h.qty*barPrice).toFixed(2)}`:"—"}</div>
                  {barPrice&&<div style={{fontSize:11,color:(barPrice-h.paidPer)>=0?C.green:C.red}}>{(barPrice-h.paidPer)>=0?"+":""}A${((barPrice-h.paidPer)*h.qty).toFixed(2)}</div>}
                </div>
                <button onClick={()=>saveH(holdings.filter(x=>x.id!==h.id))} style={{background:"transparent",border:`1px solid ${C.border}`,color:C.muted,padding:"4px 8px",borderRadius:6,fontSize:11,marginLeft:10}}>✕</button>
              </div>
            ))}
          </div>
        )}

        <div style={{background:C.panel,borderRadius:14,padding:"18px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
          <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:14}}>{totalBars>0?"Add More Bars":"Add Your Bars"}</div>
          <div style={{display:"flex",flexDirection:"column",gap:10}}>
            {[{label:"Number of bars",key:"qty",placeholder:"e.g. 2",type:"number"},
              {label:"Price paid per bar (A$)",key:"paidPer",placeholder:barPrice?barPrice.toFixed(2):"1100.00",type:"number"}].map(({label,key,placeholder,type})=>(
              <div key={key} style={{background:C.bg,borderRadius:10,padding:"12px 14px"}}>
                <div style={{fontSize:11,color:C.muted,marginBottom:6,textTransform:"uppercase",letterSpacing:0.5}}>{label}</div>
                <input type={type} value={form[key]} onChange={e=>setForm(p=>({...p,[key]:e.target.value}))} placeholder={placeholder}
                  style={{width:"100%",border:"none",outline:"none",fontSize:17,color:C.dark,fontWeight:600,background:"transparent",fontFamily:"monospace"}}/>
              </div>
            ))}
            <button onClick={addHolding} style={{background:C.amber,border:"none",color:"#fff",padding:"14px",borderRadius:12,fontSize:14,fontWeight:700}}>+ Add to Holdings</button>
          </div>
        </div>
      </div>
    </div>
  );
}


// ── Savings Screen ────────────────────────────────────────────────
function SavingsScreen(){
  const [accounts,setAccounts]=useState(()=>{try{return JSON.parse(localStorage.getItem("savings_accounts")||"[]");}catch{return [];}});
  const [screen,setScreen]=useState("list");
  const [editId,setEditId]=useState(null);
  const [form,setForm]=useState({name:"",balance:"",rate:"",type:"savings"});
  const [err,setErr]=useState("");

  const save=(a)=>{setAccounts(a);try{localStorage.setItem("savings_accounts",JSON.stringify(a));}catch{}};
  const openAdd=()=>{setForm({name:"",balance:"",rate:"",type:"savings"});setErr("");setEditId(null);setScreen("add");};
  const openEdit=(a)=>{setForm({name:a.name,balance:String(a.balance),rate:String(a.rate),type:a.type});setErr("");setEditId(a.id);setScreen("add");};
  const submit=()=>{
    const name=form.name.trim();
    const balance=parseFloat(form.balance);
    const rate=parseFloat(form.rate)||0;
    if(!name){setErr("Enter account name");return;}
    if(isNaN(balance)||balance<0){setErr("Enter valid balance");return;}
    const entry={id:editId||Date.now(),name,balance,rate,type:form.type,updatedAt:new Date().toLocaleDateString()};
    if(editId){save(accounts.map(a=>a.id===editId?entry:a));}
    else{save([...accounts,entry]);}
    setScreen("list");
  };

  const totalSavings=accounts.reduce((s,a)=>s+a.balance,0);
  const annualInterest=accounts.reduce((s,a)=>s+(a.balance*a.rate/100),0);

  const typeIcons={"savings":"🏦","offset":"🏠","term":"📅","cash":"💵","super":"🦘","other":"💰"};
  const typeLabels={"savings":"Savings","offset":"Offset","term":"Term Deposit","cash":"Cash","super":"Super","other":"Other"};

  if(screen==="add") return(
    <div style={{background:C.bg,minHeight:"100%"}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,display:"flex",alignItems:"center",justifyContent:"space-between",padding:"14px 20px"}} className="safe-top">
        <button onClick={()=>setScreen("list")} style={{background:"transparent",border:"none",color:C.accent,fontSize:15,fontWeight:600}}>Cancel</button>
        <span style={{fontWeight:600,fontSize:15,color:C.dark}}>{editId?"Edit Account":"Add Account"}</span>
        <button onClick={submit} style={{background:"transparent",border:"none",color:C.accent,fontSize:15,fontWeight:700}}>Save</button>
      </div>
      <div style={{padding:"20px 16px",display:"flex",flexDirection:"column",gap:14}}>
        <div style={{background:C.panel,borderRadius:12,padding:"14px 16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
          <div style={{fontSize:11,color:C.muted,marginBottom:6,textTransform:"uppercase",letterSpacing:0.5}}>Account Name</div>
          <input value={form.name} onChange={e=>setForm(p=>({...p,name:e.target.value}))} placeholder="e.g. ING Savings, Offset Account"
            style={{width:"100%",border:"none",outline:"none",fontSize:17,color:C.dark,fontWeight:600,background:"transparent"}}/>
        </div>
        <div style={{background:C.panel,borderRadius:12,padding:"14px 16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
          <div style={{fontSize:11,color:C.muted,marginBottom:6,textTransform:"uppercase",letterSpacing:0.5}}>Current Balance (A$)</div>
          <input type="number" value={form.balance} onChange={e=>setForm(p=>({...p,balance:e.target.value}))} placeholder="e.g. 25000"
            style={{width:"100%",border:"none",outline:"none",fontSize:17,color:C.dark,fontWeight:600,background:"transparent",fontFamily:"monospace"}}/>
        </div>
        <div style={{background:C.panel,borderRadius:12,padding:"14px 16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
          <div style={{fontSize:11,color:C.muted,marginBottom:6,textTransform:"uppercase",letterSpacing:0.5}}>Interest Rate % p.a. (optional)</div>
          <input type="number" value={form.rate} onChange={e=>setForm(p=>({...p,rate:e.target.value}))} placeholder="e.g. 5.00"
            style={{width:"100%",border:"none",outline:"none",fontSize:17,color:C.dark,fontWeight:600,background:"transparent",fontFamily:"monospace"}}/>
        </div>
        <div style={{background:C.panel,borderRadius:12,padding:"14px 16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
          <div style={{fontSize:11,color:C.muted,marginBottom:10,textTransform:"uppercase",letterSpacing:0.5}}>Account Type</div>
          <div style={{display:"flex",gap:8,flexWrap:"wrap"}}>
            {Object.entries(typeLabels).map(([key,label])=>(
              <button key={key} onClick={()=>setForm(p=>({...p,type:key}))}
                style={{background:form.type===key?C.accent:`${C.accent}10`,border:`1px solid ${form.type===key?C.accent:C.border}`,color:form.type===key?"#fff":C.accent,padding:"7px 12px",borderRadius:20,fontSize:12,fontWeight:600}}>
                {typeIcons[key]} {label}
              </button>
            ))}
          </div>
        </div>
        {err&&<div style={{color:C.red,fontSize:13,textAlign:"center"}}>{err}</div>}
        <button onClick={submit} style={{background:C.accent,border:"none",color:"#fff",padding:"15px",borderRadius:12,fontSize:15,fontWeight:700,marginTop:4}}>
          {editId?"Update Account":"Add Account"}
        </button>
      </div>
    </div>
  );

  return(
    <div style={{background:C.bg,minHeight:"100%",paddingBottom:20}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,padding:"16px 20px 14px"}} className="safe-top">
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div>
            <div style={{fontSize:20,fontWeight:700,color:C.dark}}>Savings</div>
            <div style={{fontSize:12,color:C.muted,marginTop:2}}>{accounts.length} account{accounts.length!==1?"s":""}</div>
          </div>
          <button onClick={openAdd} style={{background:C.accent,border:"none",color:"#fff",width:36,height:36,borderRadius:18,fontSize:22,display:"flex",alignItems:"center",justifyContent:"center"}}>+</button>
        </div>
      </div>

      {accounts.length===0?(
        <div style={{padding:"60px 24px",textAlign:"center"}}>
          <div style={{fontSize:48,marginBottom:16}}>🏦</div>
          <div style={{fontSize:17,fontWeight:600,color:C.dark,marginBottom:8}}>No accounts yet</div>
          <div style={{fontSize:14,color:C.muted,marginBottom:28,lineHeight:1.6}}>Add your savings, offset, term deposits and cash accounts to track your total wealth.</div>
          <button onClick={openAdd} style={{background:C.accent,border:"none",color:"#fff",padding:"14px 32px",borderRadius:24,fontSize:15,fontWeight:700}}>+ Add First Account</button>
        </div>
      ):(
        <div style={{padding:"16px"}}>
          <div style={{background:C.panel,borderRadius:14,padding:"18px",marginBottom:14,boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
            <div style={{fontSize:12,color:C.muted,marginBottom:4}}>Total Savings</div>
            <div style={{fontSize:30,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${totalSavings.toLocaleString("en-AU",{minimumFractionDigits:2,maximumFractionDigits:2})}</div>
            {annualInterest>0&&<div style={{fontSize:13,color:C.green,marginTop:6,fontWeight:600}}>≈ A${annualInterest.toFixed(2)}/year interest</div>}
          </div>
          <div style={{display:"flex",flexDirection:"column",gap:8}}>
            {accounts.map(a=>(
              <div key={a.id} style={{background:C.panel,borderRadius:14,padding:"16px",boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start"}}>
                  <div style={{display:"flex",alignItems:"center",gap:10}}>
                    <div style={{width:44,height:44,borderRadius:22,background:`${C.accent}15`,display:"flex",alignItems:"center",justifyContent:"center",fontSize:20}}>
                      {typeIcons[a.type]||"💰"}
                    </div>
                    <div>
                      <div style={{fontWeight:700,fontSize:15,color:C.dark}}>{a.name}</div>
                      <div style={{fontSize:11,color:C.muted,marginTop:1}}>{typeLabels[a.type]||"Account"}{a.rate>0?` · ${a.rate}% p.a.`:""}</div>
                    </div>
                  </div>
                  <div style={{textAlign:"right"}}>
                    <div style={{fontSize:16,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${a.balance.toLocaleString("en-AU",{minimumFractionDigits:2})}</div>
                    {a.rate>0&&<div style={{fontSize:11,color:C.green,marginTop:2}}>+A${(a.balance*a.rate/100).toFixed(2)}/yr</div>}
                  </div>
                </div>
                {a.updatedAt&&<div style={{fontSize:10,color:C.muted,marginTop:8}}>Updated {a.updatedAt}</div>}
                <div style={{display:"flex",gap:8,marginTop:10}}>
                  <button onClick={()=>openEdit(a)} style={{flex:1,background:`${C.accent}10`,border:`1px solid ${C.accent}30`,color:C.accent,padding:"8px",borderRadius:8,fontSize:12,fontWeight:600}}>Edit</button>
                  <button onClick={()=>save(accounts.filter(x=>x.id!==a.id))} style={{flex:1,background:`${C.red}08`,border:`1px solid ${C.red}20`,color:C.red,padding:"8px",borderRadius:8,fontSize:12,fontWeight:600}}>Remove</button>
                </div>
              </div>
            ))}
          </div>
          <button onClick={openAdd} style={{width:"100%",background:C.accent,border:"none",color:"#fff",padding:"14px",borderRadius:12,fontSize:14,fontWeight:700,marginTop:14}}>+ Add Account</button>
        </div>
      )}
    </div>
  );
}

// ── Total Portfolio Screen ─────────────────────────────────────────
function TotalPortfolioScreen({prices, swanBarPrice}){
  // Load all data from localStorage
  const holdings=JSON.parse(localStorage.getItem("cmc_holdings")||"[]");
  const swanHoldings=JSON.parse(localStorage.getItem("swan_holdings")||"[]");
  const savings=JSON.parse(localStorage.getItem("savings_accounts")||"[]");
  // swanBarPrice is passed in from App (same live fetch used by Gold Bar tab)

  // Shares
  const shareRows=holdings.map(h=>{
    const meta=TICKERS.find(t=>t.id===h.ticker);
    const p=prices[h.ticker];
    const curr=p?.current||h.avgCost;
    const mktVal=curr*h.units;
    const costBase=h.avgCost*h.units;
    return{label:h.ticker,value:mktVal,cost:costBase,pnl:mktVal-costBase,icon:meta?.icon||"📊",color:"#1a56db"};
  });
  const totalShares=shareRows.reduce((s,r)=>s+r.value,0);
  const totalSharesCost=shareRows.reduce((s,r)=>s+r.cost,0);

  // Swan gold
  const swanBars=swanHoldings.reduce((s,h)=>s+h.qty,0);
  const swanValue=swanBars*swanBarPrice;
  const swanCost=swanHoldings.reduce((s,h)=>s+h.qty*h.paidPer,0);

  // Savings
  const totalSavings=savings.reduce((s,a)=>s+a.balance,0);
  const annualInterest=savings.reduce((s,a)=>s+(a.balance*a.rate/100),0);

  // Totals
  const totalWealth=totalShares+swanValue+totalSavings;
  const totalInvested=totalSharesCost+swanCost;
  const totalPnL=(totalShares-totalSharesCost)+(swanValue-swanCost);
  const totalPct=totalInvested>0?(totalPnL/totalInvested)*100:0;

  const segments=[
    {label:"Shares & ETFs",value:totalShares,cost:totalSharesCost,color:"#1a56db",icon:"📈"},
    {label:"Physical Gold",value:swanValue,cost:swanCost,color:"#f5a623",icon:"🥇"},
    {label:"Savings & Cash",value:totalSavings,cost:totalSavings,color:"#00a86b",icon:"🏦"},
  ].filter(s=>s.value>0);

  // Donut chart via SVG arcs
  const DonutChart=({segments,total,size=180})=>{
    if(!total) return null;
    let cum=0;
    const cx=size/2,cy=size/2,r=size*0.38,ir=size*0.24;
    const paths=segments.map(s=>{
      const pct=s.value/total;
      const start=cum*2*Math.PI-Math.PI/2;
      cum+=pct;
      const end=cum*2*Math.PI-Math.PI/2;
      const large=pct>0.5?1:0;
      const x1=cx+r*Math.cos(start),y1=cy+r*Math.sin(start);
      const x2=cx+r*Math.cos(end),y2=cy+r*Math.sin(end);
      const ix1=cx+ir*Math.cos(start),iy1=cy+ir*Math.sin(start);
      const ix2=cx+ir*Math.cos(end),iy2=cy+ir*Math.sin(end);
      return{path:`M${x1},${y1} A${r},${r} 0 ${large},1 ${x2},${y2} L${ix2},${iy2} A${ir},${ir} 0 ${large},0 ${ix1},${iy1} Z`,color:s.color,label:s.label,pct:(pct*100).toFixed(1)};
    });
    return(
      <svg width={size} height={size} viewBox={`0 0 ${size} ${size}`}>
        {paths.map((p,i)=><path key={i} d={p.path} fill={p.color} opacity="0.9"/>)}
        <text x={cx} y={cy-8} textAnchor="middle" fontSize="11" fill={C.muted}>Total</text>
        <text x={cx} y={cy+10} textAnchor="middle" fontSize="13" fontWeight="700" fill={C.dark}>
          {`A$${(total/1000).toFixed(0)}k`}
        </text>
      </svg>
    );
  };

  return(
    <div style={{background:C.bg,minHeight:"100%",paddingBottom:20}}>
      <div style={{background:C.panel,borderBottom:`1px solid ${C.border}`,padding:"16px 20px 14px"}} className="safe-top">
        <div style={{fontSize:20,fontWeight:700,color:C.dark}}>Total Portfolio</div>
        <div style={{fontSize:12,color:C.muted,marginTop:2}}>Shares · Gold · Savings</div>
      </div>

      {totalWealth===0?(
        <div style={{padding:"60px 24px",textAlign:"center"}}>
          <div style={{fontSize:48,marginBottom:16}}>🌐</div>
          <div style={{fontSize:17,fontWeight:600,color:C.dark,marginBottom:8}}>Nothing here yet</div>
          <div style={{fontSize:14,color:C.muted,lineHeight:1.6}}>Add your shares in Portfolio, gold bars in Gold Bar, and cash in Savings — it all rolls up here.</div>
        </div>
      ):(
        <div style={{padding:"16px",display:"flex",flexDirection:"column",gap:12}}>

          {/* Hero total */}
          <div style={{background:C.panel,borderRadius:16,padding:"20px",boxShadow:"0 2px 8px rgba(0,0,0,0.08)"}}>
            <div style={{fontSize:12,color:C.muted,marginBottom:4}}>Total Net Worth</div>
            <div style={{fontSize:34,fontWeight:800,color:C.dark,fontFamily:"monospace"}}>
              A${totalWealth.toLocaleString("en-AU",{minimumFractionDigits:2,maximumFractionDigits:2})}
            </div>
            {totalInvested>0&&(
              <div style={{marginTop:8}}>
                <span style={{fontSize:14,fontWeight:600,color:totalPnL>=0?C.green:C.red}}>
                  {totalPnL>=0?"+":""}A${totalPnL.toFixed(2)} ({totalPct>=0?"+":""}{totalPct.toFixed(2)}%)
                </span>
                <span style={{fontSize:12,color:C.muted,marginLeft:8}}>on invested assets</span>
              </div>
            )}
            {annualInterest>0&&<div style={{fontSize:12,color:C.green,marginTop:4}}>+ A${annualInterest.toFixed(2)}/year savings interest</div>}
          </div>

          {/* Donut + legend */}
          <div style={{background:C.panel,borderRadius:16,padding:"20px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
            <div style={{fontWeight:600,fontSize:13,color:C.dark,marginBottom:16}}>Asset Allocation</div>
            <div style={{display:"flex",alignItems:"center",gap:20}}>
              <DonutChart segments={segments} total={totalWealth}/>
              <div style={{flex:1,display:"flex",flexDirection:"column",gap:10}}>
                {segments.map(s=>{
                  const pct=(s.value/totalWealth*100).toFixed(1);
                  return(
                    <div key={s.label}>
                      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:4}}>
                        <div style={{display:"flex",alignItems:"center",gap:6}}>
                          <div style={{width:10,height:10,borderRadius:2,background:s.color}}/>
                          <span style={{fontSize:12,color:C.muted}}>{s.icon} {s.label}</span>
                        </div>
                        <span style={{fontSize:12,fontWeight:600,color:C.dark}}>{pct}%</span>
                      </div>
                      <div style={{height:4,background:C.bg,borderRadius:2,overflow:"hidden"}}>
                        <div style={{height:"100%",width:`${pct}%`,background:s.color,borderRadius:2}}/>
                      </div>
                      <div style={{fontSize:11,color:C.muted,marginTop:2,fontFamily:"monospace"}}>A${s.value.toLocaleString("en-AU",{minimumFractionDigits:2,maximumFractionDigits:2})}</div>
                    </div>
                  );
                })}
              </div>
            </div>
          </div>

          {/* Shares breakdown */}
          {shareRows.length>0&&(
            <div style={{background:C.panel,borderRadius:16,padding:"18px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
              <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:14}}>
                <div style={{fontWeight:600,fontSize:13,color:C.dark}}>📈 Shares & ETFs</div>
                <div style={{textAlign:"right"}}>
                  <div style={{fontSize:14,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${totalShares.toFixed(2)}</div>
                  <div style={{fontSize:11,color:(totalShares-totalSharesCost)>=0?C.green:C.red}}>{(totalShares-totalSharesCost)>=0?"+":""}A${(totalShares-totalSharesCost).toFixed(2)}</div>
                </div>
              </div>
              {shareRows.map(r=>(
                <div key={r.label} style={{display:"flex",justifyContent:"space-between",alignItems:"center",padding:"8px 0",borderTop:`1px solid ${C.border}`}}>
                  <div style={{display:"flex",alignItems:"center",gap:8}}>
                    <span style={{fontSize:16}}>{r.icon}</span>
                    <span style={{fontSize:13,fontWeight:600,color:C.dark}}>{r.label}</span>
                  </div>
                  <div style={{textAlign:"right"}}>
                    <div style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>A${r.value.toFixed(2)}</div>
                    <div style={{fontSize:11,color:r.pnl>=0?C.green:C.red}}>{r.pnl>=0?"+":""}A${r.pnl.toFixed(2)}</div>
                  </div>
                </div>
              ))}
            </div>
          )}

          {/* Gold bars */}
          {swanValue>0&&(
            <div style={{background:C.panel,borderRadius:16,padding:"18px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
              <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
                <div>
                  <div style={{fontWeight:600,fontSize:13,color:C.dark}}>🥇 Physical Gold Bars</div>
                  <div style={{fontSize:11,color:C.muted,marginTop:2}}>{swanBars} × 5g Perth Mint bar{swanBars!==1?"s":""} ({swanBars*5}g total)</div>
                </div>
                <div style={{textAlign:"right"}}>
                  <div style={{fontSize:14,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${swanValue.toFixed(2)}</div>
                  <div style={{fontSize:11,color:(swanValue-swanCost)>=0?C.green:C.red}}>{(swanValue-swanCost)>=0?"+":""}A${(swanValue-swanCost).toFixed(2)}</div>
                </div>
              </div>
            </div>
          )}

          {/* Savings breakdown */}
          {totalSavings>0&&(
            <div style={{background:C.panel,borderRadius:16,padding:"18px",boxShadow:"0 1px 4px rgba(0,0,0,0.08)"}}>
              <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:savings.length>1?14:0}}>
                <div style={{fontWeight:600,fontSize:13,color:C.dark}}>🏦 Savings & Cash</div>
                <div style={{textAlign:"right"}}>
                  <div style={{fontSize:14,fontWeight:700,color:C.dark,fontFamily:"monospace"}}>A${totalSavings.toLocaleString("en-AU",{minimumFractionDigits:2,maximumFractionDigits:2})}</div>
                  {annualInterest>0&&<div style={{fontSize:11,color:C.green}}>+A${annualInterest.toFixed(2)}/yr</div>}
                </div>
              </div>
              {savings.map(a=>(
                <div key={a.id} style={{display:"flex",justifyContent:"space-between",alignItems:"center",padding:"8px 0",borderTop:`1px solid ${C.border}`}}>
                  <span style={{fontSize:13,color:C.muted}}>{a.name}</span>
                  <span style={{fontSize:13,fontWeight:600,color:C.dark,fontFamily:"monospace"}}>A${a.balance.toLocaleString("en-AU",{minimumFractionDigits:2,maximumFractionDigits:2})}</span>
                </div>
              ))}
            </div>
          )}

        </div>
      )}
    </div>
  );
}

// ── Root App ──────────────────────────────────────────────────────
function App(){
  const prices=useLivePrices();
  const [selected,setSelected]=useState(null);
  const [mainTab,setMainTab]=useState("watchlist");

  // ── Live Swan Bullion bar price (shared across Gold Bar + Total tabs) ──
  const TROY_OZ=5/31.1035, PREMIUM=1.04;
  const [swanBarPrice,setSwanBarPrice]=useState(null);

  const fetchSwanPrice=useCallback(async()=>{
    try{
      const [fxRes,goldRes]=await Promise.all([
        fetch("https://api.frankfurter.app/latest?from=USD&to=AUD"),
        fetch("https://data-asg.goldprice.org/dbXRates/USD")
      ]);
      const fx=await fxRes.json();
      const gold=await goldRes.json();
      const spotAUDPerOz=gold.items[0].xauPrice*fx.rates.AUD;
      setSwanBarPrice(spotAUDPerOz*TROY_OZ*PREMIUM);
    }catch{
      // fallback: derive from GOLD ETF (each unit ≈ 1/100 oz)
      const goldETFPrice=prices["GOLD"]?.current||57.04;
      setSwanBarPrice(goldETFPrice*100*TROY_OZ*PREMIUM);
    }
  },[prices]);

  useEffect(()=>{
    fetchSwanPrice();
    const id=setInterval(fetchSwanPrice, 5*60*1000);
    return()=>clearInterval(id);
  },[]);

  if(selected) return <StockDetail tickerId={selected} prices={prices} onBack={()=>setSelected(null)}/>;

  return(
    <div style={{display:"flex",flexDirection:"column",height:"100vh",background:C.bg,overflow:"hidden"}}>
      <div style={{flex:1,overflowY:"auto",WebkitOverflowScrolling:"touch"}}>
        {mainTab==="watchlist"?<WatchlistScreen prices={prices} onSelect={setSelected}/>
        :mainTab==="portfolio"?<PortfolioScreen prices={prices} onSelectTicker={setSelected}/>
        :mainTab==="gold"?<SwanBullionScreen sharedBarPrice={swanBarPrice} onPriceUpdate={setSwanBarPrice}/>
        :mainTab==="savings"?<SavingsScreen/>
        :<TotalPortfolioScreen prices={prices} swanBarPrice={swanBarPrice}/>}
      </div>
      <div style={{background:C.panel,borderTop:`1px solid ${C.border}`,display:"flex",flexShrink:0}} className="safe-bot">
        {[{id:"watchlist",icon:"📈",label:"Watchlist"},{id:"portfolio",icon:"💼",label:"Portfolio"},{id:"gold",icon:"🥇",label:"Gold"},{id:"savings",icon:"🏦",label:"Savings"},{id:"total",icon:"🌐",label:"Total"}].map(tab=>(
          <button key={tab.id} onClick={()=>setMainTab(tab.id)} style={{flex:1,background:"transparent",border:"none",padding:"10px 0 4px",display:"flex",flexDirection:"column",alignItems:"center",gap:2}}>
            <span style={{fontSize:20}}>{tab.icon}</span>
            <span style={{fontSize:9,fontWeight:600,color:mainTab===tab.id?C.accent:C.muted,letterSpacing:0}}>{tab.label}</span>
            {mainTab===tab.id&&<div style={{width:20,height:2,background:C.accent,borderRadius:1,marginTop:2}}/>}
          </button>
        ))}
      </div>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<App/>);
</script>

<!-- PWA Service Worker for offline support -->
<script>
if('serviceWorker' in navigator){
  const sw=`
    const CACHE='stock-tracker-v1';
    const ASSETS=['/'];
    self.addEventListener('install',e=>e.waitUntil(caches.open(CACHE).then(c=>c.addAll(ASSETS))));
    self.addEventListener('fetch',e=>e.respondWith(
      caches.match(e.request).then(r=>r||fetch(e.request).catch(()=>caches.match('/')))
    ));
  `;
  const blob=new Blob([sw],{type:'application/javascript'});
  navigator.serviceWorker.register(URL.createObjectURL(blob)).catch(()=>{});
}
</script>
</body>
</html>
