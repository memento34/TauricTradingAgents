# TradingAgents nasıl çalışıyor?

Bu rehber, bu fork'u kurcalayarak öğrenmek isteyenler için yazıldı. Önce büyük resmi, sonra her parçanın hangi dosyada olduğunu, en sonda da ilk çalıştırmayı anlatıyor.

> TradingAgents bir araştırma aracıdır. Ürettiği kararlar yatırım tavsiyesi değildir ve gerçek parayla işlem yapmaz.

## 1. Büyük resim

TradingAgents bir **karar motorudur**. Ona bir sembol ve bir tarih verirsin (`NVDA`, `2026-09-01`), o da gerçek bir yatırım firmasındaki ekipleri taklit eden bir dizi LLM ajanını sırayla çalıştırır ve sonunda **beş kademeli bir not** üretir:

`Buy` (al) → `Overweight` (ağırlık artır) → `Hold` (tut) → `Underweight` (ağırlık azalt) → `Sell` (sat)

Not okunamazsa sonuç `REVIEW` olur. Bu bilerek böyle: okunamayan bir karar "Tut" gibi davranmasın diye.

Akış şöyle:

```
                ┌─ Market (teknik) analisti ─┐
                ├─ Sentiment analisti ───────┤
 START ─────────┼─ News (haber) analisti ────┼──► Boğa araştırmacı ⇄ Ayı araştırmacı
                ├─ Fundamentals analisti ────┤            (tartışma, N tur)
                └─ Memory Log (geçmiş) ──────┘                   │
                                                                 ▼
                                                        Research Manager
                                                       (yatırım planı + not)
                                                                 │
                                                                 ▼
                                                              Trader
                                                       (işlem önerisi)
                                                                 │
                                                                 ▼
                                Agresif ⇄ Muhafazakâr ⇄ Nötr risk analistleri
                                                       (tartışma, N tur)
                                                                 │
                                                                 ▼
                                                       Portfolio Manager
                                                   (son karar + 5 kademeli not)
```

Analistler **aynı anda** çalışır. Hepsi raporunu bitirince tartışma başlar. Bu grafik [LangGraph](https://langchain-ai.github.io/langgraph/) ile kurulmuş: her kutu bir "node", oklar "edge".

## 2. Ajanlar ve dosyaları

| Ajan | Ne yapıyor | Dosya |
|---|---|---|
| Market analisti | Fiyat geçmişi ve göstergeler (RSI, MACD vb.) | `tradingagents/agents/analysts/market_analyst.py` |
| Sentiment analisti | Haber başlıkları, StockTwits, Reddit havası | `tradingagents/agents/analysts/sentiment_analyst.py` |
| News analisti | Şirket haberleri ve makro haberler (FED, enflasyon...) | `tradingagents/agents/analysts/news_analyst.py` |
| Fundamentals analisti | Bilanço, gelir tablosu, nakit akışı (SEC EDGAR) | `tradingagents/agents/analysts/fundamentals_analyst.py` |
| Boğa / Ayı araştırmacı | Analist raporları üzerinden lehte ve aleyhte tartışır | `tradingagents/agents/researchers/` |
| Research Manager | Tartışmayı tartıp yatırım planı yazar | `tradingagents/agents/managers/research_manager.py` |
| Trader | Planı somut bir işlem önerisine çevirir | `tradingagents/agents/trader/trader.py` |
| Risk analistleri (3 kişi) | Aynı öneriyi agresif, muhafazakâr ve nötr gözle eleştirir | `tradingagents/agents/risk_mgmt/` |
| Portfolio Manager | Son sözü söyler, notu verir | `tradingagents/agents/managers/portfolio_manager.py` |

Bir ajanın nasıl "düşündüğünü" anlamanın en hızlı yolu, o dosyadaki **prompt metnini** okumaktır. Davranışı değiştirmek istediğinde ilk deneyeceğin yer de orası.

Grafiğin kendisi (hangi ajan kimden sonra çalışıyor) `tradingagents/graph/setup.py` içinde. Tartışmanın kaç tur süreceğine `tradingagents/graph/conditional_logic.py` karar veriyor.

## 3. İki model katmanı

Ajanlar iki "beyin" kullanır (`tradingagents/default_config.py`):

- **quick_think_llm**: analistler, araştırmacılar, risk analistleri ve trader. Çok sayıda çağrı yapar, hızlı ve ucuz model yeterli.
- **deep_think_llm**: Research Manager ve Portfolio Manager. Az ama önemli çağrı, en iyi model buraya.

İkisi farklı sağlayıcılarda bile olabilir (`quick_think_provider`, `deep_think_provider`).

## 4. Veri nereden geliyor?

`tradingagents/dataflows/` klasörü. Her veri türü için bir "vendor" (sağlayıcı) zinciri var:

| Veri | Varsayılan kaynak | Anahtar gerekir mi? |
|---|---|---|
| Fiyat ve göstergeler | Yahoo Finance | Hayır |
| ABD şirket bilançoları | SEC EDGAR, sonra Yahoo | Hayır |
| Haberler | Yahoo | Hayır |
| Makro (faiz, enflasyon) | FRED | Evet, ücretsiz `FRED_API_KEY` |
| Tahmin piyasaları | Polymarket | Hayır |

Önemli bir tasarım ilkesi: **geçmiş tarihli bir çalıştırma, o tarihte henüz yayınlanmamış veriyi göremez** (point-in-time). Yani 2024 tarihli bir analiz, 2025'te açıklanan bilançoyu kullanamaz. Backtest'in dürüst olması için bu şart.

## 5. Bizim ilgilendiğimiz piyasalar

| Piyasa | Sembol örnekleri | Not |
|---|---|---|
| ABD hisseleri | `AAPL`, `NVDA`, `SPY` | Dört analistin hepsi çalışır |
| Kripto | `BTC-USD`, `ETH-USD` | `asset_type="crypto"`, bilanço analisti çalışmaz |
| Emtia | Altın `GC=F`, gümüş `SI=F`, petrol `CL=F` | Yahoo vadeli kontratları. Bilançosu olmadığı için `market,news` analistlerini seç |

`XAUUSD` gibi broker sembollerini yazsan da Yahoo karşılığına (`GC=F`) kendisi çevirir (`tradingagents/dataflows/symbols.py`).

## 6. Hafıza: kendi hatalarından öğrenmesi

Her çalıştırmanın kararı `~/.tradingagents/memory/trading_memory.md` dosyasına eklenir. Bir sonraki çalıştırmada, tutma süresi (varsayılan 5 işlem günü) dolmuş kararlar **sonuçlandırılır**: gerçek getiri ve benchmark'a (ABD için SPY) göre alfa hesaplanır, kısa bir "ders" yazılır. Portfolio Manager karar verirken bu dersleri okur.

Kod: `tradingagents/memory/` (`log.py`, `settlement.py`, `reflection.py`).

## 7. Backtest: sistem iyi karar veriyor mu?

Tek bir karar şans eseri doğru olabilir. `tradingagents/backtest.py` aynı süreci bir **sembol × tarih ızgarası** üzerinde çalıştırır ve kararları nota göre gruplayıp ortalama alfayı gösterir. "Buy dediklerinde gerçekten piyasayı yeniyor mu?" sorusunun cevabı buradan çıkar.

```bash
tradingagents backtest NVDA,AAPL --start 2026-06-01 --end 2026-08-01 --every 7 --analysts market,news
```

**Sınırı:** Bu bir portföy simülatörü değil. Kaç adet alınacağı, nakit takibi, stop-loss gibi şeyler yok. Her hücre bağımsız. Bizim planımızdaki 3. adım (risk ve portföy katmanı) tam olarak bu boşluğu dolduracak.

## 8. İlk çalıştırma (ChatGPT Plus ile)

```bash
git clone https://github.com/memento34/TauricTradingAgents.git
cd TauricTradingAgents
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install .

tradingagents login chatgpt      # tarayıcıda ChatGPT hesabınla giriş yap
tradingagents                    # menüden "Subscription" sağlayıcısını seç
```

İlk seferde etkileşimli menüyü kullan: "Subscription" seçince planının sunduğu modelleri listeler. Oradan seçtiğin model kimliklerini not al, sonra aynı şeyi soru sormadan tek komutla çalıştırabilirsin:

```bash
export TRADINGAGENTS_LLM_PROVIDER=chatgpt
export TRADINGAGENTS_QUICK_THINK_LLM=<menüde gördüğün hızlı model>
export TRADINGAGENTS_DEEP_THINK_LLM=<menüde gördüğün güçlü model>
export TRADINGAGENTS_OUTPUT_LANGUAGE=Turkish
export TRADINGAGENTS_MAX_DEBATE_ROUNDS=1 TRADINGAGENTS_MAX_RISK_ROUNDS=1
tradingagents --ticker NVDA --date 2026-10-20 --analysts market,news,fundamentals --save
```

İpuçları:
- Abonelik kullanım limitini korumak için önce **az analist** (`market,news`) ve **1 tartışma turu** ile başla.
- `TRADINGAGENTS_OUTPUT_LANGUAGE=Turkish` raporları Türkçe yazar. Ajanlar kendi aralarında yine İngilizce tartışır, çünkü akıl yürütme kalitesi orada daha iyi.
- `--save` ile rapor `~/.tradingagents/logs/` altına kaydedilir. `complete_report.html` dosyasını tarayıcıda açarsan her ajanın raporunu ayrı ayrı okuyabilirsin.
- Adım adım görmek için `notebooks/adim_adim.ipynb` defterini kullan.

## 9. Kurcalamak için önerilen deneyler

1. **Tartışma turunu artır:** `config["max_debate_rounds"] = 3`. Karar değişiyor mu, ne kadar daha uzun sürüyor?
2. **Bir analisti çıkar:** sadece `market` ile çalıştır, sonra `market,news`. Hangi analist kararı en çok etkiliyor?
3. **Prompt değiştir:** `conservative_debator.py` içindeki prompt'u daha temkinli yap ve Portfolio Manager'ın notunun değişip değişmediğine bak.
4. **Portföy ver:** `PortfolioContext` ile elinde pozisyon varmış gibi çalıştır, öneri nasıl değişiyor?
5. **Aynı şeyi iki kez çalıştır:** LLM'ler her seferinde aynı cevabı vermez. Sonucun ne kadar tutarlı olduğunu görmek, sisteme ne kadar güvenebileceğini anlamanın ilk adımı.

## 10. Testler

```bash
pip install pytest pytest-subtests
python -m pytest -q
```

Testler internet ya da API anahtarı istemez. Bu fork'ta 1361 test geçiyor (1 test Bedrock eklentisi kurulu olmadığı için atlanıyor). Kodda bir şey değiştirdiğinde önce testleri çalıştır.
