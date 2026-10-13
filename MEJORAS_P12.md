# P12 bot 1.4.0 — qué cambia y por qué

## Fallos corregidos (todos con prueba en `test_p12.py`)
| Problema en 1.2.0 | Efecto con dinero real | Arreglo |
|---|---|---|
| El límite de pérdida diaria y la pausa por racha sumaban las operaciones **hipotéticas de todo el top seguido**, no las ejecutadas | Podía pausarse sin haber perdido nada, o no pararse cuando las pérdidas reales sí llegaban | Con LIVE y DRY_RUN=false mandan las **R reales** de BingX (`finish_live`); las hipotéticas quedan solo como estadística |
| `md_get` no conocía 109429 (bloqueo por exceso de errores) ni 109415 (contrato pausado) | Seguía llamando a una ruta bloqueada, alargando el bloqueo | Respeta la hora de reintento que da BingX y aparta 6 h los contratos pausados |
| Nada comprobaba que SL/TP siguieran vivos tras abrir | Una posición podía quedarse sin stop | `guard_stops()` cada ciclo: repone SL/TP si faltan (y no inventa nada si la consulta falla) |
| `MAX_POS` solo contaba posiciones de este bot | Con otro bot en la misma cuenta, exposición doble | `MAX_ACCOUNT_POS` cuenta TODA la cuenta |
| Sin cortacircuitos de equity | Solo había límites en R | `MAX_DD_PCT` (8%) pausa si el equity cae desde su máximo; el máximo es por cuenta (no arrastra el de demo) |
| Sin auditoría al arrancar | Posiciones ajenas o huérfanas pasaban desapercibidas | Mensaje de arranque con equity, posiciones abiertas y las ajenas al bot |

## Herramienta nueva: `/sweep [días] [top]`
Ejecuta el motor **sin ningún filtro** y compara cada uno con "no filtrar": 70 % de los días para entrenar, 30 % final para probar, con corrección de Bonferroni. Un filtro solo sale ✅ si mejora en ambos tramos, la prueba es > 0 y t ≥ umbral. No toca las variables del bot en vivo (los filtros se pasan por parámetro a `simulate`).

## Qué NO está verificado
- Todo se ha probado con un exchange simulado y velas sintéticas, **no contra BingX real**.
- No hay datos que demuestren que la estrategia P12 gane: eso lo dice `/sweep` y `/backtest` sobre tus pares, y luego DRY_RUN.
- El sweep no incluye noticias (el calendario no tiene histórico).

## v1.4 — analizar TODAS las monedas de BingX
| Cambio | Detalle |
|---|---|
| **Escanear todo, ejecutar solo lo líquido** | `MIN_VOL_USDT=3M`, `MAX_UNIVERSE=700`, `MAX_SPREAD_PCT=0.15` para el escaneo (señales y estadística). En real solo se ejecuta si `volumen ≥ MIN_VOL_LIVE (20M)` y `horquilla ≤ MAX_SPREAD_LIVE (0.05%)`; el resto sale como "solo señal" |
| **Categorías** | `CATEGORIES=crypto` (por defecto) / `tradfi` / `all`. Antes los NC… (forex, materias primas, acciones) entraban mezclados y eran los que daban el error 109415 |
| **Escaneo en paralelo** | `SCAN_WORKERS=4` hilos; el limitador de peticiones sigue siendo global (no sube el ritmo contra BingX) |
| **Coste real por moneda** | La horquilla (bid/ask) se suma al coste de ida y vuelta: en monedas de 3 M de volumen pesa mucho más que en BTC |
| **`/universo`** | Embudo: total, categoría, cerrados por API, pausados, volumen, horquilla → escaneadas → ejecutables en real |
| **Telegram** | Los resúmenes largos se trocean por líneas (antes se cortaban a 4096 caracteres, a veces en mitad de una etiqueta HTML) |

### Lo que dice la investigación
- La liquidez y la volatilidad del cripto se concentran en las horas de EE. UU. y Londres, con los spreads más bajos entonces (Aleti & Mizrach 2021; Dimpfl 2017). Operar en la apertura de NY está justificado por **costes**, no por dirección.
- Un estudio con 15 M de observaciones (Baur et al.) no halló patrones de rentabilidad por hora/día que persistan. La ventaja direccional de P12 hay que demostrarla con `/sweep` y `/backtest`, no darla por supuesta.
- Límite de peticiones públicas de BingX: fuentes contradictorias (20/s por IP según Mesh; 500 por 10 s según un cliente de terceros). Se mantiene ≤ 7/s.
- Sesgo de supervivencia: el backtest usa las monedas líquidas **de hoy**; las deslistadas no están.
