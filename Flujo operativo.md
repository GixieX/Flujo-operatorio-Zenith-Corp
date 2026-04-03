# ZENITH CORP — Flujo Operativo Completo
## Decagon V11.0 — 10 Cerebros · Paper Trading · GPU Continuo

> Última actualización: Abril 2026  
> Arquitectura: Decagon (10 brains) · TitanAI V5.0 · BiLSTM+Attention · Perception Loop · Background Trainer

---

## Índice

1.  [Visión General](#1-visión-general)
2.  [Arquitectura de Capas](#2-arquitectura-de-capas)
3.  [Los 10 Cerebros del Decagon](#3-los-10-cerebros-del-decagon)
4.  [Flujo de Decisión — El Sínodo](#4-flujo-de-decisión--el-sínodo)
5.  [Sistema de Pesos Dinámicos por Régimen](#5-sistema-de-pesos-dinámicos-por-régimen)
6.  [Gestión de Riesgo Adaptativa](#6-gestión-de-riesgo-adaptativa)
7.  [Capas de Seguridad](#7-capas-de-seguridad)
8.  [Pipeline DeepNet — Aprendizaje Continuo](#8-pipeline-deepnet--aprendizaje-continuo)
9.  [Ciclo de Vida de un Trade](#9-ciclo-de-vida-de-un-trade)
10. [Archivos y Estructura](#10-archivos-y-estructura)
11. [Métricas y Logs](#11-métricas-y-logs)
12. [Proceso de Entrenamiento TitanAI](#12-proceso-de-entrenamiento-titanai)
13. [Checklist Operativo](#13-checklist-operativo)

---

## 1. Visión General

El **Zenith** es un sistema de trading algorítmico multi-cerebro que opera sobre MetaTrader 5.  
Opera en dos modos:

| Modo | Archivo | Órdenes | Propósito |
|------|---------|---------|-----------|
| **PAPER** | `master_node_paper.py` | Simuladas | Validación, recolección de datos, entrenamiento |
| **LIVE** | `master_node.py` | Reales al broker | Trading en cuenta fondeada |

**Símbolos activos:** XAUUSD · XAGUSD · EURUSD · GBPUSD · USDJPY

**Principio de diseño:** Ningún cerebro individual decide. El consenso ponderado de 10 cerebros especializados —el **Sínodo**— es quien aprueba o rechaza cada señal.

---

## 2. Arquitectura de Capas

```
┌─────────────────────────────────────────────────────────────────────┐
│  CAPA 0 — DATOS EN VIVO (MT5)                                       │
│  Ticks · Velas M5/H1/H4/D1 · Order Book · Tick Volume              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CAPA 1 — ESCUDO DE NOTICIAS (NewsShield)                           │
│  Bloquea todos los cerebros 15min antes/después de eventos macro    │
│  Fuente: calendario económico en tiempo real                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CAPA 2 — LOS 10 CEREBROS (Workers paralelos)                       │
│  Cada cerebro analiza de forma independiente y vota al Sínodo       │
│  Voto: { action: +1/-1, confidence: [0,1], meta: {...} }           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CAPA 3 — EL SÍNODO (PaperSynod / LiveSynod)                        │
│  Gestión de votos · TTL · Quórum · Score ponderado por régimen      │
│  Elastic Threshold · Detección de régimen · Consenso final          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CAPA 4 — SENTINEL (Verificación de seguridad)                      │
│  Spread · Horario · Correlaciones · Exposición total                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CAPA 5 — RISK MANAGER (InstitutionalRiskManager)                   │
│  Cálculo de lotes · SL/TP · Risk adaptativo por régimen             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CAPA 6 — EJECUCIÓN + MONITOREO                                     │
│  Orden paper/real · Break-even adaptativo · Dataset collection      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Los 10 Cerebros del Decagon

### Brain 1 — Wyck_Sense (`core/analyst.py`)
- **Especialidad:** Estructuras Wyckoff — acumulación, distribución, springs, upthrusts
- **Timeframe:** M5 principal
- **Ciclo:** 5 min · TTL: 180s
- **Rol en el Sínodo:** Ancla de estructura. Sin su voto, el trade requiere bypass (≥4 cerebros alineados)

### Brain 2 — TitanAI (`engines/ai_engine.py`)
- **Especialidad:** Predicción por Machine Learning (LightGBM)
- **Features:** 33 variables — RSI, ADX, Hurst, Lyapunov, K-Means cluster, Order Blocks, FVG, EQH/EQL, VSA Climax, Delta Divergence + 3 rugosidad P2
- **Modelos:** `models/titan_brain.pkl` + `models/regime_kmeans.pkl`
- **Ciclo:** 20s · TTL: 60s
- **Survival Objective V2:** Bonus a señales con confirmación SMC institucional

### Brain 3 — Lumen (`core/lumen_engine.py`)
- **Especialidad:** Correlaciones inter-mercado — DXY, commodities, índices
- **Lógica:** Si el contexto macro no acompaña la dirección, reduce confianza
- **Ciclo:** 45s · TTL: 120s

### Brain 4 — Quant_Delta (`core/quant_engine.py`)
- **Especialidad:** Análisis estadístico — distribución de retornos, entropía de Shannon
- **Salida clave:** Campo `entropy` [0,1] → alimenta el **Elastic Threshold** del Sínodo
- **Ciclo:** 60s · TTL: 90s

### Brain 5 — Chronos (`core/chronos_engine.py`)
- **Especialidad:** Rendimiento histórico por hora del día, día de semana, patrón estacional
- **Lógica:** Pondera la señal según el historial de éxito en ese momento temporal
- **Ciclo:** 15 min · TTL: 960s

### Brain 6 — Biblioteca (`engines/biblioteca_engine.py`)
- **Especialidad:** Base de datos de patrones históricos similares (embeddings + similitud coseno)
- **Lógica:** Busca patrones pasados análogos al contexto actual y vota según outcome histórico
- **Ciclo:** 60s · TTL: 120s

### Brain 7 — MacroLiquidity (`engines/macro_engine.py`)
- **Especialidad:** Flujos macro de liquidez global — Fed, ECB, PBoC, BoJ
- **Caché:** `macro_cache/liquidity_index.json`
- **Ciclo:** 2h · TTL: 7200s · Un proceso para todos los símbolos

### Brain 8 — Lazarus (`engines/lazarus_engine.py`)
- **Especialidad:** Order Flow institucional
  - **Stop-Run Detector:** Liquidity grab + wick + RVOL
  - **Killzone Scanner:** Asian range breakout en London/NY open
  - **VSA Effort/Result:** Veto de absorción institucional
  - **Delta Divergence:** Divergencia precio/volumen delta
- **Ciclo:** 30s · TTL: 150s · Un proceso para todos los símbolos

### Brain 9 — SesgoHTF (`engines/htf_engine.py`)
- **Especialidad:** Sesgo institucional de largo plazo (H4 + D1)
- **Lógica:** EMA structure + Market Structure Breaks en H4/D1
- **Rol:** "El anciano del Decagon" — voto de largo plazo que ancla la dirección macro
- **Ciclo:** 4h · TTL: 14400s · Un proceso para todos los símbolos

### Brain 10 — DeepNet (`engines/lstm_brain.py`)
- **Especialidad:** Red neuronal BiLSTM + Self-Attention
- **Input:** 60 velas × 16 features (10 base OHLCV + 6 rugosidad P2)
- **Features rugosidad P2:** Hurst local · RV_5 · RV_20 · RV_ratio · Burstiness · Wick_density
- **Modelo:** `models/deepnet_brain.pth` (GPU CUDA / RTX 4050)
- **Ciclo:** 60s · TTL: 120s
- **Estado inicial:** Neutro (action=0) hasta que exista el modelo entrenado

---

## 4. Flujo de Decisión — El Sínodo

```
NUEVO VOTO llega al Sínodo
        │
        ▼
[1] Expirar votos TTL vencidos
        │
        ▼
[2] Leer régimen de TitanAI.meta.regime (0=Trend / 1=Range / 2=Chaos)
    → Seleccionar REGIME_WEIGHTS activos
        │
        ▼
[3] QUÓRUM: peso activo ≥ 40% del total
    Si NO → abortar (insuficiente información)
        │
        ▼
[4] SCORE ponderado:
    Vc = Σ( action_i × confidence_i × weight_i )
    Vc ∈ [-1.0, +1.0]
        │
        ▼
[5] ELASTIC THRESHOLD (entropía de Quant_Delta):
    H ≤ 0.55 → umbral base (0.30 paper / 0.55 live)
    H ≥ 0.80 → umbral caos  (base + 0.20)
    Intermedio → interpolación lineal
        │
        ▼
[6] ¿|Vc| ≥ umbral_dinámico?
    SI → continuar   NO → registrar en chaos_log si régimen==2, abortar
        │
        ▼
[7] ¿Wyck_Sense votó en la misma dirección?
    SI → ejecución normal
    NO → ¿bypass? (≥4 cerebros alineados) → ejecución con meta sintética ATR
        │
        ▼
[8] Sentinel → Risk Manager → Ejecución
```

---

## 5. Sistema de Pesos Dinámicos por Régimen

El Sínodo selecciona el set de pesos **antes** de calcular el score.  
Cada set suma exactamente **1.00**.

| Cerebro | Trend (0) | Range (1) | Chaos (2) |
|---------|-----------|-----------|-----------|
| Wyck_Sense | 0.16 | 0.14 | 0.08 |
| TitanAI | 0.15 | 0.14 | 0.13 |
| Lumen | 0.11 | 0.13 | **0.15** |
| Quant_Delta | 0.09 | 0.12 | **0.14** |
| Chronos | 0.10 | 0.10 | 0.07 |
| Biblioteca | 0.07 | 0.08 | 0.05 |
| MacroLiquidity | 0.08 | 0.07 | 0.06 |
| Lazarus | 0.09 | 0.10 | **0.15** |
| SesgoHTF | 0.10 | 0.07 | 0.05 |
| DeepNet | 0.05 | 0.05 | **0.12** |

> **Chaos:** Lazarus + Lumen + Quant_Delta + DeepNet al frente.  
> **Trend:** Wyck_Sense + SesgoHTF + TitanAI dominan.  
> **Range:** Quant_Delta + Lumen + Lazarus equilibrados.

---

## 6. Gestión de Riesgo Adaptativa

### Lotaje por régimen (base: 1% de equity)

| Régimen | Factor de lote | Break-Even trigger | Descripción |
|---------|---------------|-------------------|-------------|
| Trend (0) | 1.00× (full) | 50% del recorrido | Posición completa, largo plazo |
| Range (1) | 0.75× | 50% del recorrido | Lote reducido en consolidación |
| Chaos (2) | 0.50× | **25% del recorrido** | Mínima exposición, salida rápida |
| Desconocido (-1) | 1.00× | 50% | Tratado como Trend |

### SL/TP
- **Con Wyck_Sense:** Usa los niveles del análisis Wyckoff (estructura real)
- **Bypass (meta sintética):** `SL = 1.5 × ATR(H1)` · `TP = 2.25 × ATR(H1)` → RR 1.5:1

### Break-Even adaptativo
Cuando el precio avanza el % definido hacia el TP → SL se mueve a entry + 1 pip.  
En Chaos (25%): el sistema asegura ganancia muy temprano para minimizar exposición.

---

## 7. Capas de Seguridad

### 7.1 NewsShield (`utils/news_shield.py`)
Bloquea el news_event 15 minutos antes y después de eventos de alto impacto (★★★).  
Todos los brain workers respetan el bloqueo — ningún voto se emite durante noticias.

### 7.2 Sentinel (`core/sentinel_engine.py`)
Última línea de defensa antes de ejecutar. Verifica:
- **Spread** vs máximo histórico del símbolo
- **Horario de mercado** (no operar en cierre de sesión o fin de semana)
- **Correlación** — no abrir dos posiciones altamente correlacionadas simultáneamente
- **Exposición total** — límite de trades abiertos simultáneos

### 7.3 Conflict Log
Cada vez que el Sínodo detecta cerebros discordantes, registra en:
`logs/brain_conflicts_YYYY-MM-DD.csv`

### 7.4 Chaos Log
Cuando el régimen es Chaos (2), registra **cada evaluación** en:
`logs/chaos_sessions_YYYY-MM-DD.log`  
Campos: `timestamp · symbol · dirección · Vc · entropía · umbral · FIRE/MISS · OB · FVG · liquidez · killzone · acuerdo`

### 7.5 Bypass Audit
Trades ejecutados sin Wyck_Sense quedan auditados en:
`logs/bypass_audit_YYYY-MM-DD.csv`

---

## 8. Pipeline DeepNet — Aprendizaje Continuo

```
┌──────────────────────────────────────────────────────────────┐
│  MT5 — Datos vivos (M5, 60 velas × 5 símbolos)               │
└──────────────┬───────────────────────────────────────────────┘
               │
       ┌───────▼────────┐          ┌────────────────────────┐
       │  Brain #10     │          │  Perception Loop       │
       │  DeepNet       │          │  (cada 30s, GPU FP16)  │
       │  inference     │          │                        │
       │  cada 60s      │          │  - liquidity pressure  │
       │  → vota al     │          │  - chaos index         │
       │    Sínodo      │          │  - institutional mom.  │
       └───────┬────────┘          │  - trend strength      │
               │                   │  → replay_buffer.npz   │
               │                   │  → deepnet_perception  │
               │                   │    .json (GUI)         │
               │                   └──────────┬─────────────┘
               │                              │
       ┌───────▼──────────────────────────────▼─────────────┐
       │  Sínodo → FIRE → Trade abierto                      │
       │  snapshot features guardado en trade_record         │
       └───────────────────────┬─────────────────────────────┘
                               │
               ┌───────────────▼──────────────────┐
               │  Trade cierra WIN / LOSS           │
               │  DatasetCollector.add_sample()     │
               │  → data/lstm_dataset.npz           │
               └───────────────┬──────────────────┘
                               │
               ┌───────────────▼──────────────────┐
               │  Background Trainer (cada 2h)     │
               │  Combina:                         │
               │  - lstm_dataset.npz  (trades)     │
               │  - replay_buffer.npz (continuo)   │
               │                                   │
               │  Fine-tune 8 épocas FP16 GPU      │
               │  WeightedSampler + EarlyStopping  │
               │  → models/deepnet_brain.pth       │
               │  → models/deepnet_chaos.pth       │
               │                                   │
               │  Perception Loop recarga en       │
               │  caliente (detects mtime change)  │
               └──────────────────────────────────┘
```

### Cuándo el DeepNet está listo para entrenar

| Condición | Valor mínimo |
|-----------|-------------|
| Muestras totales | ≥ 100 |
| Muestras WIN | ≥ 50 |
| Muestras LOSS | ≥ 50 |
| Para Chaos Sub-Brain | ≥ 80 (40+40) |

**Entrenamiento manual:**
```bash
# Modelo general
python scripts/background_trainer.py --once

# Chaos Sub-Brain
python scripts/background_trainer.py --once --chaos-only
```

---

## 9. Ciclo de Vida de un Trade

```
[07:03:48]  Sínodo detecta Vc=+0.367 en XAUUSD (régimen=Trend)
            → Wyck_Sense confirmó BUY
            → Sentinel: spread OK, horario OK
            → Risk Manager: lotes=0.01 (paper), lot_factor=1.0 (Trend)
            → PAPER_ejecutar_orden() → ticket #90000001

[07:03:48]  extract_features() captura snapshot 60×16 → guardado en trade_record._dn_features

[07:03:48 → 20:48:08]  Monitor cada 2s:
            - bid sube de 4603.39 hacia TP 4702.65
            - Al 50% del recorrido (~4652): BE activado → SL movido a 4603.40
            - Telegram: "🔒 [BE] BUY XAUUSD — SL en break-even"

[20:48:08]  bid >= 4702.65 → RESULTADO: WIN
            - pips = +9,926
            - DatasetCollector.add_sample(features, label=1, regime=0)
            - Telegram: "✅ [PAPER] BUY XAUUSD → WIN +9926.0 pips"
            - Muestra guardada en data/lstm_dataset.npz
```

---

## 10. Archivos y Estructura

```
Wyck_Scanner/
│
├── master_node_paper.py      # Nodo principal PAPER (lanzar este)
├── master_node.py            # Nodo principal LIVE
├── gui_main.py               # Terminal gráfica V11.0 — Decagon
├── labeler.py                # Etiquetador de dataset TitanAI (offline)
├── data_miner.py             # Descarga históricos para entrenamiento
├── trainer.py                # Entrenamiento TitanAI (LightGBM)
│
├── core/
│   ├── analyst.py            # Brain 1 — Wyckoff
│   ├── quant_engine.py       # Brain 4 — Quant_Delta + entropía
│   ├── lumen_engine.py       # Brain 3 — Correlaciones
│   ├── chronos_engine.py     # Brain 5 — Rendimiento histórico
│   ├── sentinel_engine.py    # Capa de seguridad
│   ├── risk_manager.py       # Gestión de riesgo institucional
│   └── tick_buffer.py        # Buffer de ticks para Lazarus
│
├── engines/
│   ├── ai_engine.py          # Brain 2 — TitanAI (LightGBM + K-Means)
│   ├── lazarus_engine.py     # Brain 8 — Order Flow
│   ├── htf_engine.py         # Brain 9 — SesgoHTF H4/D1
│   ├── lstm_brain.py         # Brain 10 — DeepNet BiLSTM+Attention
│   ├── perception_loop.py    # Análisis continuo GPU FP16 (nuevo)
│   ├── macro_engine.py       # Brain 7 — MacroLiquidity
│   ├── biblioteca_engine.py  # Brain 6 — Patrones históricos
│   ├── smc_engine.py         # Order Blocks + FVG (SMC)
│   └── liquidity_engine.py   # EQH/EQL + SweptLiquidity
│
├── utils/
│   ├── dataset_collector.py  # Recolector de muestras DeepNet
│   ├── news_shield.py        # Escudo de noticias
│   └── telegram_bot.py       # Alertas en tiempo real
│
├── scripts/
│   ├── background_trainer.py # Entrenador automático GPU (nuevo)
│   └── train_lstm.py         # Entrenamiento manual DeepNet
│
├── models/
│   ├── titan_brain.pkl       # Modelo LightGBM entrenado
│   ├── regime_kmeans.pkl     # K-Means de régimen (3 clusters)
│   ├── deepnet_brain.pth     # BiLSTM general (se crea al entrenar)
│   └── deepnet_chaos.pth     # BiLSTM Chaos Sub-Brain (se crea al entrenar)
│
├── data/
│   ├── lstm_dataset.npz      # Dataset trades WIN/LOSS (auto-generado)
│   ├── replay_buffer.npz     # Snapshots auto-etiquetados (Perception Loop)
│   ├── deepnet_perception.json  # Estado perceptual en tiempo real
│   └── trainer_stats.json    # Métricas del último entrenamiento
│
├── logs/
│   ├── paper_trading.log     # Log principal — trades, resultados, votos
│   ├── chaos_sessions_YYYY-MM-DD.log   # Log específico de régimen Chaos
│   ├── brain_conflicts_YYYY-MM-DD.csv  # Cerebros discordantes
│   ├── bypass_audit_YYYY-MM-DD.csv     # Trades sin Wyck_Sense
│   └── background_trainer.log          # Log del entrenador automático
│
└── estado_zenith.json        # Snapshot del estado actual (→ GUI)
```

---

## 11. Métricas y Logs

### paper_trading.log — Entradas clave

```
# Voto de cerebro
[VOTO] Titan  | XAUUSD     | BUY  conf=0.823 peso=0.15 contrib=+0.1235 | V5.0 regime=Trend

# Evaluación del Sínodo
[SINODO XAUUSD]  Vc=+0.3670  BUY  >>> DISPARO <<<  régimen=Trend  peso=0.82  acuerdo=7/10

# Ejecución
[PAPER TRADE #1]  BUY XAUUSD
  Entry : 4603.3900  SL : 4601.5000  TP : 4702.6500
  R:R   : 1:1.50  Lotes : 0.01  Vc=0.3670

# Break-even
🔒 [BE] BUY XAUUSD — SL en break-even @ 4603.4000  Ticket#90000001

# Resultado
✅ [PAPER RESULTADO] BUY XAUUSD  WIN  +9926.0 pips  entry=4603.3900 close=4721.9100  Ticket#90000001
```

### chaos_sessions_YYYY-MM-DD.log — Formato CSV
```
timestamp,symbol,direccion,Vc,entropia,umbral,resultado,ob,fvg,liq,killzone,acuerdo
09:41:50,EURUSD,BUY,0.3017,0.600,0.340,MISS,ob=0,fvg=1,liq=1,killzone=—,acuerdo=5
```

---

## 12. Proceso de Entrenamiento TitanAI

Para reentrenar TitanAI cuando hay nuevos datos históricos:

```bash
# 1. Descargar históricos (banco_datos_historicos/)
python data_miner.py

# 2. Etiquetar con 33 features + P2
python labeler.py

# 3. Entrenar LightGBM + K-Means
python trainer.py
```

**Features del dataset TitanAI (33 total):**
- Base: RSI, ADX, EMA50, EMA200, ATR, RVOL, BB_pos, BB_width, Efficiency Ratio, Hurst, Autocorr, Vol Regime, Price Accel, Lyapunov Speed, Markov Prob, Macro Delta, Lumen Proxy, Estrategia
- SMC: OB_active, OB_direction, OB_dist_ATR, FVG_active, FVG_direction, FVG_dist_ATR, EQH_dist, EQL_dist, SweptLiq, VSA_climax, Delta_divergence, HTF_bias
- Rugosidad P2: RV_ratio, Burstiness, Wick_density

---

## 13. Checklist Operativo

### Inicio de sesión (Paper Trading)
```
[ ] MT5 abierto y conectado al broker
[ ] python master_node_paper.py  (desde el directorio Wyck_Scanner/)
[ ] Verificar banner: "PAPER TRADING NODE V11.0 INICIANDO"
[ ] Verificar: "[PERCEPTION] Continuous Perception Loop activo"
[ ] Verificar: "[BGTRAINER] Background Trainer activo"
[ ] Verificar: "[NEWS SHIELD] Activo"
[ ] Primeros votos llegan en ~2 minutos
```

### Monitoreo diario
```
[ ] Dashboard se actualiza cada 10s en terminal
[ ] GUI: tab PAPER TRADES muestra operaciones abiertas/cerradas
[ ] Telegram: alertas de trades en tiempo real
[ ] logs/paper_trading.log: registro completo
[ ] logs/chaos_sessions_*.log: revisar en días de alta volatilidad
```

### Cuando el DeepNet esté listo para entrenar
```
[ ] Verificar: python -c "import numpy as np; d=np.load('data/lstm_dataset.npz'); print(len(d['y']), 'muestras')"
[ ] Cuando ≥100 muestras (50W+50L): python scripts/background_trainer.py --once
[ ] El Background Trainer también lo hace automáticamente cada 2h
[ ] Verificar: models/deepnet_brain.pth creado
[ ] El sistema recarga el modelo en caliente — sin reiniciar
```

### Transición a LIVE
```
[ ] Mínimo 2 semanas de paper trading con WR > 55%
[ ] Dataset DeepNet ≥ 500 muestras (modelo entrenado)
[ ] Cuenta fondeada configurada en MT5
[ ] Cambiar threshold en master_node.py: VC_THRESHOLD = 0.55
[ ] GUI: cambiar toggle PAPER → LIVE (requiere confirmación)
[ ] Empezar con equity $25,000 → lotaje ~0.30 en XAUUSD
```

---

*Zenith Corp © 2026 — Sistema de trading algorítmico multi-cerebro*  
*Desarrollado por Samuel Esteban Imbrecht Bermudez, todos los derechos reservados*
