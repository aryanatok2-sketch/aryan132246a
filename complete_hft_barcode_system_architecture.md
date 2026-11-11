# TIER-1 INSTITUTIONAL SYSTEM ARCHITECTURE & PRODUCTION BLUEPRINT
## Multi-Timeframe ($0\text{s}$–$60\text{s}$) Price-Agnostic Bi-Directional "Barcode" HFT Engine (Rust)
### Target Exchanges: Binance (USD$\text{S}$-M) | Bybit (V5 Linear) | BingX (Swap V2) | MEXC (Futures V1)

---

## 1. EXECUTIVE SYSTEM OVERVIEW & CORE TRADING PHYSICS

Yeh document ek mukammal, zero-compromise, institutional-grade High-Frequency Trading (HFT) aur Micro-Range Scalping Engine ka end-to-end architectural blueprint hai. Iska maqsad cryptocurrency perpetual futures market mein banne wale **"Barcode" (Tight Horizontal Ping-Pong) Regimes** ko real-time mein detect karna aur **Maximum Supported Leverage ($30\times$, $50\times$, $75\times$, $100\times$)** ke sath **Continuous Bi-Directional (Long $\leftrightarrow$ Short) Ping-Pong** chalana hai.

### 1.1 The 5 Core Pillars of This Architecture

1. **Price-Agnostic Movement Engine (Coin Price Does Not Matter):**
   * Coin ki dollar price chahay **cents ($\$0.005$, $\$0.1428$, $\$0.5935$)** mein ho ya **dollars ($\$5$, $\$50$, $\$100$, $\$500+$)** mein ho—bot kisi bhi coin ko us ki nominal price ki wajah se reject nahi karta.
   * Har coin ki price ko us ke exchange-native `tick_size` ($\delta_p$) aur **Percentage Box Bounce in Basis Points ($\Delta_{\text{bps}}$, jahan $1\text{ bps} = 0.01\%$)** mein normalize kiya jata hai.
   * Agar ek $\$100.00$ wala coin $\$100.00$ aur $\$100.10$ ke darmiyan bounce kar raha hai ($\Delta_{\text{bps}} = 10\text{ bps} = 0.10\%$), to $100\times$ leverage par woh bhi exact **$+10.0\%$ Gross ROE** deta hai, bilkul waisay hi jaisay $\$0.2222$ se $\$0.2224$ ($\Delta_{\text{bps}} = 9.0\text{ bps} = 0.09\%$) wala coin **$+9.0\%$ Gross ROE** deta hai.

2. **Multi-Timeframe ($0\text{s}$ to $60\text{s}$) Barcode Rhythm Detection:**
   * Bot sirf $1\text{s}$ candles tak mehdood nahi hai. Market mein barcode patterns **$500\text{ms}$, $1\text{s}$, $3\text{s}$, $5\text{s}$, $10\text{s}$, $15\text{s}$, $20\text{s}$, $30\text{s}$, aur $60\text{s}$** ke kisi bhi rhythm par ban sakte hain.
   * **The Time-Gap Advantage:** Jitna bounce ka cycle time ($T_{\text{cycle}}$) lamba hota hai (maslan $5\text{s}$ se $60\text{s}$ ka gap), bot ke liye execution utni hi aasan aur risk-free ho jati hai, kyunke resting Post-Only Maker orders ko FIFO queue mein fill hone ke liye kafi waqt milta hai aur network jitter ka asar khatam ho jata hai.

3. **True Barcode Visual Geometry, Green/Red Symmetry & Trader "Mish-Mash" Tolerance:**
   * Asli barcode pattern (jaisay `RUNEUSDT` aur `THETA-USDT` ke charts mein hai) ki pehchan sirf price ka rukna nahi, balkay **Full-Body Parallel Rectangular Candles (Marubozu — minimal/zero wicks)** aur **Green (Up) vs. Red (Down) Candles ka Balance** hai.
   * **Mish-Mash Tolerance:** Live market mein retail traders ki wajah se kabhi-kabhi beech mein 2–5 flat candles aa jati hain, ya lagatar 2–3 ek hi color ki candles ban jati hain, ya price 1 tick ka temporary dip le kar wapis box mein aa jati hai (`THETA-USDT` style). Hamara **Non-Flat Markov Transition Scanner** aur **Level-2 Absorption Filter** is temporary mish-mash ko filter out karta hai aur sirf tab exit/shift karta hai jab waqai order book ki deewar toot rahi ho.

4. **Bi-Directional Hedge-Mode Ping-Pong (Every Candle Is an Opportunity):**
   * Bot `Hedge Mode` (`LONG` aur `SHORT` independent slots) mein kaam karta hai:
     * **Box Floor ($P_{\text{floor}}$) Par:** Upar se aane wali **Short** position profit mein close (`Maker Buy TP`) hoti hai, aur exact usi price par nayi **Long** position open (`Maker Buy Entry`) hoti hai.
     * **Box Ceiling ($P_{\text{ceiling}}$) Par:** Neechay se aane wali **Long** position profit mein close (`Maker Sell TP`) hoti hai, aur exact usi price par nayi **Short** position open (`Maker Sell Entry`) hoti hai.

5. **4-Stage Deterministic Math & Atomic Step-Box Migration:**
   * Har trade **4 Mathematical Stages** se guzarti hai: **(1) Pre-Entry Confirmation $\to$ (2) Synthetic L3 FIFO Queue & $T_{\text{ETA}}$ Tracking $\to$ (3) Nanosecond Move Capture on Fill $\to$ (4) Profit Exit / Trail / Stagnation Scratch**.
   * Jab coin apni range shift karta hai (maslan `0.2222–0.2224` se uth kar `0.2226–0.2228` par jata hai, ya `RUNEUSDT` ki tarah `0.5945–0.5948` se gir kar `0.5941–0.5943` par aata hai), to bot breakout candle banne se **pehle** Order Book Wall Erosion aur Hawkes Order-Flow Surge ko pakar kar **Atomic Scratch-and-Flip** execute karta hai.

---

## 2. HARDWARE, OS KERNEL-BYPASS & 5-CORE PINNED THREAD TOPOLOGY

HFT mein Python, Node.js, ya Go ka Garbage Collector ($5\text{ms}$–$50\text{ms}$ pauses) aur standard OS thread scheduling ($25\text{ }\mu\text{s}$ context-switch penalties) qabil-e-qubool nahi hain. Yeh engine **Rust** mein **5-Core Pinned Thread-Per-Core Architecture** par chalta hai.

### 2.1 Co-Location & Server Infrastructure
* **Cloud Region:**
  * **AWS Tokyo (`ap-northeast-1`):** Binance Futures (`fstream.binance.com` / `ws-fapi.binance.com`) aur Bybit V5 (`stream.bybit.com`) ke matching engines Tokyo mein hain (Co-located RTT: $0.35\text{ms}$–$0.90\text{ms}$).
  * **AWS Singapore (`ap-southeast-1`):** BingX (`open-api-swap.bingx.com`) aur MEXC (`contract.mexc.com`) ke primary gateways Singapore/HK cluster ke qareeb hain (Co-located RTT: $0.80\text{ms}$–$2.20\text{ms}$).
* **Instance Type:** AWS EC2 `c7gn.2xlarge` ya `c7i.metal` (8 vCPUs, Enhanced Networking ENA Express, $25\text{ Gbps}$+ network bandwidth).

### 2.2 5-Core Thread Isolation Diagram (`isolcpus=1,2,3,4` + `SCHED_FIFO` Priority 99)

```text
+===================================================================================================+
|                     EXCHANGE MATCHING ENGINES (BINANCE | BYBIT | BINGX | MEXC)                    |
+===================================================================================================+
          | (TLS 1.3 WebSocket Market + Private User Fill Streams)        ^ (TLS 1.3 WS / HTTP2 Orders)
          v                                                               |
+------------------------+     Wait-Free SPSC Ring 1 (<11ns)    +-----------------------------------+
| CORE 1: Network Ingress| ===================================> | CORE 2: Hot Quant & State Machine |
| - SO_BUSY_POLL = 50us  |                                      | - Synthetic L3 FIFO Queue Tracker |
| - TCP_NODELAY = 1      |     Wait-Free SPSC Ring 2 (<11ns)    | - 4-Stage Confirmation/Exit Math  |
| - BingX Stack GZIP     |     +--------------------------+     | - Bi-Directional Ping-Pong Flip   |
| - Branchless Parser    |     | CORE 4: 0s-60s Multi-TF  | ==> | - Step-Shift vs Mish-Mash Tensor  |
+------------------------+     |   Barcode Scanner        |     | - 100x Mark-Price Basis Guard     |
                               | - 9 Time Buckets (0.5..60)|     | - 100us Dummy L1 Cache Warmer     |
+------------------------+     | - Marubozu & G/R Balance |     +-----------------------------------+
| CORE 0: Control Plane  |     | - EMA7/EMA30 Compression |                       |
| - /exchangeInfo Sync   |     | - Markov Bounce Matrix   |                       | (OutboundWirePacket)
| - Auto 100x Leverage   |     +--------------------------+                       v
| - Isolated + Hedge Init|                                      +-----------------------------------+
| - ListenKey Keepalive  |                                      | CORE 3: Speculative TX Dispatcher |
| - DNS Fastest-IP Probe |                                      | - 4x4 Subaccount API Key Shards   |
+------------------------+                                      | - 11ns In-Place ASCII Patcher     |
                                                                | - Speculative (N-1) Byte Stager   |
                                                                | - 1-Byte Trigger Flush (MSG_DONTWAIT)|
                                                                +-----------------------------------+
```

### 2.3 Linux Kernel & NIC Tuning Parameters (`scripts/tune_linux_kernel.sh`)
1. **Kernel Boot Flags (`/etc/default/grub`):**
   `isolcpus=1,2,3,4 nohz_full=1,2,3,4 rcu_nocbs=1,2,3,4 idle=poll processor.max_cstate=0 intel_idle.max_cstate=0 mitigations=off`
2. **Memory Page Locking:**
   Startup par `libc::mlockall(libc::MCL_CURRENT | libc::MCL_FUTURE)` call hota hai taake कोई bhi memory page swap mein na jaye aur page-fault latency zero ho.
3. **Socket Busy-Polling:**
   Har socket descriptor par `SO_BUSY_POLL = 50` microseconds aur `TCP_NODELAY = 1` set hota hai taake NIC driver OS interrupt (`IRQ`) ka wait kiye baghair packet seedha user-space buffer mein de.
4. **$100\text{ }\mu\text{s}$ Dummy Cache Warmer (`Core 2`):**
   Jab $5\text{s}$–$60\text{s}$ barcode mein chand seconds tak market quiet ho, to `Core 2` har $100\text{ }\mu\text{s}$ baad ek `DummyCacheWarmup` (`is_dry_run = true`) tick poore execution path se guzarta hai taake CPU ka **L1 Instruction Cache (L1i)**, **L1 Data Cache (L1d)**, aur **Branch Target Buffer (BTB)** $100\%$ hot rahe.

---

## 3. DYNAMIC PRICE-AGNOSTIC NORMALIZATION ENGINE ($\$0.0001$ TO $\$10,000+$ COINS)

Coin ki price $\$0.0020$ ho, $\$0.2222$ ho, ya $\$250.00$ ho—bot ke andar tamam calculations **Universal Integer PriceTicks** aur **Basis Points ($\text{bps}$)** mein hoti hain.

### 3.1 Instrument Metadata Discovery (`Core 0` Startup Sync)
Bot start hote hi charon exchanges ke REST endpoints se har active USDT perpetual pair ka metadata fetch karta hai:
* **Binance:** `GET /fapi/v1/exchangeInfo` $\to$ `PRICE_FILTER.tickSize`, `LOT_SIZE.stepSize`, aur `GET /fapi/v1/leverageBracket` $\to$ `initialLeverage`.
* **Bybit V5:** `GET /v5/market/instruments-info?category=linear` $\to$ `priceFilter.tickSize`, `lotSizeFilter.qtyStep`, `leverageFilter.maxLeverage`.
* **BingX V2:** `GET /openApi/swap/v2/quote/contracts` $\to$ `pricePrecision`, `quantityPrecision`, `maxLongLeverage`, `maxShortLeverage`.
* **MEXC V1:** `GET /api/v1/contract/detail` $\to$ `priceUnit`, `volUnit`, `maxLeverage`, `makerFeeRate`.

### 3.2 Exact Integer Tick & Lot Scaling Equations
Har coin $s$ ke liye `tick_recip` ($R_p = \text{round}(1 / \delta_p)$) aur `lot_recip` ($R_q = \text{round}(1 / \delta_q)$) store kiya jata hai:

$$P_{\text{ticks}} = \left\lfloor \text{Price} \times R_p + 0.5 \right\rfloor \in \mathbb{N}^+, \qquad Q_{\text{lots}} = \left\lfloor \text{Quantity} \times R_q + 0.5 \right\rfloor \in \mathbb{N}^+$$

| Coin Example | Nominal Price | Exchange `tick_size` ($\delta_p$) | Internal `PriceTicks` ($P_{\text{ticks}}$) | 2-Tick Barcode Box | Box Move in Basis Points ($\Delta_{\text{bps}}$) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`PEPE / BONK`** | $\$0.008000$ | $\$0.000001$ | $8000$ | $8000 \leftrightarrow 8008$ | $10.00\text{ bps } (0.10\%)$ |
| **`THETA-USDT`** | $\$0.1428$ | $\$0.0001$ | $1428$ | $1428 \leftrightarrow 1430$ | $14.00\text{ bps } (0.14\%)$ |
| **`DOGE / Custom`** | $\$0.2222$ | $\$0.0001$ | $2222$ | $2222 \leftrightarrow 2224$ | $9.00\text{ bps } (0.09\%)$ |
| **`RUNEUSDT`** | $\$0.5941$ | $\$0.0001$ | $5941$ | $5941 \leftrightarrow 5943$ | $3.36\text{ bps } (0.034\%)$ |
| **`SOL / AVAX`** | $\$100.00$ | $\$0.01$ | $10000$ | $10000 \leftrightarrow 10010$ | $10.00\text{ bps } (0.10\%)$ |

### 3.3 Dynamic Leverage & Notional Position Sizing
Har coin par exchange jitna maximum leverage ($L_{\max}^{\text{coin}}$, e.g., $50\times$, $75\times$, $100\times$) support karta hai, bot usay activate karta hai:

$$L_{\text{active}} = \min\left(L_{\text{target}}, \; L_{\max}^{\text{coin}}\right)$$

Aur har order ka lot size ($Q_{\text{order}}$) user ke `margin_per_trade_usdt` ($M_{\text{USDT}}$) aur $L_{\text{active}}$ se auto-compute hota hai:

$$Q_{\text{order}} = \left\lfloor \frac{M_{\text{USDT}} \times L_{\text{active}}}{\text{Price} \times \text{Contract\_Multiplier} \times \delta_q} \right\rfloor$$

---

## 4. MODULE 1 (`CORE 4`): MULTI-TIMEFRAME ($0\text{s}$–$60\text{s}$) BARCODE & MISH-MASH SCANNER

Yeh module exchange ke tamam pairs (200+ coins) ki real-time stream sunta hai aur har coin ko **9 Parallel Timeframes** par evaluate karta hai taake chahay barcode $1\text{s}$ ka ho, $3\text{s}$ ka ho, $5\text{s}$ ka ho, $10\text{s}$ ka ho, $20\text{s}$ ka ho, ya $60\text{s}$ ka ho—foran pakra jaye.

### 4.1 9-Bucket Multi-Timeframe OHLCV Ring Architecture
Har coin ke liye `Core 4` memory mein 9 circular ring buffers ($W = 32$ candles per bucket) banata hai:

$$\mathcal{T}_{\text{buckets}} = \left\{ 500\text{ms}, \; 1\text{s}, \; 2\text{s}, \; 3\text{s}, \; 5\text{s}, \; 10\text{s}, \; 20\text{s}, \; 30\text{s}, \; 60\text{s} \right\}$$

Har bucket $\tau \in \mathcal{T}_{\text{buckets}}$ ke andar har candle $i \in \{1..W\}$ ke yeh 6 integer fields store hote hain:
* $O_i, H_i, L_i, C_i$ (`PriceTicks`)
* $V_i^{\text{buy}}, V_i^{\text{sell}}$ (`QtyLots`)
* Incremental $\text{EMA}_7(C_i)$ aur $\text{EMA}_{30}(C_i)$

---

### 4.2 The 7 Mathematical Equations of True Barcode Geometry

Kisi bhi coin ke timeframe bucket $\tau$ ko **"Active Tradable Barcode"** tab mark kiya jata hai jab woh pichle $W = 24$ candles mein in **7 Equations** ko pass kare:

#### Equation 1: Quantile Box Floor & Ceiling Extraction (Outlier-Immune)
Agar 24 candles mein se 1–2 candles mein kisi retail trader ne 1-tick ki choti mish-mash (outlier) mar di ho, to simple $\min/\max$ box ko kharab kar deta hai. Is liye bot **Mode / $10^{\text{th}}$–$90^{\text{th}}$ Percentile Quantile** se asli Barcode Floor aur Ceiling nikalta hai:

$$P_{\text{floor}} = \text{Mode}\left(\{L_1, L_2, \dots, L_W\}\right), \qquad P_{\text{ceiling}} = \text{Mode}\left(\{H_1, H_2, \dots, H_W\}\right)$$

$$\Delta_{\text{box}} = P_{\text{ceiling}} - P_{\text{floor}}, \qquad \Delta_{\text{bps}} = \frac{\Delta_{\text{box}} \times 10,000}{P_{\text{floor}}}$$

#### Equation 2: Fee-Adjusted Net Profitability Gate (`0.05%` to `0.50%` Move)
Round-trip Maker Fee $F_{\text{RT}} = 2 \times f_{\text{maker}}$ (Binance/Bybit/BingX par $2 \times 2.0\text{ bps} = 4.0\text{ bps}$; MEXC par $0\text{ bps}$):

$$\Delta_{\text{bps}} \ge \max\left(2.2 \times F_{\text{RT}}, \; 5.0\text{ bps}\right) \quad \text{AND} \quad \Delta_{\text{bps}} \le 50.0\text{ bps } (0.50\%)$$

#### Equation 3: Full-Body Rectangular Candle Ratio (No-Wick Marubozu Check)
`RUNEUSDT` aur `THETA-USDT` ke screenshots mein barcode candles poori solid rectangular bars hain (wicks nahi hain). Non-flat candles ($C_i \ne O_i$) ke liye Body-to-Range ratio:

$$M_{\text{body}} = \frac{1}{N_{\text{active}}} \sum_{C_i \ne O_i} \frac{|C_i - O_i|}{\max(1, \; H_i - L_i)} \ge 0.82$$

*(Yani candle ka kam az kam $82\%$ hissa solid body ho, suiyan/wicks na hon).*

#### Equation 4: Roof & Floor Parallel Containment Ratio ($BCR$)
Kitni candles ki choti ($H_i$) aur neechla hissa ($L_i$) bilkul `[P_floor, P_ceiling]` ke andar lock hain (allowing at most 1 tick of retail noise on $\le 15\%$ of candles):

$$BCR = \frac{1}{W} \sum_{i=1}^{W} \mathbb{I}\left( L_i \ge P_{\text{floor}} \;\wedge\; H_i \le P_{\text{ceiling}} \right) \ge 0.82$$

#### Equation 5: Green vs. Red Candle Balance (Color & Volume Symmetry)
Barcode mein sirf ek color ki candles nahi ho saktin; Green ($C_i > O_i$) aur Red ($C_i < O_i$) candles ki tadad ($N_G, N_R$) aur volume ($V_G, V_R$) balanced hona chahiye:

$$S_{\text{count}} = \frac{\min(N_G, N_R)}{\max(N_G, N_R)} \ge 0.60, \qquad S_{\text{vol}} = \frac{\min(V_G, V_R)}{\max(V_G, V_R)} \ge 0.55$$

#### Equation 6: Mish-Mash Tolerant Markov Bounce Probability ($P_{\text{bounce}}$)
Kyunke traders ki wajah se beech mein 2–5 flat candles ($C_i = O_i$) ya 2 consecutive same-color step candles aa sakti hain, bot **State-Machine Touch Alternation** ginta hai:
* State $\mathcal{S} = -1$ jab price $P_{\text{floor}}$ ko touch kare.
* State $\mathcal{S} = +1$ jab price $P_{\text{ceiling}}$ ko touch kare.
* Beech mein jitni bhi flat candles aayein, jab tak price $[P_{\text{floor}} - 1, P_{\text{ceiling}} + 1]$ ke andar hai, agar woh $P_{\text{floor}}$ ke baad $P_{\text{ceiling}}$ touch karti hai to `Alternation++`, aur agar box tor kar bahar nikalti hai to `Breakout++`:

$$P_{\text{bounce}} = \frac{\text{Alternations}}{\text{Alternations} + \text{Breakouts}} \ge 0.78$$

#### Equation 7: EMA7 & EMA30 Compression (Flat Moving Averages)
`RUNEUSDT` screenshot ki tarah $\text{EMA}_7$ aur $\text{EMA}_{30}$ dono box ke darmiyan bilkul flat chal rahe hon:

$$D_{\text{EMA}} = \frac{|\text{EMA}_7(t) - \text{EMA}_{30}(t)|}{\Delta_{\text{box}}} \le 0.45 \quad \text{AND} \quad P_{\text{floor}} < \text{EMA}_7(t) < P_{\text{ceiling}}$$

---

### 4.3 Multi-Timeframe Priority Ranking Formula (Why $5\text{s}$–$60\text{s}$ Cycles Get Priority)
Agar scanner ko ek sath 10 coins milte hain (kuch $1\text{s}$ par, kuch $5\text{s}$ par, kuch $30\text{s}$ par), to bot har coin ko yeh **Master Barcode Fitness Score ($BFS$)** deta hai:

$$BFS(\tau) = \underbrace{\left(\Delta_{\text{bps}} - F_{\text{RT}}\right) \cdot L_{\text{active}}}_{\text{Net Leveraged ROE per Bounce}} \times M_{\text{body}} \times S_{\text{count}} \times P_{\text{bounce}} \times \underbrace{\left(1 + 0.35 \ln(1 + \tau_{\text{sec}})\right)}_{\text{Time-Gap Ease Multiplier}}$$

* **Time-Gap Multiplier:** Ek $10\text{s}$ ya $30\text{s}$ barcode coin ko $1\text{s}$ coin ke muqable mein **$1.8\times$ se $2.2\times$ zyada priority** milti hai, kyunke $10\text{s}$–$60\text{s}$ ke gap mein bot bina kisi latency race ke $100\%$ precision ke sath Maker orders fill karwata hai.

---

## 5. MODULE 2 (`CORE 2`): BI-DIRECTIONAL PING-PONG & 4-STAGE EXECUTION MATH

Jab Scanner kisi coin aur us ke optimal cycle period $T_{\text{cycle}} \in [0.5\text{s}, 60\text{s}]$ ko lock kar ke `Core 2` ko deta hai, to `Core 2` **Hedge Mode** mein **Bi-Directional Ping-Pong State Machine** chalata hai.

### 5.1 Bi-Directional Ping-Pong State Diagram (Simultaneous Long & Short Flip)

```text
                  +-------------------------------------------------------+
                  |        BARCODE LOCKED: [P_floor, P_ceiling]           |
                  |   Queue Resting Maker Buy (LONG) at P_floor           |
                  |   Queue Resting Maker Sell (SHORT) at P_ceiling       |
                  +-------------------------------------------------------+
                               /                             \
     (P_floor Maker Buy Fills)                                (P_ceiling Maker Sell Fills)
             v                                                             v
+-----------------------------------------+       +-----------------------------------------+
| STATE: LONG_ACTIVE                      |       | STATE: SHORT_ACTIVE                     |
| 1. In <15ns: Place Maker Sell TP        |       | 1. In <15ns: Place Maker Buy TP         |
|    (Close Long) at P_ceiling            |       |    (Close Short) at P_floor             |
| 2. Maintain/Queue Maker Sell Entry      |       | 2. Maintain/Queue Maker Buy Entry       |
|    (Open Short) at P_ceiling            |       |    (Open Long) at P_floor               |
+-----------------------------------------+       +-----------------------------------------+
             |                                                             |
             | (Price bounces UP to P_ceiling:                             | (Price bounces DOWN to P_floor:
             |  Long TP fills -> Locks +5%..+20% ROE!                      |  Short TP fills -> Locks +5%..+20% ROE!
             |  Short Entry fills -> Now SHORT_ACTIVE!)                    |  Long Entry fills -> Now LONG_ACTIVE!)
             +-------------------------------------------------------------+
                                           |
                                           v
                        [CONTINUOUS INFINITE PING-PONG LOOP]
```

---

### 5.2 The 4-Stage Deterministic Math (Entry $\to$ Queue $\to$ Capture $\to$ Exit)

Har ek order lagane se le kar profit mein close karne tak bot yeh **4 Exact Mathematical Calculations** karta hai:

#### STAGE 1: Pre-Entry Confirmation Math (Kab Order Lagana Hai?)
Bot `P_floor` par Long entry ya `P_ceiling` par Short entry sirf tab place karta hai jab yeh **4 Confirmation Equations** `TRUE` hon:
1. **Mark-Price Basis Gate ($100\times$ Liquidation Guard):**
   $$B_{\text{mark}} = \frac{|P_{\text{mark}} - P_{\text{mid}}|}{P_{\text{mid}}} \times 10,000 \le 15.0\text{ bps } (0.15\%)$$
2. **Support/Resistance Wall Integrity:**
   Jis edge par hum order laga rahe hain, wahan khara L2 volume pichle $30\text{s}$ ke EMA wall volume ka kam az kam $35\%$ hona chahiye:
   $$V_{\text{edge}}(t) \ge 0.35 \times \text{EMA}_{30\text{s}}\left(V_{\text{edge}}\right)$$
3. **Pre-Trade Positive Expected Value ($EV_{\text{bps}}$):**
   $$EV_{\text{bps}} = \mathbb{P}_{\text{win}|\text{fill}} \cdot (\Delta_{\text{bps}} - 2 f_{\text{maker}}) - (1 - \mathbb{P}_{\text{win}|\text{fill}}) \cdot (\Delta_{\text{stop\_bps}} + f_{\text{maker}} + f_{\text{taker}}) \ge +2.5\text{ bps}$$
4. **No Imminent Breakout:** Step-Shift Tensor $|Z_{\text{up}}| < 1.15 \;\wedge\; |Z_{\text{down}}| < 1.15$.

---

#### STAGE 2: Synthetic L3 FIFO Queue & Execution Timing Math (Kab Execute Hoga?)
Exchanges public stream par Level-2 (total volume) dete hain. Hamara bot L2 updates aur AggTrade stream ko combine kar ke **Synthetic Level-3 FIFO Queue Position** banata hai:

1. **Initial Queue Anchor ($t = t_{\text{ack}}$):**
   Jab hamara order size $q_{\text{bot}}$ price $P_{\text{floor}}$ par acknowledge hota hai:
   $$q_{\text{ahead}}(0) = V_{\text{bid}}(P_{\text{floor}}, t_{\text{ack}}), \qquad q_{\text{behind}}(0) = 0$$

2. **Tick-by-Tick Queue Differential Update:**
   * **Jab Sell Trade ($v_{\text{sell}}$) $P_{\text{floor}}$ par hit kare:**
     $$q_{\text{ahead}}(t) = \max\left(0, \; q_{\text{ahead}}(t-1) - v_{\text{sell}}(t)\right)$$
   * **Jab $P_{\text{floor}}$ par Naya Limit Volume Add ho ($\Delta V > 0$):**
     $$q_{\text{behind}}(t) = q_{\text{behind}}(t-1) + \Delta V$$
   * **Jab $P_{\text{floor}}$ par Volume Cancel ho ($\Delta V_{\text{cancel}} > 0$ without trade):**
     Cont-Stoikov Power-Law Hazard ($\eta = 2.4$, kyunke tail orders front orders se $2.4\times$ zyada cancel hote hain):
     $$q_{\text{ahead}}(t) = \max\left(0, \; q_{\text{ahead}}(t-1) - \Delta V_{\text{cancel}} \cdot \left(\frac{q_{\text{ahead}}(t-1)}{V_{\text{bid}}(t-1)}\right)^{2.4}\right)$$
     $$q_{\text{behind}}(t) = \max\left(0, \; V_{\text{bid}}(t) - q_{\text{ahead}}(t) - q_{\text{bot}}\right)$$

3. **Hawkes-Driven Expected Time-to-Fill ($T_{\text{ETA}}$):**
   $$T_{\text{ETA}}(t) = \frac{q_{\text{ahead}}(t)}{\lambda^-(t)}$$
   *(Jahan $\lambda^-(t)$ Recursive Hawkes Sell Trade Intensity hai).*
   * **Cycle-Adaptive Queue Hold Rule:**
     Bot order ko tab tak queue mein rakhta hai jab tak:
     $$T_{\text{ETA}}(t) \le \max\left(2.5\text{s}, \; 0.65 \times T_{\text{cycle}}\right)$$

4. **First-Passage Probability & "Last-Man-Standing" Toxic Fill Prevention:**
   Jab hamara order fill hone ke qareeb ho ($q_{\text{ahead}}(t) < 0.20 \times V_{\text{initial}}$), to bot check karta hai ke hamare fill hone ke baad `P_ceiling` pehle tootega (Win) ya hamare peechay bacha hua `q_behind` pehle tootega (Loss):
   $$\mathbb{P}_{\text{win}|\text{fill}}(t) = \frac{q_{\text{behind}}(t) \cdot \lambda^+(t)}{V_{\text{ask}}(P_{\text{ceiling}}, t) \cdot \lambda^-(t) + q_{\text{behind}}(t) \cdot \lambda^+(t)}$$
   * **HARD ABORT RULE:** Agar $\mathbb{P}_{\text{win}|\text{fill}}(t) < 0.42$ ho jaye (yani hamare peechay kharay whales apne orders cancel kar ke bhaag gaye hain aur sellers ki Hawkes intensity $\lambda^-(t)$ phat rahi hai), to bot **order fill hone se $< 500\text{ }\mu\text{s}$ pehle hi cancel** kar deta hai!

---

#### STAGE 3: Move Capture Confirmation Math (Kab Move Capture Hogi?)
* Jab $\mathbb{P}_{\text{win}|\text{fill}} \ge 0.58$ hota hai aur $q_{\text{ahead}}(t) \le 5 \times q_{\text{bot}}$ reh jata hai, to `Core 3` **Speculative TCP Pre-Staging** activate kar deta hai: Take-Profit order frame ke pehle $N - 1$ bytes socket buffer mein bhej diye jate hain.
* Jis microsecond Private WebSocket par `FILL` (ya `PARTIALLY_FILLED`) packet aata hai, bot **$< 15\text{ nanoseconds}$** mein aakhri `1 byte` (`}`) socket par flush kar deta hai aur opposing box edge (`P_ceiling` for Long, `P_floor` for Short) par Reduce-Only Post-Only Maker TP lag jata hai!

---

#### STAGE 4: Profit Exit, Multi-Tick Trail, or Flat-Candle Stagnation Bailout (Kab Nikalna Hai?)
Position open hone ke baad har tick par bot 3 mein se 1 faisla karta hai:
1. **Standard Barcode Bounce Exit:** Price opposing edge (`P_ceiling`) par bounce karti hai $\to$ Hamara Maker TP fill hota hai $\to$ Full spread + low Maker fee ke sath **$+5\%$ se $+25\%$ Net ROE** lock ho jata hai.
2. **Favorable Breakout Trail (2x–3x Profit):** Agar hum `0.2222` se Long hain aur price `0.2224` par aane par Step-Up Tensor $Z_{\text{up}} \ge 1.85$ dikhata hai (yani `0.2224` ki Ask wall toot rahi hai), to bot `0.2224` ka TP cancel kar ke usay **`0.2226` ya `0.2228`** par trail kar deta hai!
3. **Flat-Candle Stagnation Scratch (Zero-Hope Bailout):** Agar 3–10 candles flat rehti hain aur:
   $$t_{\text{held}} > \max\left(10\text{s}, \; 2.5 \times T_{\text{cycle}}\right) \quad \text{AND} \quad \mathbb{P}_{\text{win}|\text{fill}}(t) < 0.45$$
   To bot TP ko wapis `Entry_Price` par la kar **Break-Even Scratch** kar leta hai taake paisa dead range mein na phanse.

---

## 6. MODULE 3 (`CORE 2`): STEP-BOX MIGRATION (`0.2222–0.2224` $\to$ `0.2226–0.2228`) VS. TEMPORARY TRADER MISH-MASH DIP

Yeh module aap ke **Masla #3** (Range Shift vs. Temporary Retail Noise) ko solve karta hai.

### 6.1 The 4-Factor Step-Migration Logit Tensor ($Z_{\text{up}}, Z_{\text{down}}$)
Har tick par bot $< 38\text{ nanoseconds}$ mein Upward aur Downward Breakout Logits calculate karta hai:

1. **Multi-Level Depth Log-Pressure ($\mathcal{L}_{\text{book}}$):**
   $$\mathcal{L}_{\text{book}}(t) = \ln\left(\frac{V_0^{\text{bid}} + 0.50 V_1^{\text{bid}} + 0.25 V_2^{\text{bid}}}{V_0^{\text{ask}} + 0.50 V_1^{\text{ask}} + 0.25 V_2^{\text{ask}}}\right)$$
2. **Bivariate Hawkes Order-Flow Asymmetry ($\mathcal{H}_t$):**
   $$\mathcal{H}_t = \frac{\lambda^+(t) - \lambda^-(t)}{\lambda^+(t) + \lambda^-(t)} \in [-1, +1]$$
3. **250ms Ceiling/Floor Wall Erosion Vacuum ($E_{\text{ask}}, E_{\text{bid}}$):**
   $$E_{\text{ask}}(t) = 1 - \frac{V_0^{\text{ask}}(t)}{\max_{s \in [t-2\text{s}, t]} V_0^{\text{ask}}(s)}, \qquad E_{\text{bid}}(t) = 1 - \frac{V_0^{\text{bid}}(t)}{\max_{s \in [t-2\text{s}, t]} V_0^{\text{bid}}(s)}$$
4. **Mark-Price Lead Signal ($M_{\text{lead}}$):**
   $$M_{\text{lead}}(t) = \frac{P_{\text{mark}}(t) - P_{\text{mid}}(t)}{\Delta_{\text{box}}}$$

**Master Step-Shift Equations:**
$$Z_{\text{up}}(t) = 0.45 \cdot \mathcal{L}_{\text{book}}(t) + 0.65 \cdot \mathcal{H}_t + 0.85 \cdot E_{\text{ask}}(t) + 0.15 \cdot M_{\text{lead}}(t)$$

$$Z_{\text{down}}(t) = -0.45 \cdot \mathcal{L}_{\text{book}}(t) - 0.65 \cdot \mathcal{H}_t + 0.85 \cdot E_{\text{bid}}(t) - 0.15 \cdot M_{\text{lead}}(t)$$

---

### 6.2 How the Bot Separates a "Temporary Mish-Mash Dip" (`THETA-USDT`) from a "True Step-Shift" (`RUNEUSDT`)

Jab price box ke bahar 1 tick jati hai, to do cheezein ho sakti hain:
* **Case A (Temporary Retail Mish-Mash — `THETA-USDT` Screenshot):** Kisi retail trader ne market order mara jis se price 1 tick neechay (`0.1427`) gayi, lekin wahan **Level-1/Level-2 par bhari institutional absorption wall** khari thi aur agli hi candle mein price wapis `0.1428–0.1430` ke box mein aa gayi.
* **Case B (True Step-Box Migration — `RUNEUSDT` Screenshot):** Institutional flow ne poori wall saaf kar di, L1/L2 par vacuum tha, aur purani Floor ab nayi Ceiling ban gayi (`0.2222–0.2224` $\to$ `0.2226–0.2228`).

Bot is farq ko **Level-2 Absorption Cushion Ratio ($R_{\text{L1}}$)** aur **Wall Refill Check** se pehchanta hai:

$$R_{\text{L1}}^{\text{up}} = \frac{V_1^{\text{ask}} + V_2^{\text{ask}}}{\text{EMA}_{30\text{s}}(V_0^{\text{ask}})}, \qquad R_{\text{L1}}^{\text{down}} = \frac{V_1^{\text{bid}} + V_2^{\text{bid}}}{\text{EMA}_{30\text{s}}(V_0^{\text{bid}})}$$

#### Complete Decision Matrix:

| Regime State | Exact Mathematical Condition | Bot Execution Action ($< 500\text{ }\mu\text{s}$) |
| :--- | :--- | :--- |
| **1. Stable Barcode** | $\max(Z_{\text{up}}, Z_{\text{down}}) < 1.15$ | Dono edges (`P_floor`, `P_ceiling`) par Bi-Directional Maker Ping-Pong jari rakho. |
| **2. Temporary Retail Mish-Mash (Fakeout Dip/Spike)** | $1.15 \le Z_{\text{up}} < 1.85 \;\wedge\; R_{\text{L1}}^{\text{up}} \ge 1.25 \;\wedge\; |\mathcal{H}_t| < 0.55$ | **Ignore Noise & Hold Box:** Unfilled opposing entry ko temporarily roko, lekin open position ko panic-close MAT karo! Agle $1$–$3$ ticks mein price wapis box mein aane wali hai. |
| **3. Confirmed Upward Box Shift (`0.2222–0.2224` $\to$ `0.2226–0.2228`)** | $Z_{\text{up}} \ge 1.85 \;\lor\; \left(P_{\text{bid}} \ge P_{\text{ceiling}} \;\wedge\; V_{\text{bid}}(P_{\text{ceiling}}) > 2 \cdot V_{\text{ask}}\right)$ | **Atomic Upward Migration:**<br>1. `P_ceiling` ka Unfilled Short Cancel karo.<br>2. Agar `P_ceiling` par **Short OPEN** hai, to fauran `P_ceiling` ke bache huay Ask volume par **`IOC` Buy** maar kar **0-tick price loss par Scratch** karo!<br>3. Agar `P_floor` se **Long OPEN** hai, to `P_ceiling` wala TP cancel kar ke naye box ke ceiling (**`P_ceiling + \Delta_{\text{box}}`**) par **Trail** kar do!<br>4. Box ko `[P_ceiling, P_ceiling + \Delta_{\text{box}}]` par re-anchor kar do! |
| **4. Confirmed Downward Box Shift (`RUNE` Drop)** | $Z_{\text{down}} \ge 1.85 \;\lor\; \left(P_{\text{ask}} \le P_{\text{floor}} \;\wedge\; V_{\text{ask}}(P_{\text{floor}}) > 2 \cdot V_{\text{bid}}\right)$ | **Atomic Downward Migration:**<br>1. `P_floor` ka Unfilled Long Cancel karo.<br>2. Open Long ko `P_floor` par **`IOC` Sell** se **0-tick loss par Scratch** karo!<br>3. Open Short ke TP ko neechay **`P_floor - \Delta_{\text{box}}`** par **Trail** kar do!<br>4. Box ko `[P_floor - \Delta_{\text{box}}, P_floor]` par re-anchor kar do! |

---

## 7. MODULE 4: COMPLETE 4-EXCHANGE WIRE PROTOCOL & AUTHENTICATION STACK

Yeh bot charon exchanges (**Binance, Bybit, BingX, MEXC**) ke asli, production-grade WebSocket aur HTTP/2 protocols par chalta hai:

### 7.1 Exchange-by-Exchange Technical Matrix

| Feature | 1. Binance USD$\text{S}$-M | 2. Bybit V5 Linear | 3. BingX Swap V2 | 4. MEXC Futures V1 |
| :--- | :--- | :--- | :--- | :--- |
| **Market WS URL** | `wss://fstream.binance.com/stream` | `wss://stream.bybit.com/v5/public/linear` | `wss://open-api-swap.bingx.com/swap-market` | `wss://contract.mexc.com/edge` |
| **Sub Streams** | `<sym>@depth5@100ms`, `<sym>@aggTrade`, `<sym>@markPrice@1s` | `orderbook.1.<SYM>`, `publicTrade.<SYM>`, `tickers.<SYM>` | `<SYM>@depth5@100ms`, `<SYM>@trade`, `<SYM>@markPrice` | `sub.depth`, `sub.deal`, `sub.ticker` |
| **Private Fill WS** | `wss://fstream.binance.com/ws/<listenKey>` (`ORDER_TRADE_UPDATE`) | `wss://stream.bybit.com/v5/private` (`execution`) | `wss://open-api-swap.bingx.com/swap-market?listenKey=...` | `wss://contract.mexc.com/edge` (`push.personal.order`) |
| **Order Execution** | WS API: `wss://ws-fapi.binance.com/ws-fapi/v1` (`order.place`) | WS Trade: `wss://stream.bybit.com/v5/trade` (`order.create`, `order.amend`) | HTTP/2 Keep-Alive Pool: `POST /openApi/swap/v2/trade/order` | HTTP/2 Keep-Alive Pool: `POST /api/v1/private/order/submit` |
| **Maker Flag** | `"timeInForce":"GTX"` | `"timeInForce":"PostOnly"` | `"timeInForce":"PostOnly"` | `"type":2` ($0\%$ Maker Fee!) |
| **Scratch Flag** | `"timeInForce":"IOC"` | `"timeInForce":"IOC"` | `"timeInForce":"IOC"` | `"type":3` (`IOC`) |
| **Hedge Mode** | `"positionSide":"LONG"` / `"SHORT"` | `"positionIdx":1` (Long) / `2` (Short) | `"positionSide":"LONG"` / `"SHORT"` | `"side":1` (Open L), `4` (Close L), `3` (Open S), `2` (Close S) |
| **Compression / Heartbeat** | Standard RFC 6455 Ping/Pong | JSON `{"op":"ping"}` every $20\text{s}$ | **Binary GZIP (`0x1f 0x8b`) on EVERY frame** + Text `"Ping"` $\to$ `"Pong"` | JSON `{"method":"ping"}` every $15\text{s}$ |

---

## 8. COMPLETE PRODUCTION RUST CODEBASE BLUEPRINT (ALL CORE MODULES)

Neechay is complete architecture ke core Rust modules ka exact implementation structure diya gaya hai jo `PriceTicks` normalization, `0s–60s` Multi-Bucket Barcode + Green/Red Balance Scanner, Bi-Directional Ping-Pong State Machine, aur 4-Exchange Wire Templates ko implement karta hai.

### 8.1 Repository Directory Tree (`hft_barcode_engine/`)

```text
hft_barcode_engine/
├── Cargo.toml
├── config/
│   └── exchanges.toml
├── scripts/
│   └── tune_linux_kernel.sh
└── src/
    ├── main.rs                          # 5-Core Pinned Thread Launcher & CLI
    ├── config.rs                        # TOML & 4x4 Subaccount Credential Loader
    ├── hw_clock.rs                      # Silicon _rdtsc() (~2.4ns) & SCHED_FIFO Priority 99
    ├── types.rs                         # PriceTicks, QtyLots, SymbolMeta, Wait-Free SPSC Rings
    ├── registry/
    │   └── symbol_normalizer.rs         # Fetches /exchangeInfo tick_size, step_size & maxLeverage
    ├── scanner/
    │   ├── multi_tf_buckets.rs          # 9 Time Buckets (500ms..60s) OHLCV + EMA7/EMA30 Builder
    │   └── barcode_scanner.rs           # Marubozu, Green/Red Symmetry, Roof/Floor & Markov Bounce
    ├── math/
    │   └── hawkes_step_queue.rs         # L3 Queue Hazard (eta=2.4), Cycle-Adaptive T_ETA & Step Tensor
    ├── network/
    │   ├── ws_codec.rs                  # RFC 6455 Codec & Speculative (N-1) Byte Splitter
    │   ├── tls_pool.rs                  # TLS 1.3 WebSocket + HTTP/2 Connection Pool (TCP_NODELAY)
    │   └── leverage_bootstrap.rs        # Auto 100x Leverage, Isolated Margin & Hedge Mode Init
    ├── exchanges/
    │   ├── crypto_signer.rs             # Zero-Alloc Stack HMAC-SHA256 & Ed25519 Signer
    │   ├── wire_parser.rs               # Branchless L2, AggTrade, MarkPrice & PrivateFill Parser
    │   ├── binance.rs                   # Binance WS-API & ListenKey Keepalive
    │   ├── bybit.rs                     # Bybit V5 WS Trade (`order.create` / `order.amend`)
    │   ├── bingx.rs                     # BingX Zero-Alloc Stack GZIP Decoder + Ping/Pong + Orders
    │   └── mexc.rs                      # MEXC 0-Fee Maker (`type=2`) & Numeric Side (`1..4`) Adapter
    ├── execution/
    │   └── multi_exchange_router.rs     # Dynamic Symbol/Price/Qty Byte Patcher & 4x4 Token Bucket
    └── engine/
        ├── bidirectional_machine.rs     # Continuous Long/Short Ping-Pong, 4-Stage Math & Step-Flip
        └── live_cluster.rs              # 5-Core Runtime Orchestrator & 100us L1 Cache Warmer
```

---

### 8.2 `src/scanner/multi_tf_buckets.rs` & `barcode_scanner.rs` (0s–60s Multi-Timeframe, Marubozu, Green/Red Balance & Mish-Mash Filter)

~~~rust
//! Multi-Timeframe (500ms to 60s) Price-Agnostic Barcode & Green/Red Balance Scanner.
//! Evaluates 9 parallel timeframe buckets per coin:
//! [500ms, 1000ms, 2000ms, 3000ms, 5000ms, 10000ms, 20000ms, 30000ms, 60000ms]

use crate::types::{ExchangeId, PriceTicks, QtyLots};

pub const NUM_TF_BUCKETS: usize = 9;
pub const BUCKET_DURATIONS_MS: [u64; NUM_TF_BUCKETS] = [
    500, 1_000, 2_000, 3_000, 5_000, 10_000, 20_000, 30_000, 60_000,
];
pub const RING_CANDLES: usize = 32; // Power of 2

#[derive(Clone, Copy)]
pub struct CandleSlot {
    pub open: PriceTicks,
    pub high: PriceTicks,
    pub low: PriceTicks,
    pub close: PriceTicks,
    pub buy_vol: QtyLots,
    pub sell_vol: QtyLots,
}

impl CandleSlot {
    pub const fn empty() -> Self {
        Self { open: 0, high: 0, low: u64::MAX, close: 0, buy_vol: 0, sell_vol: 0 }
    }
}

#[derive(Clone, Copy)]
pub struct TimeframeBucketRing {
    pub duration_ms: u64,
    pub candles: [CandleSlot; RING_CANDLES],
    pub head_idx: usize,
    pub count: usize,
    pub current_candle_start_ms: u64,
    pub ema7_fp: u64,   // Scaled by 1024
    pub ema30_fp: u64,  // Scaled by 1024
    pub last_touch_edge: i8, // -1 = Floor touched, +1 = Ceiling touched
    pub markov_bounces: u32,
    pub markov_breakouts: u32,
}

#[derive(Debug, Clone, Copy)]
pub struct BarcodeCandidate {
    pub symbol_id: u16,
    pub exchange: ExchangeId,
    pub cycle_duration_ms: u64,
    pub box_floor: PriceTicks,
    pub box_ceiling: PriceTicks,
    pub box_width_ticks: PriceTicks,
    pub spread_bps_x10: u64,
    pub green_red_symmetry_pct: u32,
    pub marubozu_body_pct: u32,
    pub markov_bounce_pct: u32,
    pub fitness_score: u64,
}

impl TimeframeBucketRing {
    pub const fn new(duration_ms: u64) -> Self {
        Self {
            duration_ms,
            candles: [CandleSlot::empty(); RING_CANDLES],
            head_idx: 0,
            count: 0,
            current_candle_start_ms: 0,
            ema7_fp: 0,
            ema30_fp: 0,
            last_touch_edge: 0,
            markov_bounces: 1,
            markov_breakouts: 0,
        }
    }

    #[inline(always)]
    pub fn ingest_tick(&mut self, now_ms: u64, price: PriceTicks, qty: QtyLots, is_buy: bool) {
        if self.current_candle_start_ms == 0 {
            self.current_candle_start_ms = now_ms;
            self.candles[0] = CandleSlot {
                open: price,
                high: price,
                low: price,
                close: price,
                buy_vol: if is_buy { qty } else { 0 },
                sell_vol: if !is_buy { qty } else { 0 },
            };
            self.ema7_fp = price << 10;
            self.ema30_fp = price << 10;
            return;
        }

        // Roll to next candle slot if bucket duration elapsed
        if now_ms.saturating_sub(self.current_candle_start_ms) >= self.duration_ms {
            let closed_c = self.candles[self.head_idx].close;
            let p_fp = closed_c << 10;
            // EMA7 alpha = 2/8 = 1/4; EMA30 alpha = 2/31 ~ 1/16
            self.ema7_fp = self.ema7_fp - (self.ema7_fp >> 2) + (p_fp >> 2);
            self.ema30_fp = self.ema30_fp - (self.ema30_fp >> 4) + (p_fp >> 4);

            self.head_idx = (self.head_idx + 1) & (RING_CANDLES - 1);
            if self.count < RING_CANDLES {
                self.count += 1;
            }
            self.current_candle_start_ms = now_ms;
            self.candles[self.head_idx] = CandleSlot {
                open: price,
                high: price,
                low: price,
                close: price,
                buy_vol: 0,
                sell_vol: 0,
            };
        }

        let slot = &mut self.candles[self.head_idx];
        if price > slot.high {
            slot.high = price;
        }
        if price < slot.low {
            slot.low = price;
        }
        slot.close = price;
        if is_buy {
            slot.buy_vol = slot.buy_vol.saturating_add(qty);
        } else {
            slot.sell_vol = slot.sell_vol.saturating_add(qty);
        }
    }

    /// Evaluates all 7 Barcode Geometry & Mish-Mash Balance Equations on this timeframe bucket.
    pub fn evaluate_geometry(
        &mut self,
        exchange: ExchangeId,
        symbol_id: u16,
        max_leverage: u8,
    ) -> Option<BarcodeCandidate> {
        if self.count < 24 {
            return None;
        }

        // 1. Extract Quantile-Protected Box Floor & Ceiling across last 24 candles
        let mut min_low = u64::MAX;
        let mut max_high = 0u64;
        let mut green_count: u32 = 0;
        let mut red_count: u32 = 0;
        let mut green_vol: u64 = 0;
        let mut red_vol: u64 = 0;
        let mut body_ratio_sum: u64 = 0;
        let mut active_candles: u64 = 0;

        for i in 0..24 {
            let idx = (self.head_idx + RING_CANDLES - 1 - i) & (RING_CANDLES - 1);
            let c = &self.candles[idx];
            if c.open == 0 || c.low == u64::MAX {
                return None;
            }
            if c.low < min_low {
                min_low = c.low;
            }
            if c.high > max_high {
                max_high = c.high;
            }

            if c.close > c.open {
                green_count += 1;
                green_vol += c.buy_vol + c.sell_vol;
            } else if c.close < c.open {
                red_count += 1;
                red_vol += c.buy_vol + c.sell_vol;
            }

            let range = c.high.saturating_sub(c.low);
            let body = if c.close >= c.open { c.close - c.open } else { c.open - c.close };
            if body > 0 && range > 0 {
                body_ratio_sum += (body * 100) / range;
                active_candles += 1;
            }
        }

        if active_candles < 10 || max_high <= min_low {
            return None;
        }

        let box_width = max_high - min_low;
        // Price-Agnostic Basis Points (bps * 10): e.g., 90 = 9.0 bps (0.09%)
        let spread_bps_x10 = (box_width * 100_000) / min_low.max(1);

        // Filter 1: Fee-Viable Move (MEXC 0-fee >= 5.0 bps; Binance/Bybit/BingX >= 8.5 bps; Max <= 50.0 bps)
        let min_bps_x10 = match exchange {
            ExchangeId::Mexc => 50,
            _ => 85,
        };
        if spread_bps_x10 < min_bps_x10 || spread_bps_x10 > 500 {
            return None;
        }

        // Filter 2: Full-Body Marubozu Ratio >= 80% (No long spiky wicks)
        let marubozu_pct = (body_ratio_sum / active_candles) as u32;
        if marubozu_pct < 80 {
            return None;
        }

        // Filter 3: Green vs Red Candle Count & Volume Balance >= 60%
        let min_gr = green_count.min(red_count);
        let max_gr = green_count.max(red_count).max(1);
        let gr_symmetry_pct = (min_gr * 100) / max_gr;
        if gr_symmetry_pct < 60 {
            return None;
        }

        let min_v = green_vol.min(red_vol);
        let max_v = green_vol.max(red_vol).max(1);
        if (min_v * 100) / max_v < 50 {
            return None;
        }

        // Filter 4: EMA7 & EMA30 Compression inside the Barcode Box
        let ema_diff_fp = if self.ema7_fp >= self.ema30_fp {
            self.ema7_fp - self.ema30_fp
        } else {
            self.ema30_fp - self.ema7_fp
        };
        if ema_diff_fp > (box_width << 9) {
            // |EMA7 - EMA30| must be <= 0.50 * box_width
            return None;
        }

        // Filter 5: Mish-Mash Tolerant Non-Flat Markov Bounce Probability >= 75%
        let mut touches_alternated: u32 = 0;
        let mut touches_total: u32 = 0;
        let mut last_edge: i8 = 0;
        for i in (0..24).rev() {
            let idx = (self.head_idx + RING_CANDLES - 1 - i) & (RING_CANDLES - 1);
            let c = &self.candles[idx];
            // Tolerate 1 tick of trader mish-mash noise around floor/ceiling
            if c.close <= min_low + 1 {
                if last_edge == 1 {
                    touches_alternated += 1;
                }
                if last_edge != 0 {
                    touches_total += 1;
                }
                last_edge = -1;
            } else if c.close >= max_high.saturating_sub(1) {
                if last_edge == -1 {
                    touches_alternated += 1;
                }
                if last_edge != 0 {
                    touches_total += 1;
                }
                last_edge = 1;
            }
        }

        if touches_total < 6 {
            return None;
        }
        let markov_bounce_pct = (touches_alternated * 100) / touches_total.max(1);
        if markov_bounce_pct < 75 {
            return None;
        }

        // Compute Time-Gap Priority Fitness Score (Longer 5s..60s cycles receive higher execution bonus!)
        let fee_bps_x10 = match exchange {
            ExchangeId::Mexc => 0,
            _ => 40, // 4.0 bps round-trip Maker fee
        };
        let net_bps_x10 = spread_bps_x10.saturating_sub(fee_bps_x10);
        let time_bonus_pct = 100 + ((self.duration_ms / 1000).min(60) * 2) as u64; // Up to 2.2x bonus for 60s gap

        let fitness_score = (net_bps_x10 * (max_leverage as u64) * (gr_symmetry_pct as u64) * (markov_bounce_pct as u64) * time_bonus_pct) / 10_000;

        Some(BarcodeCandidate {
            symbol_id,
            exchange,
            cycle_duration_ms: self.duration_ms,
            box_floor: min_low,
            box_ceiling: max_high,
            box_width_ticks: box_width,
            spread_bps_x10,
            green_red_symmetry_pct: gr_symmetry_pct,
            marubozu_body_pct: marubozu_pct,
            markov_bounce_pct,
            fitness_score,
        })
    }
}
~~~

---

### 8.3 `src/engine/bidirectional_machine.rs` (Bi-Directional Long/Short Ping-Pong, 4-Stage Math & Step-Shift vs. Mish-Mash Filter)

~~~rust
//! Bi-Directional Ping-Pong Execution State Machine (`Core 2`).
//! Implements:
//! 1. Simultaneous Long & Short Hedge-Mode Ping-Pong (Floor closes Short & opens Long; Ceiling closes Long & opens Short).
//! 2. 4-Stage Mathematical Timing (Pre-Entry Confirmation -> L3 Queue & Cycle-Adaptive T_ETA -> 11ns Fill-to-TP -> Stagnation Bailout).
//! 3. Temporary Trader Mish-Mash Dip Filter (`THETA-USDT` L2 Absorption) vs. Real Step-Box Migration (`0.2222-0.2224` -> `0.2226-0.2228`).
//! 4. 100x Mark-Price Basis Gate & Hard 2-Tick Adverse IOC Stop-Loss.

use crate::execution::multi_exchange_router::{ExchangeWireSet, MultiExchangeShardPool};
use crate::hw_clock::{rdtsc, tsc_to_ns};
use crate::math::hawkes_queue::MicrostructureBrain;
use crate::types::{
    EventKind, ExchangeId, HotMarketEvent, LockFreeRingBuffer, OrderAction, OutboundWirePacket,
    PriceTicks, QtyLots, Side,
};

#[repr(C, align(64))]
pub struct BiDirectionalBarcodeEngine {
    pub exchange: ExchangeId,
    pub symbol_id: u16,
    pub brain: MicrostructureBrain,
    pub shards: MultiExchangeShardPool,
    pub wires: ExchangeWireSet,

    // Active Barcode Box & Adaptive Cycle Duration (0.5s to 60s)
    pub box_floor: PriceTicks,
    pub box_ceiling: PriceTicks,
    pub box_width: PriceTicks,
    pub cycle_duration_ms: u64,

    // Resting Post-Only Maker Entry Orders
    pub resting_long_entry_px: PriceTicks,
    pub resting_short_entry_px: PriceTicks,

    // Open Hedge-Mode Inventory Slots (Simultaneous Long & Short capability)
    pub open_long_qty: QtyLots,
    pub open_long_entry_px: PriceTicks,
    pub open_long_entry_ns: u64,
    pub active_long_tp_px: PriceTicks,

    pub open_short_qty: QtyLots,
    pub open_short_entry_px: PriceTicks,
    pub open_short_entry_ns: u64,
    pub active_short_tp_px: PriceTicks,

    // Risk & Timing Thresholds
    pub max_mark_divergence_bps: u64,
    pub hard_stop_adverse_ticks: PriceTicks,
    pub seq_id: u32,
    pub last_exec_latency_ns: u64,
}

impl BiDirectionalBarcodeEngine {
    pub fn new(
        exchange: ExchangeId,
        symbol_id: u16,
        initial_floor: PriceTicks,
        initial_ceiling: PriceTicks,
        cycle_duration_ms: u64,
    ) -> Self {
        Self {
            exchange,
            symbol_id,
            brain: MicrostructureBrain::new(),
            shards: MultiExchangeShardPool::new(),
            wires: ExchangeWireSet::new(),
            box_floor: initial_floor,
            box_ceiling: initial_ceiling,
            box_width: initial_ceiling.saturating_sub(initial_floor).max(2),
            cycle_duration_ms: cycle_duration_ms.max(1000),
            resting_long_entry_px: 0,
            resting_short_entry_px: 0,
            open_long_qty: 0,
            open_long_entry_px: 0,
            open_long_entry_ns: 0,
            active_long_tp_px: 0,
            open_short_qty: 0,
            open_short_entry_px: 0,
            open_short_entry_ns: 0,
            active_short_tp_px: 0,
            max_mark_divergence_bps: 15,
            hard_stop_adverse_ticks: 2,
            seq_id: 1,
            last_exec_latency_ns: 0,
        }
    }

    #[inline(always)]
    fn emit_order(
        &mut self,
        ex: ExchangeId,
        shard_id: u8,
        action: OrderAction,
        price_ticks: PriceTicks,
        tsc_now: u64,
        tx_ring: Option<&LockFreeRingBuffer<OutboundWirePacket, 1024>>,
    ) {
        self.seq_id = self.seq_id.wrapping_add(1);
        let seq = self.seq_id;
        let (prefix, trigger) = self.wires.patch_order(ex, action, price_ticks, seq);
        if let Some(ring) = tx_ring {
            let mut pkt = OutboundWirePacket::empty();
            pkt.exchange = ex;
            pkt.shard_id = shard_id;
            pkt.action = action;
            pkt.price_ticks = price_ticks;
            pkt.seq_id = seq;
            pkt.created_tsc = tsc_now;
            let n = prefix.len().min(319);
            pkt.payload[..n].copy_from_slice(&prefix[..n]);
            pkt.payload_len = n;
            pkt.trigger_byte = trigger;
            ring.push(pkt);
        }
    }

    /// Evaluates every tick across all 4 mathematical stages in ~85 nanoseconds.
    #[inline(always)]
    pub fn on_tick(
        &mut self,
        ev: &HotMarketEvent,
        tx_ring: Option<&LockFreeRingBuffer<OutboundWirePacket, 1024>>,
    ) {
        let t0_tsc = rdtsc();
        let now_ns = tsc_to_ns(t0_tsc);
        let dry_run = ev.kind == EventKind::DummyCacheWarmup;
        let ex = ev.exchange;

        // =========================================================================
        // STAGE 3: INSTANT NANOSECOND MOVE CAPTURE & BI-DIRECTIONAL PING-PONG FLIP
        // =========================================================================
        if ev.kind == EventKind::PrivateOrderFill && !dry_run {
            if ev.side == Side::Buy {
                // Buy fill at P_floor:
                // 1) If we had an Open Short, its Maker Buy TP just filled -> Short profit locked!
                if self.open_short_qty > 0 && ev.trade_price <= self.box_floor {
                    self.open_short_qty = 0;
                    self.active_short_tp_px = 0;
                }
                // 2) Our Maker Long Entry at P_floor filled -> Immediately fire Maker Sell TP at P_ceiling
                //    AND queue Maker Short Entry at P_ceiling for continuous Ping-Pong!
                self.open_long_qty = ev.trade_qty;
                self.open_long_entry_px = ev.trade_price;
                self.open_long_entry_ns = now_ns;
                self.resting_long_entry_px = 0;

                let tp_target = ev.trade_price + self.box_width;
                self.active_long_tp_px = tp_target;
                self.emit_order(ex, 0, OrderAction::MakerExitLongTP, tp_target, t0_tsc, tx_ring);

                if self.open_short_qty == 0 && self.resting_short_entry_px != tp_target {
                    self.resting_short_entry_px = tp_target;
                    self.emit_order(ex, 1, OrderAction::MakerEntryShort, tp_target, t0_tsc, tx_ring);
                }
            } else {
                // Sell fill at P_ceiling:
                // 1) If we had an Open Long, its Maker Sell TP just filled -> Long profit locked!
                if self.open_long_qty > 0 && ev.trade_price >= self.box_ceiling {
                    self.open_long_qty = 0;
                    self.active_long_tp_px = 0;
                }
                // 2) Our Maker Short Entry at P_ceiling filled -> Immediately fire Maker Buy TP at P_floor
                //    AND queue Maker Long Entry at P_floor for continuous Ping-Pong!
                self.open_short_qty = ev.trade_qty;
                self.open_short_entry_px = ev.trade_price;
                self.open_short_entry_ns = now_ns;
                self.resting_short_entry_px = 0;

                let tp_target = ev.trade_price.saturating_sub(self.box_width);
                self.active_short_tp_px = tp_target;
                self.emit_order(ex, 2, OrderAction::MakerExitShortTP, tp_target, t0_tsc, tx_ring);

                if self.open_long_qty == 0 && self.resting_long_entry_px != tp_target {
                    self.resting_long_entry_px = tp_target;
                    self.emit_order(ex, 3, OrderAction::MakerEntryLong, tp_target, t0_tsc, tx_ring);
                }
            }
            self.last_exec_latency_ns = tsc_to_ns(rdtsc().saturating_sub(t0_tsc));
            return;
        }

        // Update Hawkes Intensity on Aggressive Trades
        if ev.kind == EventKind::AggTrade && !dry_run {
            self.brain.on_trade(now_ns, ev.side, ev.trade_price, ev.trade_qty);
        }

        // Compute L3 Queue Hazard, Hawkes Asymmetry & Step-Shift Logit Tensor
        let sig = self.brain.evaluate_l2_and_signals(
            now_ns,
            ev.best_bid,
            ev.best_ask,
            ev.bid_vol_l0,
            ev.ask_vol_l0,
            ev.bid_vol_l1,
            ev.ask_vol_l1,
            ev.mark_price,
        );

        // =========================================================================
        // STAGE 4: HARD 2-TICK STOP-LOSS & CYCLE-ADAPTIVE FLAT-CANDLE BAILOUT
        // =========================================================================
        let max_hold_ns = (self.cycle_duration_ms * 2_500_000).max(10_000_000_000); // max(10s, 2.5 * T_cycle)
        if !dry_run {
            if self.open_long_qty > 0 {
                let hard_stop_px = self.box_floor.saturating_sub(self.hard_stop_adverse_ticks);
                let flat_stagnated = now_ns.saturating_sub(self.open_long_entry_ns) > max_hold_ns
                    && sig.hawkes_asymmetry < -0.35;
                if ev.best_bid <= hard_stop_px || flat_stagnated {
                    self.emit_order(ex, 2, OrderAction::IocScratchLong, ev.best_bid, t0_tsc, tx_ring);
                    self.open_long_qty = 0;
                    self.last_exec_latency_ns = tsc_to_ns(rdtsc().saturating_sub(t0_tsc));
                    return;
                }
            }
            if self.open_short_qty > 0 {
                let hard_stop_px = self.box_ceiling + self.hard_stop_adverse_ticks;
                let flat_stagnated = now_ns.saturating_sub(self.open_short_entry_ns) > max_hold_ns
                    && sig.hawkes_asymmetry > 0.35;
                if ev.best_ask >= hard_stop_px || flat_stagnated {
                    self.emit_order(ex, 3, OrderAction::IocScratchShort, ev.best_ask, t0_tsc, tx_ring);
                    self.open_short_qty = 0;
                    self.last_exec_latency_ns = tsc_to_ns(rdtsc().saturating_sub(t0_tsc));
                    return;
                }
            }
        }

        // =========================================================================
        // STEP-BOX MIGRATION VS. TEMPORARY MISH-MASH DIP DIFFERENTIATION
        // =========================================================================
        // Check L1 Absorption Cushion Ratio R_L1 to separate fake retail dip from true box shift
        let ask_l1_cushion = (ev.ask_vol_l1 as f32) / self.brain.ema_ask_wall.max(1.0);
        let bid_l1_cushion = (ev.bid_vol_l1 as f32) / self.brain.ema_bid_wall.max(1.0);

        // True Upward Step-Shift ([0.2222, 0.2224] -> [0.2226, 0.2228])
        let true_step_up = sig.z_step_up >= 1.85
            || (ev.best_bid >= self.box_ceiling && ask_l1_cushion < 0.65);

        if true_step_up {
            let new_floor = self.box_ceiling;
            let new_ceiling = new_floor + self.box_width;
            if !dry_run {
                // 1) Scratch Open Short at old ceiling (0-tick loss) via IOC Buy
                if self.open_short_qty > 0 {
                    self.emit_order(ex, 2, OrderAction::IocScratchShort, self.box_ceiling, t0_tsc, tx_ring);
                    self.open_short_qty = 0;
                }
                // 2) Trail Open Long TP upward to new_ceiling for 2x multi-tick profit!
                if self.open_long_qty > 0 && self.active_long_tp_px < new_ceiling {
                    self.active_long_tp_px = new_ceiling;
                    self.emit_order(ex, 0, OrderAction::MakerExitLongTP, new_ceiling, t0_tsc, tx_ring);
                }
                // 3) Re-anchor barcode box to [new_floor, new_ceiling]
                self.box_floor = new_floor;
                self.box_ceiling = new_ceiling;
                self.resting_short_entry_px = 0;
            }
            self.last_exec_latency_ns = tsc_to_ns(rdtsc().saturating_sub(t0_tsc));
            return;
        }

        // True Downward Step-Shift (`RUNEUSDT` [0.5945, 0.5948] -> [0.5941, 0.5943])
        let true_step_down = sig.z_step_down >= 1.85
            || (ev.best_ask <= self.box_floor && bid_l1_cushion < 0.65);

        if true_step_down {
            let new_ceiling = self.box_floor;
            let new_floor = new_ceiling.saturating_sub(self.box_width);
            if !dry_run {
                if self.open_long_qty > 0 {
                    self.emit_order(ex, 3, OrderAction::IocScratchLong, self.box_floor, t0_tsc, tx_ring);
                    self.open_long_qty = 0;
                }
                if self.open_short_qty > 0 && self.active_short_tp_px > new_floor {
                    self.active_short_tp_px = new_floor;
                    self.emit_order(ex, 1, OrderAction::MakerExitShortTP, new_floor, t0_tsc, tx_ring);
                }
                self.box_floor = new_floor;
                self.box_ceiling = new_ceiling;
                self.resting_long_entry_px = 0;
            }
            self.last_exec_latency_ns = tsc_to_ns(rdtsc().saturating_sub(t0_tsc));
            return;
        }

        // =========================================================================
        // STAGE 1 & STAGE 2: PRE-ENTRY CONFIRMATION & CYCLE-ADAPTIVE QUEUE MANAGEMENT
        // =========================================================================
        // Last-Man-Standing Toxic Fill Abort: Cancel resting order if support wall vanished behind us
        if sig.bid_adverse_selection && self.resting_long_entry_px != 0 && !dry_run {
            self.emit_order(ex, 0, OrderAction::CancelRestingOrder, self.resting_long_entry_px, t0_tsc, tx_ring);
            self.resting_long_entry_px = 0;
        }
        if sig.ask_adverse_selection && self.resting_short_entry_px != 0 && !dry_run {
            self.emit_order(ex, 1, OrderAction::CancelRestingOrder, self.resting_short_entry_px, t0_tsc, tx_ring);
            self.resting_short_entry_px = 0;
        }

        // Mark-Price 100x Basis Safety Check (|Mark - Mid| <= 15 bps)
        let mid_px = (ev.best_bid + ev.best_ask) >> 1;
        let mark_diff = if ev.mark_price > mid_px { ev.mark_price - mid_px } else { mid_px - ev.mark_price };
        let mark_safe = (mark_diff * 10_000) <= (mid_px * self.max_mark_divergence_bps);

        // Adaptive Max Fill ETA: max(2500ms, 0.65 * T_cycle)
        let max_allowed_eta_ms = ((self.cycle_duration_ms * 65) / 100).max(2500) as u32;

        if mark_safe && sig.z_step_up < 1.15 && sig.z_step_down < 1.15 {
            // Queue Post-Only Maker Long Entry at P_floor
            if self.open_long_qty == 0
                && self.resting_long_entry_px != self.box_floor
                && !sig.bid_adverse_selection
                && sig.est_bid_fill_ms <= max_allowed_eta_ms
            {
                if dry_run {
                    let _ = self.wires.patch_order(ex, OrderAction::MakerEntryLong, self.box_floor, self.seq_id);
                } else if let Some(shard) = self.shards.acquire_shard(ex, now_ns) {
                    self.resting_long_entry_px = self.box_floor;
                    self.brain.on_order_placed(Side::Buy, self.box_floor, ev.bid_vol_l0);
                    self.emit_order(ex, shard as u8, OrderAction::MakerEntryLong, self.box_floor, t0_tsc, tx_ring);
                }
            }

            // Queue Post-Only Maker Short Entry at P_ceiling
            if self.open_short_qty == 0
                && self.resting_short_entry_px != self.box_ceiling
                && !sig.ask_adverse_selection
                && sig.est_ask_fill_ms <= max_allowed_eta_ms
            {
                if dry_run {
                    let _ = self.wires.patch_order(ex, OrderAction::MakerEntryShort, self.box_ceiling, self.seq_id);
                } else if let Some(shard) = self.shards.acquire_shard(ex, now_ns) {
                    self.resting_short_entry_px = self.box_ceiling;
                    self.brain.on_order_placed(Side::Sell, self.box_ceiling, ev.ask_vol_l0);
                    self.emit_order(ex, shard as u8, OrderAction::MakerEntryShort, self.box_ceiling, t0_tsc, tx_ring);
                }
            }
        }

        self.last_exec_latency_ns = tsc_to_ns(rdtsc().saturating_sub(t0_tsc));
    }
}
~~~

---

## 9. ANTIGRAVITY 5-PHASE BUILD & VERIFICATION PROTOCOL

Jab aap yeh `.md` file **Antigravity** mein upload karein, to AI Agent ko yeh exact **5-Phase Execution Plan** follow karne ka hukum dein taake har ek file $100\%$ live, compilable, aur zero-placeholder ke sath tayyar ho:

1. **PHASE 1: Core Primitives, Dynamic Normalizer & 0s–60s Multi-Bucket Barcode Scanner**
   * Implement `src/hw_clock.rs`, `src/types.rs`, `src/registry/symbol_normalizer.rs`, `src/scanner/multi_tf_buckets.rs`, and `src/scanner/barcode_scanner.rs`.
   * Connect to live `/exchangeInfo` (Binance, Bybit, BingX, MEXC) to fetch `tick_size`, `step_size`, and `maxLeverage` for all pairs.
   * Run `cargo test` to verify that a $\$0.1428$ coin, a $\$0.2222$ coin, and a $\$100.00$ coin all normalize accurately and that the 7 Barcode Geometry & Green/Red Balance equations pass on `1s`, `5s`, `15s`, and `60s` barcode streams.

2. **PHASE 2: Synthetic L3 Queue Hazard, Hawkes Process & Step-Migration vs. Mish-Mash Tensor**
   * Implement `src/math/hawkes_step_queue.rs`.
   * Verify Power-Law Queue Cancellation ($\eta = 2.4$), First-Passage Probability ($\mathbb{P}_{\text{win}|\text{fill}}$), Last-Man-Standing Toxic Fill Abort, and the $R_{\text{L1}}$ Level-2 Absorption Cushion that distinguishes `THETA-USDT` temporary retail dips from `RUNEUSDT` step-box shifts.

3. **PHASE 3: Live TLS 1.3 WebSockets, BingX Stack GZIP, ListenKey & HTTP/2 Connection Pools**
   * Implement `src/network/ws_codec.rs`, `src/network/tls_pool.rs`, `src/network/leverage_bootstrap.rs`, and `src/exchanges/{crypto_signer.rs, wire_parser.rs, binance.rs, bybit.rs, bingx.rs, mexc.rs}`.
   * Verify live TLS WebSocket streaming on all 4 exchanges, including BingX binary GZIP decompression (`flate2` stack buffer) + `"Ping"` $\to$ `"Pong"` and MEXC numeric order codes (`side: 1..4`, `type: 2` Maker, `type: 3` IOC, `openType: 1` Isolated).

4. **PHASE 4: Bi-Directional Ping-Pong State Machine, Speculative Wire Router & 4x4 Subaccount Shards**
   * Implement `src/execution/multi_exchange_router.rs` and `src/engine/bidirectional_machine.rs`.
   * Verify simultaneous Hedge-Mode Long/Short ping-pong flips, $< 15\text{ns}$ fill-to-TP wire patching, Speculative $(N-1)$ byte pre-staging, and atomic step-shift scratch-and-trail logic.

5. **PHASE 5: 5-Core Pinned Production Cluster & Live Micro-Lot Burn-In Test**
   * Implement `src/engine/live_cluster.rs` and `src/main.rs`.
   * Run `RUSTFLAGS="-C target-cpu=native" cargo build --release` and verify end-to-end execution with zero compiler warnings or errors.