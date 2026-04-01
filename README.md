# ZENITH CORP — FLUJO COMPLETO DE UNA OPERACIÓN
## Arquitectura de Decisión: del Mercado a la Orden

---

```
MERCADO (MetaTrader 5 — The 5ers)
│
│  Candles M5  ·  Ticks  ·  Spreads  ·  Volumen
│
▼
══════════════════════════════════════════════════════════
           CAPA 0 — ESCUDO DE NOTICIAS (NewsShield)
══════════════════════════════════════════════════════════
│
│  news_worker() — hilo daemon independiente
│  Monitorea calendario económico cada 60s
│  Pares monitoreados: USD, GBP, EUR
│  Ventana de bloqueo: ±15 min alrededor de noticia roja
│
│  news_event.set()   → sistema operativo (normal)
│  news_event.clear() → todos los cerebros pausan (wait)
│
▼
══════════════════════════════════════════════════════════
      CAPA 1 — SIETE CEREBROS (Procesos Paralelos)
══════════════════════════════════════════════════════════
│
│  35 procesos simultáneos (7 cerebros × 5 símbolos)
│  Cada proceso es independiente — no se bloquean entre sí
│  Todos esperan news_event antes de analizar
│
├─────────────────────────────────────────────────────────
│  [A] WYCK_SENSE  (peso=0.20)  — ciclo cada ~2s / 300s tras señal
│  │
│  │  Fuente: evaluar_simbolo(symbol) → core/analyst.py
│  │
│  │  Proceso interno:
│  │  1. Detecta acumulación / distribución Wyckoff (fases A-E)
│  │  2. Identifica patrones: Spring, UTAD, SOS, SOW
│  │  3. Calcula niveles OTE (Optimal Trade Entry) via Fibonacci
│  │     - Zona válida: retroceso 61.8%–78.6%
│  │     - Rechazo si precio > 78.6% (Stop Hunt)
│  │  4. Verifica CHoCH (Change of Character) para confirmar reversión
│  │  5. Filtra DXY sintético (EURUSD + USDJPY) — veta si dólar
│  │     va en contra de la operación (para XAU y EUR)
│  │  6. Calcula ia_score via TitanAI como filtro secundario
│  │
│  │  Salida: {action, confidence, meta:{entry, sl, tp, patron}}
│  │  meta es CRÍTICO — aporta los precios de la orden
│  │  Cooldown: 300s (5 min) tras emitir señal
│
├─────────────────────────────────────────────────────────
│  [B] QUANT_DELTA  (peso=0.12)  — ciclo cada ~5s / 60s tras señal
│  │
│  │  Fuente: evaluar_estadistica(symbol) → core/quant_engine.py
│  │
│  │  Proceso interno:
│  │  1. Calcula Bandas de Bollinger (BB) en M5
│  │  2. Mide posición del precio dentro de las bandas (bb_pos)
│  │  3. Detecta squeeze (bb_width bajo) → posible expansión
│  │  4. Señal de reversión si precio toca extremos estadísticos
│  │  5. Mide volatilidad relativa (rvol = vol_actual / vol_avg_20)
│  │
│  │  Salida: {action, confidence, status}
│  │  action=0 si no hay señal estadística clara
│
├─────────────────────────────────────────────────────────
│  [D] CHRONOS  (peso=0.12)  — ciclo cada 900s (15 min)
│  │
│  │  Fuente: consultar_rendimiento_historico(symbol) → core/chronos_engine.py
│  │
│  │  Proceso interno:
│  │  1. Consulta TimescaleDB (Docker) — historial de trades del bot
│  │  2. Calcula win rate histórico por símbolo y sesión
│  │  3. Detecta rachas: si hay racha negativa → baja confidence
│  │  4. Ajusta sesgo según hora del día (sesiones Londres/NY/Asia)
│  │  5. Retorna dirección estadísticamente más probable
│  │
│  │  Salida: {action, confidence, status}
│  │  Ciclo largo (15 min) — datos DB son lentos de cambiar
│
├─────────────────────────────────────────────────────────
│  [E] TITANAI  (peso=0.20)  — ciclo cada 20s
│  │
│  │  Fuente: engines/ai_engine.py — HistGradientBoostingClassifier V3.0
│  │
│  │  16 features computadas inline:
│  │  ┌─ BASE TÉCNICOS (7) ──────────────────────────────────┐
│  │  │  rsi          → RSI 14 periodos                      │
│  │  │  rvol         → volumen relativo vs MA20             │
│  │  │  adx          → fuerza de tendencia ADX 14           │
│  │  │  dist_ema50   → distancia precio a EMA50             │
│  │  │  atr          → Average True Range 14                │
│  │  │  prob_markov  → probabilidad Markov cadena (O(n))    │
│  │  │  strategy_type→ 1=scalping M5                        │
│  │  └──────────────────────────────────────────────────────┘
│  │  ┌─ META-FEATURES (3) ──────────────────────────────────┐
│  │  │  bb_pos        → posición Bollinger [0,1] continua   │
│  │  │  macro_delta_proxy → ILG Brain H desde caché JSON    │
│  │  │  lumen_proxy   → pendiente EMA50 / ATR              │
│  │  └──────────────────────────────────────────────────────┘
│  │  ┌─ CHAOS FEATURES (6) ─────────────────────────────────┐
│  │  │  efficiency_ratio → Kaufman ER: 0=caos 1=tendencia   │
│  │  │  hurst_exp        → variance ratio: <0.5=mean-rev    │
│  │  │  autocorr_ret     → autocorrelación lag-1 retornos   │
│  │  │  vol_regime       → coef. variación del ATR          │
│  │  │  price_accel      → tanh(2da derivada del precio)    │
│  │  │  bb_width         → ancho Bollinger / mid (squeeze)  │
│  │  └──────────────────────────────────────────────────────┘
│  │
│  │  Predicción:
│  │  1. prob = model.predict_proba(features)[1] × 100
│  │  2. es_operacion_valida() → prob ≥ threshold calibrado
│  │     threshold viene del entrenamiento (precision ≥ 0.45)
│  │  3. action = (1 si prob>50 else -1) solo si valida=True
│  │
│  │  Salida: {action, confidence=prob/100, status}
│  │  action=0 si TitanAI no da el umbral de calidad
│
├─────────────────────────────────────────────────────────
│  [F] LUMEN  (peso=0.16)  — ciclo cada 45s
│  │
│  │  Fuente: analizar_correlaciones(symbol) → core/lumen_engine.py
│  │
│  │  Proceso interno:
│  │  1. DXY SINTÉTICO — no depende de USDX en el broker:
│  │     - EURUSD vs MA20: bajando = USD fuerte (inverso)
│  │     - USDJPY vs MA20: subiendo = USD fuerte (directo)
│  │     - 2/2 alineados → confidence=0.80
│  │     - 1/2 divergencia → confidence=0.45
│  │  2. Correlación por tipo de símbolo:
│  │     - XAU/EUR/GBP/XAG → inverso al DXY
│  │     - USDJPY → directo al DXY
│  │  3. SMT Divergence (solo XAU):
│  │     - Compara XAUUSD vs XAGUSD tendencia M5
│  │     - Ambos alineados → +0.10 confidence (máx 0.95)
│  │     - Divergencia → -0.20 confidence (mín 0.25)
│  │
│  │  Salida: {action, confidence, status}
│  │  Si sin datos: action=0, confidence=0 (se abstiene)
│
├─────────────────────────────────────────────────────────
│  [G] BIBLIOTECA  (peso=0.10)  — ciclo cada 60s
│  │
│  │  Fuente: engines/biblioteca_engine.py — RAG sobre 75 libros
│  │
│  │  Proceso interno:
│  │  1. Lee candles H1 (100 velas) — timeframe macro
│  │  2. Construye query semántica según régimen:
│  │     - RSI>70: "sobrecompra resistencia distribución"
│  │     - RSI<30: "sobreventa soporte acumulación"
│  │     - ADX>30: "tendencia momentum breakout"
│  │     - Lateral: "consolidación wyckoff esperar"
│  │  3. Busca en embeddings de 75 libros de trading
│  │  4. Retorna consejo del conocimiento literario
│  │
│  │  Salida: {action, confidence, status}
│  │  Peso bajo (0.10) — contexto, no timing
│
├─────────────────────────────────────────────────────────
│  [H] MACROLIQUIDITY  (peso=0.10)  — ciclo cada 7200s (2h)
│  │
│  │  Fuente: engines/macro_engine.py + utils/fred_fetcher.py
│  │
│  │  Proceso interno:
│  │  1. Descarga balances semanales de 4 bancos centrales:
│  │     - Fed (FRED: WALCL)
│  │     - ECB (FRED: ECBASSETSW)
│  │     - BoJ (FRED: JPNASSETS)
│  │     - PBoC (estimado)
│  │  2. Calcula delta: expansión(+) o contracción(-)
│  │  3. Aplica tabla de correlaciones por símbolo:
│  │     XAUUSD: Fed=-1, ECB=+1, PBoC=+1, BoJ=-1
│  │     EURUSD: Fed=-1, ECB=+1, PBoC=0,  BoJ=0
│  │     USDJPY: Fed=+1, ECB=0,  PBoC=0,  BoJ=-1
│  │  4. ILG Score = Σ(delta × correlación × peso) / 5
│  │     normalizado [-1, +1]
│  │  5. Guarda caché en macro_cache/liquidity_index.json
│  │     (usado también por TitanAI como macro_delta_proxy)
│  │
│  │  Salida: {action, confidence, status}
│  │  Ciclo muy largo — datos macro son semanales/mensuales
│
▼
══════════════════════════════════════════════════════════
           CAPA 2 — COLA DE VOTOS (verdict_queue)
══════════════════════════════════════════════════════════
│
│  multiprocessing.Queue(maxsize=500)
│  Thread-safe — cualquier cerebro puede escribir en paralelo
│  Cada voto: {brain_id, symbol, action, confidence, meta, _ts}
│
│  El Sínodo lee la cola con queue.get(block=False)
│  Latencia del loop principal: 10ms (time.sleep(0.01))
│
▼
══════════════════════════════════════════════════════════
        CAPA 3 — SÍNODO (ZenithSynod — proceso maestro)
══════════════════════════════════════════════════════════
│
│  Recibe voto → _procesar_veredicto() → _evaluar_consenso()
│
├── PASO 3.1 — ALMACENAMIENTO CON TIMESTAMP
│   votos[symbol][brain_id] = voto
│   voto['_ts'] = time.time()  ← marca de tiempo para TTL
│
├── PASO 3.2 — EXPIRACIÓN TTL (anti-zombie)
│   Antes de calcular, se expiran votos viejos:
│
│   Cerebro          TTL
│   ─────────────────────────────
│   Wyck_Sense       180s  (3 min)
│   TitanAI           60s  (1 min)
│   Quant_Delta        90s
│   Lumen             120s  (2 min)
│   Chronos           960s  (16 min)
│   Biblioteca        120s  (2 min)
│   MacroLiquidity   7200s  (2 h)
│
├── PASO 3.3 — QUÓRUM POR PESO (nuevo paradigma V2)
│
│   peso_activo = Σ weights[cerebros con voto no expirado]
│
│   peso_activo < 0.40 → ABORTAR (Octágono insuficiente)
│   peso_activo ≥ 0.40 → continuar evaluación
│
│   MODOS DE DISPARO:
│   ┌─ MODO NORMAL ──────────────────────────────────────┐
│   │  Wy presente + dirección alineada con consenso     │
│   │  Usa entry/SL/TP calculados por Wyck_Sense         │
│   └────────────────────────────────────────────────────┘
│   ┌─ MODO BYPASS DE ESTRUCTURA ────────────────────────┐
│   │  Wy ausente O discordante, PERO:                   │
│   │    peso_activo ≥ 0.55  O  n_alineados ≥ 5         │
│   │  Usa entry/SL/TP sintéticos via ATR (RR 1.5:1)    │
│   │  Registra en logs/bypass_audit_YYYY-MM-DD.csv     │
│   └────────────────────────────────────────────────────┘
│
├── PASO 3.4 — SUMA PONDERADA (Vc)
│
│   Vc = Σ (action × confidence × peso)  para cada cerebro activo
│
│   action:     +1 = BUY  |  -1 = SELL
│   confidence: [0.0, 1.0]
│   peso:       weight del Octágono
│
│   fuerza    = |Vc|          ← magnitud del consenso
│   dirección = signo(Vc)     ← BUY o SELL
│
│   Ejemplo con 4 cerebros activos (peso=0.48):
│   Lu(0.16×0.80) + Ch(0.12×0.70) + Bi(0.10×0.62) + Ma(0.10×0.61)
│   = 0.128 + 0.084 + 0.062 + 0.061 = 0.335 → sub-umbral
│
│   Con Wy + Ti añadidos (peso total ~0.88):
│   + Wy(0.20×0.85) + Ti(0.20×0.72)
│   = 0.335 + 0.170 + 0.144 = 0.649 → disparo
│
├── PASO 3.5 — UMBRAL DE DISPARO
│
│   fuerza ≥ 0.55  →  DISPARO
│   fuerza < 0.55  →  esperar (sub-umbral, logueado si > 0.30)
│
│   Matemática del 40% break-even:
│     RR = 1.5:1
│     Break-even = SL/(TP+SL) = 1.0/2.5 = 40%
│     Con 45% precisión real → EV positivo por operación
│
├── PASO 3.6 — LOG DE CONFLICTOS
│
│   Si hay disparo, se registran los cerebros DISCORDANTES:
│   logs/brain_conflicts_YYYY-MM-DD.csv
│   Campos: ts, symbol, dir_sinodo, brain, brain_dir, brain_conf
│
▼
══════════════════════════════════════════════════════════
        CAPA 4 — PROTOCOLO DE EJECUCIÓN
══════════════════════════════════════════════════════════
│
├── FILTRO 4.1 — SENTINEL (único veto binario insalvable)
│
│   verificar_seguridad(symbol) → core/sentinel_engine.py
│
│   Checks:
│   a) SPREAD actual vs límite por símbolo:
│      XAUUSD ≤ 350 pts  |  EURUSD ≤ 20  |  GBPUSD ≤ 30
│      USDJPY ≤ 25 pts   |  XAGUSD ≤ 250
│   b) Horario de rollover (swap overnight) — bloquea si aplica
│
│   PASS → continuar
│   FAIL → ABORT (sin importar Vc ni consenso)
│
├── FILTRO 4.2 — RISK MANAGER
│
│   risk_manager.auditar_cuenta() → core/risk_manager.py
│
│   Checks:
│   a) Drawdown diario ≤ límite configurado
│   b) Pérdida máxima de la cuenta ≤ kill switch
│   c) Número de operaciones abiertas simultáneas
│
│   PASS → continuar
│   FAIL → ABORT
│
├── CÁLCULO 4.3 — MULTIPLICADOR DE RIESGO
│
│   _calcular_multiplicador_acuerdo(votos, dirección)
│
│   ratio = cerebros_alineados / cerebros_activos
│
│   ratio ≥ 1.00  →  multiplicador = 1.00  (7/7 unanimidad)
│   ratio ≥ 0.85  →  multiplicador = 0.75  (6/7)
│   ratio ≥ 0.71  →  multiplicador = 0.50  (5/7)
│   ratio ≥ 0.57  →  multiplicador = 0.25  (4/7)
│   ratio < 0.57  →  multiplicador = 0.10  (mayoría débil)
│
│   El lote final = lote_base × multiplicador
│   → A mayor acuerdo, mayor exposición. Nunca lo contrario.
│
├── OBTENCIÓN 4.4 — META DE PRECIOS
│
│   MODO NORMAL (Wy presente):
│     entry = signal_data['entry']  ← precio OTE de Wyckoff
│     sl    = signal_data['sl']     ← bajo/sobre último swing
│     tp    = signal_data['tp']     ← objetivo de estructura
│
│   MODO BYPASS (sin Wy):
│     tick  = mt5.symbol_info_tick(symbol)
│     atr   = ATR(14 periodos, M5)
│     entry = ask (BUY) | bid (SELL)
│     sl    = entry ∓ 1.0 × ATR      ← RR 1.5:1
│     tp    = entry ± 1.5 × ATR
│
├── EJECUCIÓN 4.5 — BRIDGE MT5
│
│   ejecutar_orden_real(symbol, tipo, precio, sl, tp, risk_multiplier)
│   → utils/bridge.py
│
│   1. Calcula lotaje dinámico según riesgo % de la cuenta
│   2. Aplica risk_multiplier al lotaje base
│   3. Envía orden LIMIT o MARKET via mt5.order_send()
│   4. Retorna {ticket, entry_real, sl, tp} si exitoso
│
▼
══════════════════════════════════════════════════════════
         CAPA 5 — POST-EJECUCIÓN
══════════════════════════════════════════════════════════
│
├── Telegram: alerta inmediata con ticket, tipo, Vc, patrón
├── TimescaleDB (Docker): guardar_trade_db() → histórico permanente
├── Excel: registrar_operacion_excel() → track record auditable
├── Reset de votos: votos[symbol] = {b: None for b in weights}
│   → evita que la misma señal dispare dos veces
│
▼
══════════════════════════════════════════════════════════
         CAPA 6 — TELEMETRÍA GUI (cada 2s)
══════════════════════════════════════════════════════════
│
│  _publicar_telemetria() → escribe estado_zenith.json
│  Escritura atómica: escribe .tmp → os.replace() → JSON final
│  La GUI lee este JSON cada 1500ms y pinta el Octágono
│
└─────────────────────────────────────────────────────────
```

---

## DIAGRAMA DE PESOS DEL OCTÁGONO

```
     OCTAGON — suma de pesos = 1.0
     ┌─────────────────────────────────────────────┐
     │                                             │
     │  Wyck_Sense    ████████████████  0.20       │
     │  TitanAI       ████████████████  0.20       │
     │  Lumen         ████████████      0.16       │
     │  Quant_Delta   █████████         0.12       │
     │  Chronos       █████████         0.12       │
     │  Biblioteca    ███████           0.10       │
     │  MacroLiquidity███████           0.10       │
     │                                             │
     │  Para disparar (NORMAL):  Vc ≥ 0.55        │
     │  Para disparar (BYPASS):  Vc ≥ 0.55 +      │
     │                           peso≥0.55 o 5+   │
     └─────────────────────────────────────────────┘
```

---

## REGLA DE ORO: ÚNICO VETO INSALVABLE

```
           ┌──────────────────────────┐
           │         SENTINEL         │
           │                          │
           │  Spread > límite → VETO  │
           │  Rollover → VETO         │
           │                          │
           │  No importa Vc=1.0       │
           │  No importa 7/7 acuerdo  │
           │  SENTINEL manda siempre  │
           └──────────────────────────┘
```

---

## ARCHIVOS DE AUDITORÍA GENERADOS

| Archivo | Contenido |
|---|---|
| `logs/zenith_v7_deep_core.log` | Log completo del sistema en tiempo real |
| `logs/brain_conflicts_YYYY-MM-DD.csv` | Cerebros discordantes en cada disparo |
| `logs/bypass_audit_YYYY-MM-DD.csv` | Operaciones ejecutadas sin Wyck_Sense |
| `estado_zenith.json` | Snapshot del Octágono para la GUI (cada 2s) |
| `macro_cache/liquidity_index.json` | Caché ILG de bancos centrales (Brain H) |
| `dataset_entrenamiento_titan.csv` | Dataset generado por labeler.py |
| `models/titan_brain.pkl` | Modelo TitanAI V3.0 serializado |
| `Wyck_Sense_TrackRecord.xlsx` | Registro Excel de todas las operaciones |

---

## CICLOS DE ACTUALIZACIÓN POR CEREBRO

```
Tiempo real:   ──────────────────────────────────────────── →
                 2s          20s    45s  60s  90s  2min  15min  2h
                 │           │      │    │    │    │     │      │
  Wyck_Sense ───▲───────────────────────────────────────────────
  TitanAI    ────────────────▲──────────────────────────────────
  Lumen      ─────────────────────────▲──────────────────────────
  Biblioteca ──────────────────────────────▲─────────────────────
  Quant      ─────▲────────────────────────────────────────────
  Chronos    ────────────────────────────────────────────▲──────
  Macro H    ─────────────────────────────────────────────────▲─
```
