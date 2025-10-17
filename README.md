<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>OSRS Flipping Tool v4 (Fixed)</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body {
      font-family: 'Inter', sans-serif;
      background: linear-gradient(135deg, #f4f4f4, #e9ecef);
      color: #333;
      margin: 0;
      padding: 20px;
    }
    h1 {
      text-align: center;
      color: #b58e27;
      font-weight: 700;
      margin-bottom: 25px;
    }
    .card {
      background: #fff;
      border-radius: 10px;
      box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
      padding: 20px;
      margin-bottom: 20px;
      transition: transform 0.2s;
    }
    .card:hover { transform: translateY(-2px); }
    button {
      background: linear-gradient(90deg, #b58e27, #d4af37);
      border: none;
      color: #fff;
      font-weight: 600;
      border-radius: 6px;
      padding: 10px 16px;
      margin: 5px;
      transition: all 0.3s ease;
      cursor: pointer;
    }
    button:hover {
      transform: translateY(-1px);
      box-shadow: 0 3px 6px rgba(0, 0, 0, 0.2);
    }
    input[type="number"], input[type="text"] {
      border: 1px solid #ccc;
      border-radius: 6px;
      padding: 8px;
      margin-top: 5px;
      margin-bottom: 10px;
      width: 100%;
    }
    #suggestion-result img {
      border-radius: 6px;
      border: 2px solid #d4af37;
      margin-bottom: 10px;
    }
    .volume-badge {
      display: inline-block;
      margin-top: 8px;
      padding: 4px 8px;
      border-radius: 5px;
      font-size: 0.85em;
      font-weight: 600;
    }
    .high { background-color: #e0f5e9; color: #267a3d; }
    .medium { background-color: #fff4ce; color: #856404; }
    .low { background-color: #fdecea; color: #842029; }
    .trade-entry {
      display: flex;
      justify-content: space-between;
      padding: 8px 0;
      border-bottom: 1px solid #eee;
    }
    .trade-profit.positive { color: #28a745; font-weight: 600; }
    #success-message {
      position: fixed;
      top: 20px;
      right: 20px;
      background: #d4af37;
      color: #000;
      padding: 12px 20px;
      border-radius: 8px;
      font-weight: 600;
      animation: fadeInOut 3s ease-in-out;
      box-shadow: 0 2px 6px rgba(0,0,0,0.3);
    }
    @keyframes fadeInOut {
      0% { opacity: 0; transform: translateY(-10px); }
      10%, 90% { opacity: 1; transform: translateY(0); }
      100% { opacity: 0; transform: translateY(-10px); }
    }
    .profit-glow {
      animation: glowPulse 1s ease-in-out;
    }
    @keyframes glowPulse {
      0% { color: #333; }
      50% { color: #b58e27; }
      100% { color: #333; }
    }
    @media (max-width: 600px) {
      body { padding: 10px; }
      .card { padding: 15px; }
      button { width: 100%; margin: 6px 0; }
    }
  </style>
</head>
<body>
  <h1>💰 OSRS Flipping Tool v4 💰</h1>

  <div class="card">
    <label for="starting-gp">Starting GP:</label>
    <input id="starting-gp" type="number" placeholder="Enter your starting GP (e.g. 10000000)" />
    <button id="suggest-btn">Suggest Item</button>
    <button id="suggest-again-btn" style="display:none;">Suggest Again</button>
    <div id="loading" style="display:none;">Loading...</div>
    <div id="error-message" style="display:none;color:#d9534f;font-weight:600;margin-top:8px;"></div>
  </div>

  <div id="suggestion-result" class="card" style="display:none;">
    <h2 style="color:#b58e27;">Suggested Item</h2>
    <img id="item-icon" src="" alt="" width="64" height="64">
    <div><strong>Name:</strong> <span id="item-name"></span></div>
    <div><strong>Buy Price:</strong> <span id="buy-price"></span></div>
    <div><strong>Sell Price:</strong> <span id="sell-price"></span></div>
    <div><strong>Profit per Item:</strong> <span id="profit-per-item"></span></div>
    <div><strong>ROI:</strong> <span id="roi"></span></div>
    <div><strong>Quantity:</strong> <span id="quantity"></span></div>
    <div><strong>Total Profit:</strong> <span id="total-profit"></span></div>
    <div><strong>GP Used:</strong> <span id="gp-used"></span></div>
    <div id="item-volume" class="volume-badge"></div>
  </div>

  <div class="card">
    <h2 style="color:#b58e27;">Log Trade</h2>
    <form id="trade-form">
      <label>Item: <input id="trade-item-name" type="text" required></label><br><br>
      <label>Buy Price: <input id="trade-buy-price" type="number" required></label><br><br>
      <label>Sell Price: <input id="trade-sell-price" type="number" required></label><br><br>
      <label>Quantity: <input id="trade-quantity" type="number" required></label><br><br>
      <button type="submit">Log Trade</button>
    </form>
  </div>

  <div class="card">
    <h2 style="color:#b58e27;">Session Stats</h2>
    <div>Total Profit: <span id="total-session-profit">0 GP</span></div>
    <div>Total Trades: <span id="total-trades">0</span></div><br>
    <button id="reset-session">Reset Session</button>
  </div>

  <div class="card">
    <h2 style="color:#b58e27;">Profit History</h2>
    <canvas id="profitChart" height="100"></canvas>
  </div>

  <div id="trades-list" class="card"></div>

  <script>
  class OSRSFlippingTool {
    constructor() {
      this.apiBase = 'https://prices.runescape.wiki/api/v1/osrs';
      this.trades = [];
      this.chart = null;
      this.itemLookup = {};
      this.initializeApp();
    }

    async initializeApp() {
      this.bindEvents();
      this.loadTradesFromLocal();
      this.updateTradeStats();
      this.initializeChart();
      await this.loadItemMapping();
    }

    bindEvents() {
      document.getElementById('suggest-btn').addEventListener('click', () => this.suggestItem());
      document.getElementById('suggest-again-btn').addEventListener('click', () => this.suggestItem());
      document.getElementById('trade-form').addEventListener('submit', e => this.logTrade(e));
      document.getElementById('reset-session').addEventListener('click', () => this.resetSession());
    }

    async loadItemMapping() {
      const res = await fetch(`${this.apiBase}/mapping`);
      const data = await res.json();
      this.itemLookup = {};
      for (const item of data) {
        if (item.note || item.placeholder || item.duplicate) continue;
        this.itemLookup[item.id] = item;
      }
    }

    async loadLatestPrices() {
      const res = await fetch(`${this.apiBase}/latest`);
      const data = await res.json();
      this.latestPrices = data.data;
    }

    async loadVolumeData() {
      const res = await fetch(`${this.apiBase}/volumes`);
      const data = await res.json();
      this.volumeData = data.data;
    }

    async loadTrendData() {
      const res = await fetch(`${this.apiBase}/1h`);
      const data = await res.json();
      this.trendData = data.data;
    }

    async suggestItem() {
      const gpInput = document.getElementById('starting-gp');
      const startingGP = parseInt(gpInput.value) || 0;
      const error = document.getElementById('error-message');
      if (startingGP <= 0) {
        error.textContent = "Please enter your starting GP first.";
        error.style.display = 'block';
        return;
      }
      document.getElementById('loading').style.display = 'block';
      try {
        await Promise.all([this.loadLatestPrices(), this.loadVolumeData(), this.loadTrendData()]);
        const suggestion = this.findProfitableItem(startingGP);
        if (!suggestion) throw new Error("No profitable items found right now. Try again in 30s.");
        this.displaySuggestion(suggestion, startingGP);
        document.getElementById('suggestion-result').style.display = 'block';
        document.getElementById('suggest-again-btn').style.display = 'inline-block';
      } catch (e) {
        error.textContent = e.message;
        error.style.display = 'block';
      } finally {
        document.getElementById('loading').style.display = 'none';
      }
    }

    findProfitableItem(startingGP) {
      const cands = [];
      for (const [id, price] of Object.entries(this.latestPrices)) {
        const vol = this.volumeData[id];
        const info = this.itemLookup[id];
        if (!info || !price || !vol) continue;

        let buy = price.low ?? price.avgLowPrice ?? (price.high ? Math.floor(price.high * 0.95) : null);
        let sell = price.high ?? price.avgHighPrice ?? (price.low ? Math.floor(price.low * 1.05) : null);
        if (!buy || !sell || buy <= 0 || sell <= buy) continue;

        const profit = Math.floor(sell * 0.98) - buy;
        const roi = (profit / buy) * 100;
        if (profit < 2 || roi < 0.5) continue;

        const trend = this.trendData[id];
        const trendBoost = trend && trend.avgHighPrice && trend.avgHighPrice > buy ? 1.1 : 1.0;

        const recQty = this.calcQty(buy, vol, startingGP);
        const totalCost = recQty * buy;
        if (totalCost > startingGP) continue;

        const score = profit * trendBoost + roi + Math.min(vol / 10000, 50);
        cands.push({ id, name: info.name, buy, sell, profit, roi, vol, qty: recQty, cost: totalCost, score });
      }

      if (!cands.length) return null;

      const hi = cands.filter(c => c.vol >= 100000);
      const lo = cands.filter(c => c.vol < 100000);
      const useHi = Math.random() < 0.8 && hi.length;
      const pool = useHi ? hi : (lo.length ? lo : cands);

      const top = pool.sort((a, b) => b.score - a.score).slice(0, 50);
      return top[Math.floor(Math.random() * top.length)];
    }

    calcQty(price, vol, gp) {
      const maxByGP = Math.floor(gp / price);
      const volFactor = vol > 1e6 ? 0.002 : vol > 5e4 ? 0.001 : 0.0005;
      return Math.max(1, Math.min(Math.floor(Math.min(maxByGP, vol * volFactor)), 2000));
    }

    displaySuggestion(s, gp) {
      document.getElementById('item-name').textContent = s.name;
      document.getElementById('buy-price').textContent = `${s.buy.toLocaleString()} GP`;
      document.getElementById('sell-price').textContent = `${s.sell.toLocaleString()} GP`;
      document.getElementById('profit-per-item').textContent = `${s.profit.toLocaleString()} GP`;
      document.getElementById('roi').textContent = `${s.roi.toFixed(1)}%`;
      document.getElementById('quantity').textContent = s.qty.toLocaleString();
      document.getElementById('total-profit').textContent = `${(s.profit * s.qty).toLocaleString()} GP`;
      document.getElementById('gp-used').textContent = `${s.cost.toLocaleString()} GP of ${gp.toLocaleString()} GP`;

      const volBadge = document.getElementById('item-volume');
      volBadge.textContent = `${(s.vol/1000).toFixed(0)}K volume`;
      volBadge.className = 'volume-badge ' + (s.vol >= 100000 ? 'high' : s.vol >= 10000 ? 'medium' : 'low');

      document.getElementById('item-icon').src = `https://secure.runescape.com/m=itemdb_oldschool/obj_big.gif?id=${s.id}`;
      document.getElementById('trade-item-name').value = s.name;
      document.getElementById('trade-buy-price').value = s.buy;
      document.getElementById('trade-sell-price').value = s.sell;
      document.getElementById('trade-quantity').value = s.qty;
    }

    logTrade(e) {
      e.preventDefault();
      const i = document.getElementById('trade-item-name').value;
      const b = parseInt(document.getElementById('trade-buy-price').value);
      const s = parseInt(document.getElementById('trade-sell-price').value);
      const q = parseInt(document.getElementById('trade-quantity').value);
      const p = Math.floor(s * 0.98) - b;
      const total = p * q;
      this.trades.push({ name: i, total });
      this.saveTrades();
      this.updateTradeStats(true);
      this.renderTrades();
      e.target.reset();
    }

    saveTrades() {
      localStorage.setItem('osrs_trades', JSON.stringify(this.trades));
    }

    loadTradesFromLocal() {
      this.trades = JSON.parse(localStorage.getItem('osrs_trades') || '[]');
      this.renderTrades();
    }

    resetSession() {
      localStorage.removeItem('osrs_trades');
      this.trades = [];
      this.renderTrades();
      this.updateTradeStats();
    }

    renderTrades() {
      const list = document.getElementById('trades-list');
      if (!this.trades.length) {
        list.innerHTML = '<p style="color:#777;">No trades logged yet.</p>';
        return;
      }
      list.innerHTML = this.trades
        .map(t => `<div class="trade-entry"><div>${t.name}</div><div class="trade-profit positive">+${t.total.toLocaleString()} GP</div></div>`)
        .join('');
    }

    updateTradeStats(animate=false) {
      const total = this.trades.reduce((a,t)=>a+t.total,0);
      const el=document.getElementById('total-session-profit');
      el.textContent=`${total.toLocaleString()} GP`;
      if(animate){el.classList.add('profit-glow');setTimeout(()=>el.classList.remove('profit-glow'),1000);}
      document.getElementById('total-trades').textContent=this.trades.length;
      this.updateChart(total);
    }

    initializeChart() {
      const ctx = document.getElementById('profitChart').getContext('2d');
      this.chart = new Chart(ctx, {
        type: 'line',
        data: { labels: [], datasets: [{ label:'Total Profit (GP)', data: [], borderColor:'#b58e27', tension:0.3, fill:false }] },
        options: { responsive:true, animation:{duration:800}, scales:{x:{title:{display:true,text:'Trade #'}},y:{title:{display:true,text:'Total GP'}}} }
      });
      const saved = JSON.parse(localStorage.getItem('osrs_trades')||'[]');
      let total=0; const profits=[];
      saved.forEach((t,i)=>{total+=t.total; profits.push(total); this.chart.data.labels.push(i+1);});
      this.chart.data.datasets[0].data=profits; this.chart.update();
    }

    updateChart(total){
      const len=this.trades.length;
      this.chart.data.labels.push(len);
      this.chart.data.datasets[0].data.push(total);
      this.chart.update();
    }
  }

  new OSRSFlippingTool();
  </script>
</body>
</html>