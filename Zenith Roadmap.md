# ZENITH CORP — ROADMAP TECNOLÓGICO V13.0 → V18.0+
> Documento vivo. Última actualización: 2026-04-05
>
> Este documento registra la visión arquitectónica completa del sistema Zenith Corp,
> desde el sprint activo hasta la frontera de investigación avanzada.

---

## Estado Actual — V12.0 (Undecagon)

**11 cerebros activos** en arquitectura Undecagon con pesos dinámicos por régimen (Trend / Range / Chaos).

| Componente | Descripción | Estado |
|---|---|---|
| TitanAI B5 | LightGBM, 40 features, 27 post-pruning | ✓ Activo |
| DeepNet B10 | BiLSTM + Attention, GPU | ✓ Activo |
| DXY Guard B11 | Fórmula ICE oficial, Correlation Guard veto | ✓ Activo |
| KMeans Regime | 3 regímenes (Trend/Range/Chaos) | ✓ Activo |
| SMC Engine | Order Blocks, FVG, Liquidity Map | ✓ Activo |
| Risk Manager | Kill Switch 4%, auditar_paper(), JSONL persistence | ✓ Activo |
| Watchdog | Restart automático de 29 procesos cada 30s | ✓ Activo |
| Telegram | Alertas críticas (Kill Switch) | ✓ Activo |

**Features V6.2 activas:** `usdjpy_dxy_proxy` (#40), `smc_institutional_score`, `hurst_x_vol`, `chaos_composite`, y 36 más.

---

## V13.0 — SPRINT ACTIVO
> **Objetivo:** Cerrar gaps de protección de capital + enriquecer el state space de TitanAI
> **Timeline:** Semanas

### Labeler V6.3 — Enriquecimiento de Features (+8)

Todas añadibles en una sola sesión de labeler + retrain. Sin overhead arquitectónico.

| Feature | Cálculo | Señal que aporta |
|---|---|---|
| `shannon_entropy_100` | Entropía de Shannon sobre últimas 100 velas | Compresión de energía pre-breakout |
| `effort_result_ratio` | `tick_volume / (abs(close-open) + 1e-10)` | Absorción institucional (Wyckoff) |
| `htf_coherence_score` | % de TFs (M5/H1/H4) apuntando en la misma dirección | Alineamiento multi-temporal |
| `hour_sin` / `hour_cos` | Codificación cíclica de la hora | Patrones de sesión sin discontinuidad |
| `london_session` | 1 si hora ∈ [07:00, 12:00] UTC | Sesión de mayor liquidez |
| `ny_session` | 1 si hora ∈ [13:00, 18:00] UTC | Sesión de mayor volatilidad |
| `accel_zscore` | `price_accel` normalizado por rolling σ (50 barras) | Exhaustión / FOMO detection |
| `xau_btc_corr_20` | Correlación rolling 20 barras XAU vs BTC | Anomalía de apetito de riesgo |

**Resultado esperado:** TitanAI V6.3, ~48 features de entrada, ~29-32 post-pruning.

---

### Entropy AI — Veto de Emergencia Sistémica

> **Tipo:** Reglas estadísticas puras. Sin ML. Sin entrenamiento.
> **Rol:** Veto que bypasea el Sínodo completamente — no vota, cancela.

**Lógica:**
```
Monitor cada 30s:
  ATR_M5_actual / ATR_M5_rolling_20d → z-score por activo

  Si 2+ activos superan z > 3.0 simultáneamente → SYSTEMIC EVENT
    Acción: pausa 15 minutos en todos los símbolos

  Si 1 activo supera z > 4.5 → INDIVIDUAL FLASH EVENT
    Acción: pausa 15 minutos en ese símbolo

  Reactivación automática cuando z-score < 2.0
```

**Diferencia con Kill Switch:** El Kill Switch reacciona al drawdown ya ocurrido (4%). Entropy reacciona a la señal de que el daño *está por ocurrir*.

---

### Sentinel AI — Detector Proactivo de Cambio de Régimen

> **Tipo:** Hidden Markov Model (HMM), 3 estados.
> **Rol:** Predicción de transición de régimen antes de que ocurra.

**Diferencia con KMeans actual:**
- KMeans: clasifica el régimen *presente* (reactivo)
- Sentinel: predice *cuándo va a cambiar* el régimen (proactivo)

**Input:**
```
Matriz de correlación cruzada (XAUUSD, XAGUSD, BTCUSD, USDJPY) × ventana 20 barras
+ Hurst rolling, Lyapunov rolling, vol_regime rolling
```

**Output:** `P(cambio_régimen_próximas_5_barras)` ∈ [0, 1]

**Integración en Sínodo:**
```
P > 0.65 → Modo Caution: reducir pesos de todos los cerebros 30%
P > 0.85 → Standby preventivo: no ejecutar hasta que P baje de 0.50
```

---

### Ares AI — Gestor Dinámico de Salida

> **Tipo:** Máquina de estados (V1 reglas) → ML (V2 con datos).
> **Rol:** Toma control del trade post-entrada. Opera en dominio separado al Sínodo.

**Problema que resuelve:** El sistema actual entra con Titan/Deep y espera pasivamente el TP/SL fijo (1.5:1 R:R). Ares convierte esa espera en una decisión activa continua.

**Decisiones de Ares (cada N segundos sobre trade abierto):**
```
HOLD  — momentum sano, no hay OB contrario cercano
TRAIL — mover SL a breakeven o a 0.5R ganado
CLOSE — momentum muriendo + OB contrario en rango → cerrar ahora
```

**Inputs:**
```
- PnL flotante en R (precio_actual vs. entry_price)
- OB contrario dentro de 0.5 ATR (smc_engine en tiempo real)
- price_accel + Hurst últimas 10 barras (momentum proxy)
- Lyapunov speed (¿el movimiento sigue ordenado?)
- accel_zscore > 2.5 → señal de exhaustión de Neuro-Pulse concept
```

---

### Void AI — Detector de Sweep y Trampa de Liquidez

> **Tipo:** Lógica temporal de secuencia. Sin ML en V1.
> **Rol:** Detecta el ciclo antes/durante/después del stop hunt.

**Lo que existe vs. lo que Void añade:**
- Existe: `near_eqh_dist`, `near_eql_dist`, `swept_liq` como features estáticos por barra
- Void añade: **lógica temporal** — la secuencia "precio se acerca → barre → confirma"

```
Estado A: precio SE ACERCA al EQL (within 0.3 ATR) → PELIGRO, voto negativo
Estado B: precio BARRE el EQL (swept_liq activo) → NEUTRO, esperando confirmación
Estado C: precio RECUPERA por encima del EQL (10 barras post-sweep) → OPORTUNIDAD, voto positivo
```

**Impacto:** Elimina el "tenías razón en dirección pero el mecha te sacó por el SL".

---

## V14.0 — CONSOLIDACIÓN
> **Prerequisito:** 200+ paper trades cerrados en JSONL.
> **Timeline:** Meses

### Synthetix Monitor
Análisis batch cada 24h de `paper_trades_YYYY-MM-DD.jsonl`. Reporta Win Rate por símbolo, por régimen, por hora de sesión. Alerta Telegram si algún símbolo baja de 40% WR en últimas 50 trades. **Sin auto-modificación** — genera recomendaciones, el operador decide.

### Atlas AI
Score de apetito de riesgo global basado en correlaciones entre XAU, BTC y JPY. Detecta escenarios anómalos donde activos refugio y de riesgo se sincronizan de forma inusual (señal de evento sistémico). Complementa al DXY Guard sin duplicarlo — DXY mide fortaleza del dólar; Atlas mide cohesión del apetito global.

### Echo AI
Order flow aproximado usando ticks de MT5 (`copy_ticks_from`). Delta = Σ(ticks en ask) - Σ(ticks en bid) por barra. Poor man's order flow — no es Level 2 real pero es mejor que solo tick_volume. Confirma si el OB que ve Titan tiene liquidez real siendo inyectada.

### Proteus AI
Meta-learner que ajusta dinámicamente los pesos de TitanAI vs. DeepNet basándose en performance reciente (ventana 72h). Solo actúa cuando el diferencial de accuracy supera 15 puntos porcentuales. **Requiere mínimo 500 trades cerrados para validez estadística.**

### Genesis V1 — Monte Carlo Condicionado por Régimen
Versión simplificada sin GAN. Usa la distribución histórica de retornos por régimen (Trend/Range/Chaos) para lanzar 1,000 simulaciones antes de ejecutar. Produce un **Índice de Fragilidad** [0,1] — si > 0.80, el trade se aborta. No requiere GPU adicional.

---

## V15.0 — MADUREZ CAUSAL
> **Prerequisito:** 2+ años de datos tick cross-asset. Servidor dedicado para inferencia.
> **Timeline:** 6-12 meses

### Oracle AI — Inferencia Causal (Redes Bayesianas Dinámicas)
No busca correlaciones — busca causalidad. Utiliza tests de **Granger causality** entre los 4 activos (XAU, BTC, JPY, DXY) para determinar qué está causando qué.

**Ejemplo:** "El Oro baja: ¿es porque el DXY subió (causal), o el DXY subió por flujo de JPY (confounding)?" Oracle identifica la causa raíz y puede vetar una señal si el movimiento es ruido de segunda instancia.

**Limitación crítica:** Las relaciones causales en mercados son no-estacionarias. El grafo causal se invalida con cambios de política monetaria. Requiere actualización continua del DAG con priors bayesianos.

---

## V16.0 — PARADIGMA SHIFT
> **Prerequisito:** 1,000+ trades cerrados, framework de backtesting validado, más VRAM.
> **Timeline:** 1-2 años

### Morphos AI — Agente de Reinforcement Learning (PPO/DQN)
El salto de paradigma más importante del roadmap. **No es una mejora incremental — reemplaza la lógica de ejecución del Sínodo.**

```
Zenith V12-V15:  señales → Sínodo vota → ejecutar si Vc > umbral (política fija)
Zenith V16:      señales → Morphos decide política óptima (política aprendida)
```

**Acciones disponibles para el agente:**
- `WAIT` — no hacer nada (imposible en clasificadores supervisados)
- `ENTER_LONG / ENTER_SHORT` — ejecutar trade
- `CLOSE_PARTIAL` — cerrar 50% de la posición
- `CLOSE_FULL` — cerrar todo
- `MOVE_SL` — ajustar stop loss

**State space (todo el trabajo de V13-V15):**
Cada feature del labeler, cada señal de cada cerebro, el régimen actual, el PnL flotante, la hora de sesión — todo se convierte en estado del agente.

**Función de recompensa:**
```
R = ΔBalance - λ×MaxDrawdown - μ×Overtrading_penalty
```

**Nota arquitectónica:** Todo el trabajo realizado en V13-V15 es la construcción del state space que Morphos usará. No es trabajo redundante — es los cimientos.

**Implementación:** Stable Baselines 3 (PPO) + entorno Gym personalizado wrapping datos históricos MT5 con slippage/spread realistas.

---

## V17.0 — DATA-BOUND
> **Prerequisito:** Feed de datos con DOM histórico (Level 2). Más VRAM.
> **Timeline:** Requiere cambio de infraestructura de datos.

### Specter AI — CNN sobre Heatmaps de Liquidez
Red convolucional que procesa **imágenes de footprint charts** (heatmaps de volumen por nivel de precio) en lugar de secuencias de números.

**Por qué no se puede construir antes:** MT5 provee `tick_volume` (conteo de cambios de precio) pero no volumen real por nivel de precio (DOM). Para generar heatmaps de liquidez se necesita DOM histórico tick a tick — disponible solo en feeds profesionales (Rithmic, CQG, Interactive Brokers).

**Una vez con los datos:** La CNN detecta patrones visuales de absorción, icebergs y fake breakouts que son invisibles en OHLC pero evidentes en la geometría del order book.

**Hardware:** Requiere GPU dedicada. En la Victus entraría en contención con DeepNet — este es el escenario que activa la necesidad del Zenith Central Core (orquestador de GPU).

---

## V18.0+ — INVESTIGACIÓN AVANZADA
> **Prerequisito:** Recursos de investigación, equipo dedicado, infraestructura enterprise.
> **Timeline:** 3-5 años. Territorio de fondos cuantitativos de élite.

### Ego-Chain AI — Teoría de Juegos Adversarial (MARL)
Multi-Agent Reinforcement Learning para modelar el comportamiento de adversarios institucionales. Detecta cuando un Order Block es un señuelo algorítmico diseñado para cazar stop-losses de bots SMC.

**Arquitetura:** Motor de simulación de juegos + cálculo de aproximaciones de Nash Equilibrium para N agentes heterogéneos. Inspirado en AlphaStar (DeepMind) y en los modelos de microestructura de mercado usados por Renaissance Technologies.

**Barrera:** Nash Equilibrium con miles de agentes es computacionalmente intratable en tiempo real. Requiere clusters de servidores y simplificaciones matemáticas cuya validez en mercados reales no está demostrada públicamente.

---

### Chrono-Fractal AI — Neural ODEs + Geometría Fractal de Mandelbrot
Basado en el paper "Neural Ordinary Differential Equations" (Chen et al., NeurIPS 2018). Modela el precio como un fluido continuo en lugar de barras discretas — especialmente relevante para Forex donde el tiempo "tick" es irregular.

**Concepto central:** Auto-similitud fractal a través de timeframes. Si el patrón de los últimos 30 segundos es una miniatura estadística de un patrón de H4 del mes pasado, se puede estimar la magnitud del movimiento usando la **dimensión fractal de Hausdorff**.

**Barrera:** Los resolutores numéricos para Neural ODEs son inestables con series temporales financieras ruidosas. Gradientes exploden/desaparecen en alta frecuencia. Requiere investigación matemática dedicada.

---

### Genesis AI V2 — GAN/VAE para Monte Carlo en Tiempo Real
Evolución del Genesis V1 (Monte Carlo paramétrico). Un Generative Adversarial Network entrenado en datos históricos genera 10,000 escenarios de mercado sintéticos pero estadísticamente realistas en milisegundos.

**Ventaja sobre Monte Carlo clásico:** Los GANs aprenden fat tails, volatility clustering y cambios de régimen reales — la distribución normal del Monte Carlo clásico subestima sistemáticamente el tail risk.

**Barrera:** Mode collapse y training instability son problemas abiertos en GANs financieros. La validación de realismo requiere años de datos de referencia. Requiere GPU dedicada para el generador.

---

## Resumen Arquitectónico por Capas

```
┌──────────────────────────────────────────────────────────────────┐
│  CAPA 3: META (V15-V18)                                          │
│  Oracle AI (causalidad) · Proteus AI (pesos dinámicos)           │
│  Morphos AI (agente RL) · Ego-Chain AI (teoría de juegos)        │
├──────────────────────────────────────────────────────────────────┤
│  CAPA 2: SÍNODO (V12 activo + V13 expansión)                     │
│  11 cerebros existentes                                          │
│  + Entropy AI (veto sistémico)                                   │
│  + Sentinel AI (aviso de transición)                             │
│  + Void AI (lógica de sweep)                                     │
│  + Atlas AI (apetito de riesgo)                                  │
│  + Echo AI (order flow)                                          │
├──────────────────────────────────────────────────────────────────┤
│  CAPA 1: AIs INDEPENDIENTES (V12 activo + V13 expansión)         │
│  TitanAI  (CPU · LightGBM · señal de entrada)                    │
│  DeepNet  (GPU · BiLSTM   · señal de entrada)                    │
│  Ares AI  (CPU · rules→ML · gestión de salida)                   │
│  Specter  (GPU · CNN      · heatmaps DOM) ← V17                  │
├──────────────────────────────────────────────────────────────────┤
│  CAPA 0: DATOS                                                    │
│  MT5 live · parquets · tick buffer · DOM feed (V17)              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Regla de Oro Arquitectónica

> **Solo crear un cerebro separado si la lógica que implementa es imposible de capturar como feature estático en el labeler.**
>
> Si es capturrable como feature → labeler (10x más simple, TitanAI lo aprende solo).
> Si requiere estado temporal, lógica post-entrada, o es una capa meta → cerebro separado.

| Criterio | Labeler feature | Cerebro separado |
|---|---|---|
| Cómputo sin memoria temporal | ✓ | |
| Secuencia antes/durante/después | | ✓ |
| Opera sobre trade abierto | | ✓ |
| Veto que bypasea el Sínodo | | ✓ |
| Ajuste de pesos de otros componentes | | ✓ |
| Modelo ML independiente | | ✓ |

---

## Restricciones de Hardware — Victus (i5-13420H)

```
GPU: DeepNet ya vive aquí (BiLSTM).
     Cada modelo GPU adicional → contención de VRAM.
     Máximo recomendado: 2 modelos GPU simultáneos (DeepNet + 1 más).
     Specter (V17) activa la necesidad del Zenith Central Core.

CPU: TitanAI, Sentinel, Entropy, Ares, Void, Atlas, Echo.
     LightGBM + HMM + reglas estadísticas = mucho headroom.
     Sin contención hasta V15+.

RAM: 29 procesos actuales en V12.
     Cada cerebro nuevo = 1 proceso daemon adicional.
     Monitorear cuando > 45 procesos totales.
```

---

## Sprints Planificados

| Sprint | Versión | Contenido | Estado |
|---|---|---|---|
| 1 | V13.0 | Entropy AI + Sentinel AI + Ares AI + Void AI + Labeler V6.3 | 🔵 Planificado |
| 2 | V13.1 | Atlas AI + Echo AI + Synthetix Monitor | ⬜ Pendiente |
| 3 | V14.0 | Proteus AI + Genesis V1 (Monte Carlo) | ⬜ Requiere datos |
| 4 | V15.0 | Oracle AI | ⬜ Requiere infraestructura |
| 5 | V16.0 | Morphos AI | ⬜ Requiere datos + GPU |
| 6 | V17.0 | Specter AI + Zenith Central Core | ⬜ Requiere Level 2 feed |
| 7 | V18.0+ | Ego-Chain · Chrono-Fractal · Genesis V2 | ⬜ Investigación avanzada |

---

*Zenith Corp — Sistema de trading algorítmico multi-cerebro.*
*Este documento es el mapa; el código es el territorio.*
