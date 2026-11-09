# Volume Profile, Auction aur Delta: EA Feature Plan

**Project:** ai-scalping-system (MT5 EA + Python backend)
**Feature:** Volume Profile, Price Auction (Market Profile logic), Delta / Order Flow, Real Orders (DOM), aur Order Management
**Status:** Requirements draft. Process ke mutabiq code tab shuru hoga jab ye spec approve ho jaye.
**Tareekh:** 9 Nov 2026
**Times:** sab PKT (UTC+5) me

---

## 0. Short summary

| Hissa | Ye kya karega |
|---|---|
| Data layer | MT5 se har tick (bid, ask, time in ms, flags, volume) record karega |
| Volume engine | Har tick ko **buy** ya **sell** me classify karega |
| Volume profile | Har session ka POC, VAH, VAL, HVN, LVN banayega |
| Delta engine | Bar delta, price-level delta (footprint), cumulative delta, imbalance, absorption |
| Auction engine | Market balance me hai ya trend me; initiative vs responsive; acceptance/rejection |
| Order book (DOM) | Jahan broker deta hai, wahan pending buy/sell orders ki walls |
| Signal integration | SMC setup (sweep → CHoCH → FVG/OB) ko volume aur delta se confirm/filter karna |
| Order management | Entry, SL, TP, size, risk limits, spread aur news filter, manual/auto mode |

**Zaroori sach:** OANDA aur zyada tar forex/CFD brokers par **asli volume nahi hota**, sirf tick volume hota hai. Asli buy/sell order flow sirf **exchange futures** (COMEX gold, GC/MGC) par milta hai. Is liye system **dono modes** support karega (section 2).

---

## 1. Kya TradingView zaroori hai?

**EA ke liye TradingView bilkul zaroori nahi.**

| Sawal | Jawab |
|---|---|
| Tick data kahan se aayega? | **MT5 se seedha.** `OnTick()` har tick par chalta hai, aur `CopyTicksRange()` history deta hai. Ye bilkul free hai, broker ka data hai. |
| TradingView paid plan chahiye? | **Nahi.** TradingView ka free plan quote 1 per second deta hai. MT5 har tick deta hai, jo is se behtar hai. |
| TradingView ka role kya rahega? | Sirf chart par dekhne ke liye: Pine indicator "VP Buy/Sell Profile" aur session boxes. Ye optional hai. |
| Mac par abhi wala "tareeqa 2" (TradingView quote recorder)? | Free plan par chal sakta hai (1 update per second). Lekin EA bante hi MT5 ka tick data is se behtar hoga, is liye ye sirf temporary research ke liye hai. |
| Asli volume ke liye kya chahiye? | Futures broker ka MT5 account (CME data ke saath), ya CME data subscription. Ye **paid** hai (section 2.2). |

---

## 2. Data ke do modes

### 2.1 Mode A: CFD / Spot (OANDA, Exness, IC Markets waghera)
- **Volume:** sirf **tick volume** milta hai, yani kitni baar quote badla. Asli lots ya contracts nahi.
- **Trade prints nahi hote:** sirf bid/ask quotes milte hain. Kis ne market order lagaya, ye nahi pata chalta.
- **Buy/sell ka andaza:** quote ki direction se lagta hai (section 3.2).
- **DOM:** aksar nahi milta. Jo milta hai wo broker ki apni liquidity hoti hai, market ki nahi.
- **Faida:** yehi broker hai jahan aap trade karte hain, to SL/TP aur spread asli hain.

### 2.2 Mode B: Exchange futures (COMEX GC / MGC, CME)
- **Volume:** asli contracts.
- **Aggressor flag:** MT5 tick me `TICK_FLAG_BUY` / `TICK_FLAG_SELL` hota hai. Ye batata hai ke market order buyer ka tha ya seller ka.
- **DOM:** asli pending orders (Level 2) milte hain, `MarketBookAdd` aur `OnBookEvent` se.
- **Kya chahiye:**
  - Futures broker jo MT5 deta ho, aur CME real-time data ki fees.
  - **MGC (micro gold, 10 oz)** ka 1 contract taqreeban 0.1 lot XAUUSD ke barabar hai, jo aap ka current size hai.

### 2.3 Recommended setup
- **Phase 1 (free):** Mode A, aap ke apne broker ke ticks par. Is se system banega aur test hoga.
- **Phase 2 (paid, optional):** Mode B ka data sirf **signal** ke liye. Ya to trade bhi futures par ho, ya CFD par.
  - CFD aur futures me thora price farq (basis) hota hai.
  - Is liye levels ko **offset** se map karna hoga: futures price − spot price, har minute update.

---

## 3. Data layer aur tick classification

### 3.1 Tick capture (MQL5)
Har tick par ye fields store hongi:

| Field | Source | Note |
|---|---|---|
| `time_msc` | `MqlTick.time_msc` | Milliseconds |
| `bid`, `ask` | `MqlTick` | Spread = ask − bid |
| `last` | `MqlTick.last` | Sirf exchange par; CFD par 0 |
| `volume_real` | `MqlTick.volume_real` | Sirf exchange par |
| `flags` | `MqlTick.flags` | BID/ASK/LAST/BUY/SELL changes |

**Storage:**
- **Raw ticks:** Parquet files (din ke hisab se), ya DuckDB.
- **Bars aur profiles:** inhi ticks se banenge, broker ke bars se nahi.

**Live:** EA har tick ya chhote batch (100 ms) me backend ko bhejega. Raaste do hain: ZeroMQ/socket, ya MT5 Python package. Python package sirf Windows par chalta hai.

### 3.2 Buy/sell classification (preference order)
1. **Aggressor flag (Mode B):** `TICK_FLAG_BUY` = buy, `TICK_FLAG_SELL` = sell. 100% sahi.
2. **Quote rule (Lee-Ready):** trade price ask par ya upar ho to buy. Bid par ya neeche ho to sell. Beech me ho to tick rule lagega.
3. **Tick rule (Mode A):**
   - Mid price (bid+ask)/2 pichle tick se upar gaya to **buy tick**, neeche gaya to **sell tick**.
   - Barabar raha to pichli direction carry hogi.
   - Volume weight: Mode A me 1 per tick, Mode B me `volume_real`.
4. **Close-location (sab se kamzor):** sirf jab ticks na hon, jaise purana bar data. Ye wohi tareeqa hai jo abhi Pine me hai.

**Test:** Mode B data par tick rule ka result aggressor flag se compare hoga. Is se pata chalega ke Mode A ka andaza kitna sahi hai. Target: 70%+ agreement.

---

## 4. Volume profile engine

### 4.1 Profile types
| Profile | Range (PKT) | Note |
|---|---|---|
| Day | 04:00 → agle din 03:00 | Market open se close |
| Asia | 05:00–13:00 | |
| London | 13:00–18:00 | NY open par freeze hota hai (aap ka rule) |
| NY | 18:00–03:00 | |
| Opening range | 18:00–18:30 | |
| Composite | Pichle 5 ya 20 din | Bari support/resistance |
| Developing | Session ke andar live | Har tick par update |

### 4.2 Calculation
- **Bin size:** default $0.50. Options: $0.10, $0.25 ya $1.00. Har tick apne price ke bin me jayega, is liye exact hai, bars wala andaza nahi.
- **POC:** sab se zyada volume wala bin. Barabari ho to session ke mid ke qareeb wala.
- **Value Area (70%):** POC se shuru kar ke upar aur neeche wale do-do bins compare karo. Jo zyada ho use add karo, jab tak 70% na ho jaye. Ye standard CBOT method hai.
- **HVN/LVN:**
  1. Profile ko smooth karo (moving average, 5 bins).
  2. Local maxima HVN hain aur local minima LVN.
  3. Threshold: LVN = median volume ka 30% se kam.
- **Naked POC:** pichle din ka POC jise price ne dobara touch nahi kiya. Ye magnet level hai, aur list tab tak rahegi jab tak test na ho.
- **Single prints / thin zones:** ek se zyada bins jahan volume bohat kam ho. Inhe fill hone ka imkaan hota hai.

### 4.3 Output (har update par)
`{session, poc, vah, val, hvn[], lvn[], naked_pocs[], total_vol, buy_vol, sell_vol, delta}`

---

## 5. Delta engine

| Metric | Matlab | Use |
|---|---|---|
| **Bar delta** | buy − sell (har 1m/5m bar) | Candle ke andar kaun jeeta |
| **Cumulative delta (CVD)** | Session ke shuru se delta ka jor | Trend ki taqat |
| **Footprint** | Har bar ke har price par buy × sell | Exact jagah jahan fight hui |
| **Imbalance** | Mode B: diagonal (ask[p] vs bid[p−1]) ≥ 3:1. Mode A: same price ≥ 2:1 | Aggressive side |
| **Stacked imbalance** | 3+ imbalances upar-neeche lagatar | Mazboot support/resistance zone |
| **Absorption** | Bohat zyada volume aur delta, lekin price nahi hila (range < 0.3 × ATR) | Limit orders ne market orders ko roka. Reversal ka ishara. |
| **Exhaustion** | High ya low par volume bohat kam | Move khatam hone ka ishara |
| **Delta divergence** | Price naya high banaye lekin CVD nahi | Kamzor move. Sweep ke saath strong signal. |

**Misaal (9 Nov, 19:30 PKT):**
- 5m candle 4130.35 tak gayi (NY high sweep) aur 4124.16 par close hui.
- Tick volume 6,905 tha, NY average 3,300 ka taqreeban 2 guna. Buy share sirf 23% tha.
- Matlab: VAH ke paas high-volume selling, yani sell-side rejection. Agli candle me price 4113.17 tak gaya.

---

## 6. Auction engine (Market Profile logic)

### 6.1 State machine
| State | Rule | Matlab |
|---|---|---|
| **Balance** | Price VAL aur VAH ke beech | Range. Kinaron par fade karo. |
| **Initiative up** | VAH ke upar acceptance + delta positive | Trend up. Sell setups band. |
| **Responsive sell** | VAH ke upar gaya lekin delta negative aur wapas andar | Fake breakout. Sell setup. |
| **Initiative down** | VAL ke neeche acceptance + delta negative | Trend down. Buy setups band. |
| **Responsive buy** | VAL ke neeche gaya lekin delta positive aur wapas andar | Fake breakdown. Buy setup. |

### 6.2 Rules
- **Acceptance:** value ke bahar 2 lagatar 5m body close, ya 30 min bahar rehna.
- **Rejection:** bahar wick lekin body andar close, ya acceptance 15 min me fail ho jaye.
- **80% rule:** price value ke bahar khule (ya bahar ja kar) wapas value me aaye, aur 2 bars andar tike. Tab 80% imkaan hai ke price doosre kinare tak jaye.
- **Poor high/low:** high ya low par "excess" (lamba wick + volume spike) nahi. Ye unfinished auction hai, aur price wapas aa sakta hai.
- **Open type** (London 13:00 aur NY 18:00 par):
  - **Open-Drive:** pehle 15 min ek direction, pullback nahi.
  - **Open-Test-Drive:** pehle ek level test kar ke ulti direction me drive.
  - **Open-Rejection-Reverse:** pehli direction reject ho kar ulta move.
  - **Open-Auction:** range me ghoomna.
  - Har type ka next 2 ghante ka behavior history se test hoga.

---

## 7. Real orders (DOM / order book)

### 7.1 Kya mil sakta hai
| Source | Data | Note |
|---|---|---|
| MT5 `MarketBookGet` (futures broker) | Har price par pending buy/sell lots, Level 2 | Asli exchange orders |
| MT5 `MarketBookGet` (CFD broker) | Aksar khali, ya broker ki apni liquidity | Bharose ke qabil nahi |
| OANDA Order Book / Position Book | Retail traders ke orders ka % har price par, har 20 min ka snapshot | Sirf OANDA clients ka data. Sentiment ke liye kaam ka hai. |

### 7.2 Features (sirf Mode B)
- **Liquidity walls:** jis price par pending lots average ke 3 guna se zyada hon.
- **Pulling/stacking:** price qareeb aane par wall hat jaye (pull) ya barh jaye (stack). Pull matlab fake wall hai, stack matlab asli defense.
- **Iceberg detection:** ek hi price par baar baar fill hone ke bawajood wall khatam na ho.
- **Spoof filter:** jo walls 2 second se kam rahein, unhe ignore karo.

**Note:** DOM ka data bohat tez badalta hai. Ise store karne ke liye zyada disk chahiye, taqreeban 1–2 GB per din.

---

## 8. SMC ke saath integration (signal engine)

Volume/auction system **khud entry nahi dega**. Ye SMC setup ko **confirm** ya **reject** karega.

| SMC event | Volume/auction confirmation | Reject agar |
|---|---|---|
| Liquidity sweep (high) | Sweep candle par volume > 1.5 × avg aur negative delta, ya CVD divergence | Sweep ke baad acceptance (initiative up) |
| Liquidity sweep (low) | Volume spike aur positive delta, ya absorption | Initiative down |
| CHoCH / BOS | Break candle par delta usi direction me | Break par delta ulta |
| FVG/OB entry zone | Zone HVN, VAH/VAL ya stacked imbalance ke saath overlap kare | Zone LVN ke beech ho, jahan price tezi se guzarta hai |
| Target | Agla POC, HVN ya naked POC | |

**ML features** (Research sub-project ke liye): distance to POC/VAH/VAL in ATR, auction state, CVD slope, bar delta z-score, imbalance count, absorption flag, open type, session, spread.

---

## 9. Order management (execution)

**Rule:** sab orders MT5 se jayenge. Auto mode default hai. Manual mode me har order chat me confirm hoga.

### 9.1 Position sizing
- **Risk:** risk per trade fixed dollars me (default $50) ya equity ka %.
- **Lot formula:** lot = risk ÷ (SL distance × contract value). XAUUSD par 1 lot = 100 oz, yani 0.1 lot par $1 move = $10.
- **Broker limits:** lot ko broker ke min, max aur step par round karo.

### 9.2 SL / TP
- **Sell SL:** zone ke high ke upar + **spread** + buffer ($0.30). Sell ka SL **ask** par lagta hai, aur chart bid dikhata hai.
- **Buy SL:** zone ke low ke neeche + buffer.
- **TP:** default 1:3 RR. Agla POC, HVN ya liquidity level agar 3R se pehle ho, to us se pehle partial.
- **Management:**
  - 1R par SL breakeven par.
  - 2R par 50% close (optional).
  - Baqi ka trailing swing ke peeche.

### 9.3 Entry types
- **Limit order:** FVG ya OB zone par. Expiry 30 min.
- **Market order:** CHoCH confirmation ke baad, agar price zone se door na gaya ho (max 0.3R slippage).

### 9.4 Safety filters
| Filter | Default |
|---|---|
| Max spread | $0.50 (zyada ho to entry nahi) |
| News | High-impact news ±15 min koi nayi trade nahi (ForexFactory JSON) |
| Daily loss limit | −2R par us din trading band |
| Max trades per day | 4 |
| Max open trades | 1 |
| Session | London 13:00 se 22:00 PKT tak |
| Kill switch | Chat command ya dashboard button: sab orders cancel aur positions close |
| Slippage log | Har fill ka expected vs actual price record |

---

## 10. Architecture

```
MT5 terminal (Windows)
 ├─ EA (MQL5): tick capture, DOM capture, order execution
 │      ⇅ ZeroMQ / socket (JSON)
Python backend
 ├─ tick store (Parquet / DuckDB)
 ├─ volume engine (classification, profile, delta, footprint)
 ├─ auction engine (state machine)
 ├─ SMC signal engine + ML filter
 ├─ risk & order manager
 └─ API (FastAPI / WebSocket)
        ⇅
React dashboard: profile, footprint, auction ladder, CVD, trades, kill switch
Chat interface: manual mode
```

---

## 11. Use cases

| # | Use case | Expected |
|---|---|---|
| UC1 | London open par trader profile dekhe | Asia ka POC, VAH, VAL aur naked POCs dashboard par |
| UC2 | Price VAH ke upar sweep kare aur negative delta ke saath wapas aaye | "Responsive sell" state + SMC sell setup ka confirmation |
| UC3 | Price VAH ke upar accept ho | "Initiative up". Sell setups block. |
| UC4 | Spread $0.80 ho jaye (news) | Nayi entry block, reason log |
| UC5 | Daily loss −2R | Trading band, Discord par message |
| UC6 | Manual mode me "4127 par sell, SL 4131.2" | Order preview (lot, risk $, RR). Confirm ke baad hi send. |
| UC7 | Futures broker ka DOM available | Liquidity walls dashboard par. Sweep confirmation me use. |

---

## 12. Test cases

| # | Test | Pass criteria |
|---|---|---|
| T1 | Same din ke ticks se POC/VAH/VAL nikalo, TradingView Volume Profile se compare (same bin size) | Farq ≤ 1 bin |
| T2 | Tick rule vs aggressor flag (Mode B data) | ≥ 70% agreement |
| T3 | Value area exactly 70% ± 1% volume rakhe | Pass |
| T4 | London profile 18:00 ke baad freeze ho | 18:00 ke baad ke ticks shamil na hon |
| T5 | Acceptance rule (2 body closes) historic data par | Manual label ke saath ≥ 90% match |
| T6 | Sell SL ask par trigger ho, bid par nahi | Backtest me spread model sahi |
| T7 | Lot sizing: $50 risk, $5 SL | 0.10 lot |
| T8 | Kill switch | 1 second me sab orders cancel aur positions close |
| T9 | Reconnect: EA ya backend restart | Missed ticks history se fill, koi duplicate order nahi |
| T10 | Walk-forward backtest: SMC alone vs SMC + volume filter (6–12 mahine) | Filter win-rate ya expectancy behtar kare. Warna feature off. |

---

## 13. Phases

| Phase | Kaam | Depends on |
|---|---|---|
| P1 | MT5 tick capture EA + storage + replay | — |
| P2 | Volume profile (sessions, POC/VA, HVN/LVN, naked POC) + T1 aur T3 | P1 |
| P3 | Delta engine (tick rule, CVD, imbalance, absorption) + T2 | P1 |
| P4 | Auction state machine + T5 | P2, P3 |
| P5 | SMC integration + backtest T10 | P2–P4, signal engine |
| P6 | Order manager + safety filters + T6–T9 (pehle demo account) | P5 |
| P7 | DOM (agar futures data liya) | P1, broker |
| P8 | Dashboard (profile, footprint, ladder) | P2–P4 |

---

## 14. Open questions (user decide kare)

1. **Broker:** abhi kaun sa MT5 broker hai? Wo tick volume deta hai ya `volume_real` bhi?
2. **Futures data:** CME data (Mode B) lena hai ya sirf CFD (Mode A) se shuru karna hai?
3. **Bin size:** $0.50 theek hai, ya scalping ke liye $0.25?
4. **Sessions:** Day profile 04:00 se shuru ho (market open) ya 05:00 (Asia)?
5. **Risk:** $50 fixed rahe ya equity ka %? Daily loss limit −2R theek hai?
6. **Manual mode:** har trade par confirmation, ya sirf auto-mode ke bahar wali trades par?

---

## 15. Abhi kya maujood hai (TradingView side)

- **Pine indicator:** "VP Buy/Sell Profile". Source: `tradingview-mcp` ke bahar `/Users/mac/Desktop/trading-view/vp_buy_sell_profile.pine`. Ye day profile, POC, VAH, VAL, buy/sell split (close-location) aur auction ladder table banata hai. Ye Mode A ka sab se kamzor version hai, aur EA me tick rule is se behtar hoga.
- **Session sweep monitor:** Discord par sweep, break aur 15m follow-up. Ye EA ke "alerts" hisse ka prototype hai.
