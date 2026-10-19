// 比價小幫手 伺服器（Cloudflare Worker）
// 功能：/search?q=商品  → 同時搜尋多個平台，回傳價格列表
//       /identify (POST) → 用 AI 認出照片裡的商品（需要設定 ANTHROPIC_KEY）
// 平台搜尋網址參考自 MIT 授權的開源專案 coseto6125/mcp-taiwan-price-compare

const UA = 'Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.0 Mobile/15E148 Safari/604.1';
const CORS = {
  'access-control-allow-origin': '*',
  'access-control-allow-methods': 'GET,POST,OPTIONS',
  'access-control-allow-headers': 'content-type',
};
const json = (d, s = 200, extra = {}) =>
  new Response(JSON.stringify(d), { status: s, headers: { 'content-type': 'application/json; charset=utf-8', ...CORS, ...extra } });

// 在任意物件裡找第一個像圖片網址的字串
function findImg(o, depth = 0) {
  if (!o || depth > 3) return '';
  if (typeof o === 'string') return /\.(jpe?g|png|webp)(\?|$)/i.test(o) ? o : '';
  if (Array.isArray(o)) { for (const x of o) { const r = findImg(x, depth + 1); if (r) return r; } return ''; }
  if (typeof o === 'object') {
    const keys = Object.keys(o).sort((a, b) => (/img|image|pic|photo/i.test(b) ? 1 : 0) - (/img|image|pic|photo/i.test(a) ? 1 : 0));
    for (const k of keys) { const r = findImg(o[k], depth + 1); if (r) return r; }
  }
  return '';
}
const abs = (u, base) => !u ? '' : u.startsWith('//') ? 'https:' + u : u.startsWith('http') ? u : base + (u.startsWith('/') ? '' : '/') + u;
const num = v => { const n = parseFloat(String(v ?? '').replace(/[^\d.]/g, '')); return n > 0 ? n : 0; };

async function withTimeout(p, ms = 9000) {
  let t; const to = new Promise((_, rej) => (t = setTimeout(() => rej(new Error('timeout')), ms)));
  try { return await Promise.race([p, to]); } finally { clearTimeout(t); }
}

const SOURCES = {
  async pchome(q) {
    const r = await fetch(`https://ecshweb.pchome.com.tw/search/v3.3/all/results?q=${encodeURIComponent(q)}&page=1&sort=rnk/dc`, { headers: { 'user-agent': UA, accept: 'application/json' } });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const d = await r.json();
    return (d.prods || []).filter(p => !String(p.name).includes('【加價購】')).map(p => ({
      title: p.name, price: num(p.price), url: `https://24h.pchome.com.tw/prod/${p.Id}`,
      img: p.picB ? 'https://cs-a.ecimg.tw' + p.picB : '',
    }));
  },
  async momo(q) {
    const body = { host: 'ecmobile', flag: 'searchEngine', data: { searchValue: q, curPage: 1, maxPage: 30, cateLevel: -1, serviceCode: 'MT01', platform: 16, has3P: 'Y', NAM: 'N', china: 'N', cp: 'N', first: 'N', freeze: 'N', prefere: 'N', stockYN: 'N', superstore: 'N', threeHours: 'N', tomorrow: 'N', tvshop: 'N', video: 'N', cycle: 'N', cod: 'N', superstorePay: 'N', moCoinFeedback: 'N', superstoreFree: 'N', discount: 'N', isBrandSeriesPage: false, isShowAdShop: false, curRecommendedWordsCnt: 0 } };
    const r = await fetch('https://apisearch.momoshop.com.tw/momoSearchCloud/moec/textSearch', { method: 'POST', headers: { 'content-type': 'application/json', origin: 'https://m.momoshop.com.tw', referer: 'https://m.momoshop.com.tw/', 'user-agent': UA }, body: JSON.stringify(body) });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const d = await r.json();
    if (!d.success) throw new Error('blocked');
    return ((d.rtnSearchData || {}).goodsInfoList || []).filter(g => g.goodsCode && g.goodsName).map(g => ({
      title: g.goodsName, price: num(g.SALE_PRICE ?? g.goodsPrice), url: `https://www.momoshop.com.tw/goods/GoodsDetail.jsp?i_code=${g.goodsCode}`,
      img: abs(findImg(g), 'https://img.momoshop.com.tw'),
    }));
  },
  async pxbox(q) { // 全聯線上購
    const r = await fetch('https://api-pxbox.es.pxmart.com.tw/app/2.0/spu/single_search', { method: 'POST', headers: { 'content-type': 'application/json', origin: 'https://pxbox.es.pxmart.com.tw', referer: 'https://pxbox.es.pxmart.com.tw/', 'user-agent': UA },
      body: JSON.stringify({ search_setting_type: 2, keyword: q, sort_type: 3, sort_order: 1, page_index: 1, page_size: 40, filters: [] }) });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const d = await r.json();
    return ((d.data || {}).product_list || []).filter(p => !p.is_sold_out && !p.is_ad).map(p => ({
      title: p.product_name, price: num(p.sale_price), url: `https://pxbox.es.pxmart.com.tw/product/${p.id}`,
      img: abs(findImg(p), 'https://pxbox.es.pxmart.com.tw'),
    }));
  },
  async yahoo(q) {
    const r = await fetch(`https://tw.buy.yahoo.com/search/product?p=${encodeURIComponent(q)}`, { headers: { 'user-agent': UA, accept: 'text/html' } });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const html = await r.text();
    const a = html.indexOf('<script id="isoredux-data" type="mime/invalid">');
    if (a < 0) throw new Error('no data');
    const b = html.indexOf('</script>', a);
    const d = JSON.parse(html.slice(a + 47, b));
    return (((d.search || {}).ecsearch || {}).hits || []).filter(h => h.ec_item_url).map(h => ({
      title: h.ec_title, price: num(h.ec_price), url: h.ec_item_url, img: abs(h.ec_image || findImg(h), 'https://s.yimg.com'),
    }));
  },
  async costco(q) {
    const r = await fetch(`https://www.costco.com.tw/rest/v2/taiwan/products/search?query=${encodeURIComponent(q)}&fields=FULL&lang=zh_TW&curr=TWD&pageSize=30`, { headers: { 'user-agent': UA, accept: 'application/json', referer: 'https://www.costco.com.tw/search' } });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const d = await r.json();
    return (d.products || []).map(p => ({
      title: p.name, price: num((p.price || {}).value), url: 'https://www.costco.com.tw' + (p.url || ''),
      img: abs(((p.images || [])[0] || {}).url || findImg(p.images), 'https://www.costco.com.tw'),
    }));
  },
  async etmall(q) {
    const r = await fetch(`https://www.etmall.com.tw/Search/Get?Keyword=${encodeURIComponent(q)}&SortType=0&PageSize=30&PageIndex=0`, { headers: { 'user-agent': UA, accept: 'application/json', referer: 'https://www.etmall.com.tw/' } });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const d = await r.json();
    return (((d.SearchProductResult || {}).products) || []).map(p => ({
      title: p.title, price: num(p.finalPrice), url: p.pageLink ? 'https://www.etmall.com.tw' + p.pageLink : `https://www.etmall.com.tw/i/${p.id}`,
      img: abs(p.imageUrl || findImg(p), 'https://media.etmall.com.tw'),
    }));
  },
};

// 標題和關鍵字的相似度（中文逐字比對）
function relevance(title, q) {
  const t = String(title).toLowerCase().replace(/\s+/g, '');
  const words = q.toLowerCase().split(/\s+/).filter(Boolean);
  let hit = 0, all = 0;
  for (const w of words) for (const ch of w) { all++; if (t.includes(ch)) hit++; }
  return all ? hit / all : 0;
}

async function search(q) {
  const names = Object.keys(SOURCES);
  const settled = await Promise.allSettled(names.map(n => withTimeout(SOURCES[n](q))));
  const status = {}; let results = [];
  settled.forEach((s, i) => {
    const n = names[i];
    if (s.status === 'fulfilled') {
      const list = s.value.filter(x => x.title && x.price > 0).map(x => ({ ...x, platform: n, score: relevance(x.title, q) })).filter(x => x.score >= 0.6).slice(0, 25);
      status[n] = { ok: true, count: list.length };
      results.push(...list);
    } else status[n] = { ok: false, error: String(s.reason && s.reason.message || s.reason) };
  });
  results.sort((a, b) => a.price - b.price);
  return { q, time: new Date().toISOString(), status, results };
}

const PROMPT = `你是台灣的購物比價助手。這張照片可能是商品、包裝、貨架價格標籤或網購截圖。找出照片中最主要的一個商品。
只回覆一個 JSON 物件：{"name":"商品名稱（繁體中文，品牌＋品項）","keyword":"拿去網購搜尋最準的關鍵字（品牌＋品名＋容量，例如：瑞穗鮮乳 936ml）","price":照片上看得到的售價數字或null}
照片裡沒有商品時 name 給空字串。`;

async function identify(req, env) {
  if (!env.ANTHROPIC_KEY) return json({ error: 'no_key' }, 400);
  const { image } = await req.json();
  if (!image || image.length > 6_000_000) return json({ error: 'bad_image' }, 400);
  const r = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: { 'x-api-key': env.ANTHROPIC_KEY, 'anthropic-version': '2023-06-01', 'content-type': 'application/json' },
    body: JSON.stringify({ model: env.MODEL || 'claude-haiku-5-5', max_tokens: 300, messages: [{ role: 'user', content: [
      { type: 'image', source: { type: 'base64', media_type: 'image/jpeg', data: image } }, { type: 'text', text: PROMPT }] }] }),
  });
  if (!r.ok) return json({ error: 'ai_' + r.status }, 502);
  const t = ((await r.json()).content || []).map(c => c.text || '').join('');
  const a = t.indexOf('{'), b = t.lastIndexOf('}');
  try { return json(JSON.parse(t.slice(a, b + 1))); } catch { return json({ error: 'invalid_json' }, 502); }
}

export default {
  async fetch(req, env, ctx) {
    if (req.method === 'OPTIONS') return new Response(null, { headers: CORS });
    const url = new URL(req.url);
    if (url.pathname === '/identify' && req.method === 'POST') return identify(req, env);
    if (url.pathname === '/search') {
      const q = (url.searchParams.get('q') || '').trim().slice(0, 60);
      if (!q) return json({ error: 'no_query' }, 400);
      const cache = caches.default, key = new Request(`https://cache.local/search?q=${encodeURIComponent(q)}`);
      const hit = await cache.match(key);
      if (hit) return new Response(hit.body, { headers: { ...Object.fromEntries(hit.headers), ...CORS } });
      const res = json(await search(q), 200, { 'cache-control': 'public, max-age=600' });
      ctx.waitUntil(cache.put(key, res.clone()));
      return res;
    }
    return json({ ok: true, app: '比價小幫手伺服器', ai: !!env.ANTHROPIC_KEY });
  },
};
