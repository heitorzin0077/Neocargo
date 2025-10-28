<!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Dashboard - Monitoramento de Contêiner</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <style>
    :root{
      --bg:#0f1724; --card:#0b1220; --accent:#06b6d4; --muted:#94a3b8; --danger:#ef4444; --ok:#22c55e;
      --glass: rgba(255,255,255,0.03);
    }
    *{box-sizing:border-box;font-family:Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial}
    body{margin:0;background:linear-gradient(180deg,#071025 0%, #071a2b 100%);color:#e6eef8}
    .wrap{max-width:1200px;margin:18px auto;padding:18px}
    header{display:flex;align-items:center;justify-content:space-between;margin-bottom:18px}
    header h1{font-size:20px;margin:0}
    .grid{display:grid;grid-template-columns: 1fr 360px;gap:18px}
    .cards{display:grid;grid-template-columns:repeat(2,1fr);gap:12px}
    .card{background:var(--card);padding:12px;border-radius:12px;box-shadow:0 6px 18px rgba(2,6,23,0.6)}
    .small{font-size:13px;color:var(--muted)}
    .sensor-value{font-size:28px;font-weight:600}
    .row{display:flex;gap:8px;align-items:center}
    #map{height:300px;border-radius:10px;overflow:hidden}
    .status-ok{color:var(--ok);font-weight:600}
    .status-err{color:var(--danger);font-weight:600}
    .controls{display:flex;gap:8px}
    button{background:var(--glass);color:var(--accent);border:1px solid rgba(255,255,255,0.04);padding:8px 10px;border-radius:8px;cursor:pointer}
    button.ghost{background:transparent;border:1px solid rgba(255,255,255,0.06)}
    footer{margin-top:12px;color:var(--muted);font-size:13px}
    .log{height:150px;overflow:auto;background:rgba(255,255,255,0.02);padding:8px;border-radius:8px;font-family:monospace;font-size:12px}
    .indicators{display:flex;flex-direction:column;gap:8px}
    .ind{display:flex;align-items:center;justify-content:space-between;padding:8px;border-radius:8px;background:rgba(255,255,255,0.01)}
    .chart-wrap{height:180px}
    @media(max-width:900px){
      .grid{grid-template-columns:1fr;}
      .cards{grid-template-columns:1fr}
    }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Dashboard de Monitoramento de Contêiner</h1>
      <div class="controls">
        <button id="btn-sim">Simular dados</button>
        <button id="btn-ws" class="ghost">Conectar WebSocket</button>
        <button id="btn-pause" class="ghost">Pausar</button>
      </div>
    </header>

    <div class="grid">
      <main>
        <div class="cards">
          <div class="card">
            <div class="small">Temperatura</div>
            <div class="row" style="justify-content:space-between;margin-top:8px">
              <div>
                <div class="sensor-value" id="temp">-- °C</div>
                <div class="small">Intervalo ideal: -5 °C a 25 °C</div>
              </div>
              <div style="width:220px" class="chart-wrap">
                <canvas id="chartTemp"></canvas>
              </div>
            </div>
          </div>

          <div class="card">
            <div class="small">Umidade</div>
            <div class="row" style="justify-content:space-between;margin-top:8px">
              <div>
                <div class="sensor-value" id="hum">-- %</div>
                <div class="small">Intervalo ideal: 30% a 70%</div>
              </div>
              <div style="width:220px" class="chart-wrap">
                <canvas id="chartHum"></canvas>
              </div>
            </div>
          </div>

          <div class="card">
            <div class="small">Localização (tempo real)</div>
            <div id="map"></div>
            <div class="small" style="margin-top:8px">Lat: <span id="lat">--</span> Lon: <span id="lon">--</span> Velocidade: <span id="speed">--</span></div>
          </div>

          <div class="card">
            <div class="small">Alarmes e Detecções</div>
            <div class="indicators" style="margin-top:8px">
              <div class="ind"><div>Fumaça</div><div id="smoke" class="status-ok">OK</div></div>
              <div class="ind"><div>Tombamento</div><div id="tilt" class="status-ok">OK</div></div>
              <div class="ind"><div>Abertura de Porta</div><div id="door" class="status-ok">Fechada</div></div>
            </div>
            <div style="margin-top:12px">
              <div class="small">Logs</div>
              <div class="log" id="log"></div>
            </div>
          </div>
        </div>

        <footer>
          Substitua o WebSocket pelo seu endpoint real. O painel aceita mensagens JSON (ex.: veja instruções abaixo).
        </footer>
      </main>

      <aside>
        <div class="card" style="margin-bottom:12px">
          <div class="small">Conexão</div>
          <div style="margin-top:8px">
            <input id="wsUrl" placeholder="ws://seu-endpoint:porta" style="width:100%;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit" />
            <div class="small" style="margin-top:8px">Status: <span id="wsStatus">Desconectado</span></div>
          </div>
        </div>

        <div class="card">
          <div class="small">Configurações</div>
          <div style="margin-top:8px">
            <label class="small">Intervalo de atualização: <input id="interval" type="number" value="2000" style="width:90px;margin-left:8px;background:transparent;color:inherit"/> ms</label>
            <div style="margin-top:8px" class="small">Botões:</div>
            <div style="display:flex;gap:8px;margin-top:8px">
              <button id="btn-clear">Limpar logs</button>
              <button id="btn-export">Exportar CSV</button>
            </div>
          </div>
        </div>

        <div class="card">
          <div class="small">Formato JSON esperado (exemplo)</div>
          <pre style="font-size:12px;background:rgba(255,255,255,0.02);padding:8px;border-radius:6px;margin-top:8px;overflow:auto">{
  "deviceId":"C123",
  "timestamp":1696051200000,
  "temp":4.2,
  "hum":65,
  "smoke":false,
  "tilt":0.5,
  "doorOpen":false,
  "gps":{ "lat":-22.72, "lon":-43.20, "speed":12.3 }
}</pre>
        </div>
      </aside>
    </div>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <script>
    const tempEl=document.getElementById('temp'),humEl=document.getElementById('hum'),
    latEl=document.getElementById('lat'),lonEl=document.getElementById('lon'),speedEl=document.getElementById('speed'),
    smokeEl=document.getElementById('smoke'),tiltEl=document.getElementById('tilt'),doorEl=document.getElementById('door'),
    logEl=document.getElementById('log'),wsUrlEl=document.getElementById('wsUrl'),wsStatusEl=document.getElementById('wsStatus'),
    intervalInput=document.getElementById('interval');

    const map=L.map('map',{zoomControl:false}).setView([-22.9,-43.2],6);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
    const marker=L.marker([-22.9,-43.2]).addTo(map);

    function makeChart(ctx,label){return new Chart(ctx,{type:'line',data:{labels:[],datasets:[{label:label,data:[],fill:true,tension:0.3}]},options:{animation:false,scales:{x:{display:false}}}});}
    const chartTemp=makeChart(document.getElementById('chartTemp').getContext('2d'),'Temperatura (°C)');
    const chartHum=makeChart(document.getElementById('chartHum').getContext('2d'),'Umidade (%)');
    const buffer={temp:[],hum:[],time:[]},MAX_POINTS=40;
    let ws=null,sim=false,running=true,simTimer=null;

    function appendLog(t){const time=new Date().toLocaleTimeString();logEl.innerHTML=`[${time}] ${t}\\n`+logEl.innerHTML;}
    function updateCharts(ts,t,h){if(buffer.time.length>=MAX_POINTS){buffer.time.shift();buffer.temp.shift();buffer.hum.shift();}
      buffer.time.push(new Date(ts).toLocaleTimeString());buffer.temp.push(t);buffer.hum.push(h);
      chartTemp.data.labels=buffer.time;chartTemp.data.datasets[0].data=buffer.temp;chartTemp.update();
      chartHum.data.labels=buffer.time;chartHum.data.datasets[0].data=buffer.hum;chartHum.update();}
    function setStatus(el,ok,txt){el.className=ok?'status-ok':'status-err';el.textContent=txt;}
    function handleMessage(d){const ts=d.timestamp||Date.now();if(typeof d.temp==='number')tempEl.textContent=d.temp.toFixed(1)+' °C';
      if(typeof d.hum==='number')humEl.textContent=Math.round(d.hum)+' %';
      if(d.gps){latEl.textContent=d.gps.lat.toFixed(5);lonEl.textContent=d.gps.lon.toFixed(5);speedEl.textContent=d.gps.speed!=null?d.gps.speed+' km/h':'--';
        marker.setLatLng([d.gps.lat,d.gps.lon]);map.setView([d.gps.lat,d.gps.lon],map.getZoom());}
      if(typeof d.smoke==='boolean')setStatus(smokeEl,!d.smoke,d.smoke?'ALERTA':'OK');
      if(typeof d.tilt==='number')setStatus(tiltEl,Math.abs(d.tilt)<=5,d.tilt+'°');
      if(typeof d.doorOpen==='boolean')setStatus(doorEl,!d.doorOpen,d.doorOpen?'Aberta':'Fechada');
      updateCharts(ts,d.temp??null,d.hum??null);appendLog(JSON.stringify(d));}
    function startSim(){stopSim();sim=true;
      simTimer=setInterval(()=>{if(!running)return;const now=Date.now();
        const payload={deviceId:'SIM-CTNR',timestamp:now,temp:+(Math.random()*20-5).toFixed(2),hum:+(30+Math.random()*50).toFixed(1),
          smoke:Math.random()<0.02,tilt:+(Math.random()*10-5).toFixed(2),doorOpen:Math.random()<0.03,
          gps:{lat:-22.72+Math.random()*0.02,lon:-43.20+Math.random()*0.02,speed:+(Math.random()*50).toFixed(1)}};
        handleMessage(payload);},Number(intervalInput.value)||2000);
      appendLog('Simulação iniciada');document.getElementById('btn-sim').textContent='Simulando ✅';}
    function stopSim(){if(simTimer)clearInterval(simTimer);simTimer=null;sim=false;document.getElementById('btn-sim').textContent='Simular dados';appendLog('Simulação parada');}
    function connectWS(url){if(ws)ws.close();try{ws=new WebSocket(url);
        ws.onopen=()=>{wsStatusEl.textContent='Conectado';appendLog('WS conectado');};
        ws.onmessage=e=>{try{const o=JSON.parse(e.data);handleMessage(o);}catch(err){appendLog('Erro ao parsear mensagem WS: '+err.message);}};
        ws.onclose=()=>{wsStatusEl.textContent='Desconectado';appendLog('WS desconectado');};
        ws.onerror=()=>{appendLog('Erro WS');};}catch(err){appendLog('Falha ao conectar: '+err.message);}}
    document.getElementById('btn-sim').addEventListener('click',()=>{if(simTimer)stopSim();else startSim();});
    document.getElementById('btn-ws').addEventListener('click',()=>{const url=wsUrlEl.value.trim();if(!url){alert('Informe o endpoint WebSocket');return;}connectWS(url);});
    document.getElementById('btn-pause').addEventListener('click',()=>{running=!running;document.getElementById('btn-pause').textContent=running?'Pausar':'Retomar';appendLog(running?'Retomado':'Pausado');});
    document.getElementById('btn-clear').addEventListener('click',()=>{logEl.innerHTML='';});
    document.getElementById('btn-export').addEventListener('click',()=>{let csv='time,temp,hum\\n';
      for(let i=0;i<buffer.time.length;i++){csv+=`\"${buffer.time[i]}\",${buffer.temp[i]??''},${buffer.hum[i]??''}\\n`;}
      const blob=new Blob([csv],{type:'text/csv'});const url=URL.createObjectURL(blob);const a=document.createElement('a');
      a.href=url;a.download='dados.csv';a.click();URL.revokeObjectURL(url);});
    intervalInput.addEventListener('change',()=>{if(simTimer){startSim();}});
    startSim();
    window.__dashboard={connectWS,handleMessage};
  </script>
</body>
</html>
