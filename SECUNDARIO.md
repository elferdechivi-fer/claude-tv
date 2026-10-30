# SECUNDARIO — Investigaciones cerradas (detalle completo)

Este archivo guarda el detalle completo de investigaciones ya cerradas, sacadas de `MEMORY.md` para mantenerlo liviano. `MEMORY.md` tiene un resumen corto de cada una con referencia a esta sección.

---

## Sesiones 1-9 (2026-07-22) — Investigación TradingView: EMA200, CLON12, clon6, vi0/no, MACD

- **2026-07-22**: Creación inicial del espacio de memoria. Sin avances técnicos todavía.


2026-07-22 (sesión 2): Primer contacto establecido con Claude Code (sesión "Fix consensus exit minimum profit floor"). Contexto acordado con Fer y con Claude Code:

- Roles: yo (Claude/extensión de Chrome, alias "Claude Chrome") opero visualmente TradingView — leo y edito Pine Script, reviso el editor, gráficos y alertas. Claude Code maneja el servidor y los bots (no tiene acceso directo a TradingView). Fer coordina y es quien autoriza cualquier acción de riesgo real de cualquiera de las dos IAs.
- Regla de seguridad acordada con Claude Code: preguntas e intercambio de información fluyen libremente entre las dos IAs; cualquier acción con consecuencias reales (tocar servidor, disparar webhooks, cambiar configuración de bots) requiere confirmación explícita de Fer, no alcanza con que la otra IA la pida. Los tokens/credenciales (ej. webhook de clon14) no deben circular por el chat entre IAs; los pasa Fer directamente.
- Sistema del proyecto (según resumen de Claude Code): "boot" es el bot original (Binance API Demo, dinero virtual, opera SOL/USDT). Los clones (clon2 a clon14) son variantes del mismo bot con distinto TP/SL, distinta cantidad de votos requeridos y filtros extra (EMA150, EMA200, salida anticipada, SL dinámico). Cada estrategia (RSI, MACD, ARO, OTR, MIT, ENV, BB1, VI0, etc.) manda alertas desde Pine Script a un webhook propio de cada bot cuando "vota"; al juntarse suficientes votos activos a la vez (5 o 6 según el bot) se considera consenso y el bot compra. Cada voto tiene una "vida" que expira si no se renueva.
- Hallazgo ya reportado por Claude Code: el consenso de votos no muestra ventaja predictiva real sobre entrar al azar; el win rate alto observado se explica por la estructura de riesgo (TP chico / SL grande), no por calidad de señal.
- Pendiente clon14: instalado con infraestructura completa (nginx, Cloudflare, systemd) y filtro EMA150 activo, pero sin alertas de TradingView configuradas apuntando a su webhook. Token ya guardado en el .env del servidor; falta que Fer pase esa URL/token directamente para cargar las alertas en TradingView.
- Tarea abierta que Claude Code pidió investigar (prioridad: pregunta 1): boot y clon4 a veces reciben la misma alerta al mismo segundo (visto en logs del servidor) pero completan consenso en momentos muy distintos, sin relación con estar ocupados en una operación. Preguntas a resolver mirando TradingView/Pine directamente:
  1. ¿Cada bot tiene su propia instancia de Pine Script por estrategia, o hay un único script compartido que manda la misma alerta a varios webhooks a la vez?
  2. Si son instancias separadas, ¿la lógica de cada estrategia es idéntica entre bots o difiere en parámetros/condiciones/momento de evaluación (cierre de vela vs tiempo real)?
  3. ¿Hay configuración de "una alerta por barra" vs "una alerta por cada vez que se cumple la condición" que explique el desfase de timing?


2026-07-22 (sesion 3): Primera investigacion tecnica en TradingView sobre la pregunta 1/2/3 que pidio Claude Code (boot vs clon4 desfase de consenso). Hallazgos con evidencia directa (solo lectura, no edite nada):

- Mapeo de pantallas de TV dado por Fer: pantalla '0 ORIGINAL' aloja alertas de boot, clon, clon2, clon3, clon12, clon13, clon14; pantalla '4' aloja clon4; pantalla '5' aloja clon5, clon6, clon8, clon9, clon10. Son 3 pantallas casi identicas en indicadores, cada una dedicada solo a mandar alertas de un subconjunto de bots.
- Confirmado: NO hay un script Pine compartido. Cada bot tiene su propia copia de archivo por estrategia (ej. 'AROON CLON4' v14.0 editado 19.05.2026, 'AROON CLON6' v6.0 editado 19.05.2026, 'AROON SOLO' v13.0 editado 12.05.2026, probablemente esta ultima es la de boot). Mismo patron para RSI ENVE, MIN TEN, OTROS, BB+1, VI-0 (todas con sufijo CLON4 como archivos separados).
- Las copias divergen en version y fecha de edicion entre si, lo que confirma que no se actualizan de forma sincronizada.
- Confirmado el comportamiento de 'foto fija': las alertas de CLON 5 en el panel de Alertas tienen fechas de creacion/edicion muy distintas (16, 21 y 22 de julio 2026) para el mismo bot.
- Diferencia de timeframe real: CLON 5 corre en 30m; BOOT, CLON2, CLON3, CLON4, CLON13, CLON14 y CLON88 corren en 1m. CLON4 tiene ademas una alerta separada 'EMA200 N CLON4' en 30m, distinta de sus otras alertas de voto en 1m, posible causa concreta del desfase de consenso boot/clon4 que reporto Claude Code (el filtro EMA200 se re-evalua cada 30 min mientras los votos llegan cada 1 min).
- Duda sin confirmar: existen alertas nombradas 'CLON 88' y 'CLON 99' en vez de clon8/clon9, no se sabe si es nomenclatura real o error de tipeo antiguo.


2026-07-22 (sesion 4): Claude Code confirmo la causa raiz cruzando el hallazgo de TradingView con el codigo del servidor (WEBHOOK.py, lineas 536-538):

- Cuando llega un voto de cualquier bot, el servidor chequea CONFIG.FILTRO_EMA200_ACTIVO. Si esta en True, el voto se rechaza de inmediato (status: blocked_ema200) en vez de sumarse al conteo de consenso.
- Ese flag solo se actualiza cuando llega la alerta especial EMA200 (la que corre en 30m), mientras que los votos normales llegan cada 1m. Esto puede dejar pasar hasta 30 minutos con un valor de flag desactualizado, descartando votos que en teoria deberian contar.
- Claude Code conecto esto con un bug ya documentado antes: un hueco de 12.6 horas en el heartbeat del filtro EMA200 de clon4, que se habia dejado sin tocar a proposito como base de comparacion limpia. La hipotesis es que el desfase de timeframe (30m vs 1m) es la causa de ese hueco.
- Aclaracion de Fer sobre el mecanismo real: la alerta de EMA200 se recibe en clon4 cada 1 minuto, pero esta activada/calculada en el timeframe de 30m. Clon4 actua en consecuencia segun el valor que tenga esa alerta en cada momento (no es que la alerta en si solo llegue una vez cada 30 minutos).


2026-07-22 (sesion 5): Claude Code reviso su hipotesis anterior con el mecanismo correcto (mensaje llega cada 1 min, pero el valor que trae solo cambia cada 30 min):

- Retiro parcialmente la conexion con el hueco de 12.6 horas: si el mensaje llega cada minuto sin falta, un valor desactualizado hasta 30 min no alcanza para explicar un hueco de 12.6 horas por si solo. Sugiere que ese hueco podria ser el filtro funcionando correctamente durante un tramo bajista real y sostenido, no necesariamente un bug.
- Lo que si queda firme: cada voto de clon4 se evalua contra un valor de EMA200 que puede tener hasta 30 minutos de desactualizacion, lo cual alcanza para explicar el desfase de consenso de horas entre boot y clon4 en ventanas cortas.


2026-07-22 (sesion 6): Fer hablo directamente con Claude Code sobre el EMA200/clon4:

- La idea de clon14 nacio justamente para reemplazar a clon4 por este problema.
- Code reviso la tabla historica real de EMA200 de clon4: en datos viejos la mediana real de llegada era 60 minutos (no 1 minuto), con el mismo hueco de 12.6 horas ya documentado como bug de conectividad. En las ultimas 24h (a la fecha) ya no habia huecos grandes (maximo 89 minutos), la confiabilidad de entrega se reparo, pero la frecuencia real seguia siendo ~1 hora, no 1 minuto.
- Conclusion razonada entre Fer y Code: un EMA200 sobre velas de 30m no puede cambiar mas seguido que cada 30m, mandarlo cada 1 minuto seria puro ruido repetido — pasar el heartbeat a 1h es diseno razonable, no un bug. El problema real y ya reparado era la confiabilidad de entrega (el hueco de 12.6h), no la frecuencia.


2026-07-22 (sesion 7 - CIERRE EMA200): Verificacion directa en TradingView del Pine script "EMA200 FILTRO N-- CLON4" (v20, activo), modo solo lectura. Hallazgo tecnico definitivo:

- El script no sondea cada 1 minuto ni cada 30 minutos de forma fija. Dispara alert() en 3 casos: (1) transicion inmediata a condBloqueo, (2) transicion inmediata a condLibre, ambos con alert.freq_once_per_bar; y (3) un heartbeat de respaldo cada 3600000 ms (1 hora) que solo se envia si no hubo cambio de estado real en la ultima hora.
- Como los cambios de estado reales disparan de inmediato (no esperan al heartbeat horario), un cambio de regimen se refleja casi en tiempo real (atado al cierre de vela de 30m) — este mecanismo probablemente NO explica por si solo el desfase de consenso de horas entre boot y clon4 (eso queda como misterio aparte, sin explicacion completa).
- **Conclusion: cadencia horaria = diseno correcto e intencional. El problema real y ya reparado era la confiabilidad de entrega (gap de 12.6h), no la frecuencia. TEMA EMA200/CLON4 CERRADO**, confirmado por Fer, por inspeccion directa de Pine, y por Code via datos reales de DB.


## Sesion 8 (2026-07-22) — Barrido de alertas, hallazgo CLON12, cierre clon6 y BB+1/VI0 LIMPIO

Barrido de lectura sobre las 81 alertas totales del panel de TradingView (confirmado global por simbolo). Hallazgos:

- clon5 vota en 30m, clon10 en 5m (unicos aparte de EMA200/clon4 con timeframe distinto a 1m). boot/clon3/clon13/clon14 combinan BB+1 y VI-0 en una alerta; clon2/clon4/clon5/clon6/clon8/clon9/clon10 los mandan separados. Alerta especial "RECOLECTOR 8 en 1" manda las 8 senales juntas, formato distinto (va1/si/RECOLECTOR). clon2 tiene variantes BB+1 LIMPIO y VI-0 LIMPIO exclusivas. Nomenclatura: "CLON 88"/"CLON 99" en vez de CLON8/CLON9, typo "ARONN" en clon14.
- **CLON12 resuelto**: parecia mudo, en realidad tenia 6 alertas activas (2 STF CONFIRMACION, OTROS, BB COMPLETO, AROON, ENVE, MACD) mandando a su webhook correcto pero **sin el prefijo "CLON12"** en el nombre — por eso el barrido por nombre daba 0. Fer autorizo renombrarlas (solo el campo Nombre, nada de condicion/intervalo/webhook). Code confirmo del lado del servidor: clon12 activo, 10 operaciones reales, 4477 eventos de voto. **Tema CLON12 CERRADO.**
- **BB1/VI0 combinado descartado** como causa del desfase boot/clon4: aunque en TradingView aparezcan agrupados bajo un nombre, el servidor siempre recibe dos mensajes separados y limpios (bb1/si/BB+1, vi0/si/VI-0), el parser no se confunde.
- **clon6 (peor rendimiento, -1374.40 USD)**: causa ya diagnosticada por Code en CONFIG.py — concentracion de riesgo en la 5ta compra del DCA (mismo patron que boot y clon2). Retro-test valida cortar MAX_COMPRAS de 5 a 4: mejora a -1082.88 (sigue negativo pero mejor). **Nunca aplicado en produccion, pendiente autorizacion de Fer.**
- **BB+1 LIMPIO / VI-0 LIMPIO (clon2)**: versiones simplificadas de un solo umbral, webhooks verificados correctos (clon2/webhook, token propio). Solo un detalle cosmetico sin impacto: comentarios viejos en el codigo Pine mencionando CLON3 (resabio de copy-paste). **Cerrado sin cambios pendientes.**
- **Verificacion 100%**: las 81 alertas caen exactamente en el inventario esperado por bot (BOOT 6, CLON2 7, CLON3 6, CLON4 7+1 especial EMA200, CLON5 7, CLON6 7, CLON8 7, CLON9 7, CLON10 7, CLON12 6, CLON13 6, CLON14 6, + RECOLECTOR 8-en-1 que va a un endpoint "auditor2" separado, no a un bot de voto). Suma exacta, ninguna sin explicar.
- Proximo paso propuesto por Code (en su momento): revisar clon6 en profundidad — ya cubierto arriba.


## Sesion 9 (2026-07-22) — Bug vi0/no invertido en clon4, y verificacion clon14/clon6

**Bug vi0/no en clon4 — cerrado con fix aplicado:**
- Fer pregunto por que vi0/no en clon4 parecia fijo en 80.1 (464 disparos historicos) cuando la formula en teoria deberia variar. Confirmado: formula discreta (temaSignal_no + kstSignalAdj_no + stochRsiSignal_no + ulcerSignal_no + zigzagSignal_no, x100/5) solo puede dar multiplos de 20 — es diseno correcto desde el despliegue original (version 177 de BB COMPLETO, 3 mayo), no perdida de granularidad. Dos valores atipicos (85.1/86.2 en clon3) resultaron ser artefactos DUPLICADO de un arranque, no senal real.
- **Problema real, separado**: clon4 tenia el umbral vi0 no-invertido mal en el servidor (clon4/CONFIG.py): umbral_no=20 + condicion "menor que", cuando el resto de los bots (clon3/6/8/9/10/14) usan correctamente umbral_no=79 + "mayor que". Esto hacia que 80.1 nunca pasara — **clon4 rechazaba el 100% de sus votos bajistas de vi0 desde el 12 de mayo (519 rechazos historicos)**. boot/clon2/clon12/clon13 usan un mecanismo distinto (umbral unico de 80) sin este problema.
- **Fix aplicado por Code** (autorizado directamente por Fer en su sesion): backup de CONFIG.py, umbral_no a 79, condicion a "mayor que", py_compile, verificacion de en_posicion antes de reiniciar. Ejecucion 100% de Code, no de esta sesion.

**Verificacion clon14 BB1 — sin bug:**
- Alerta CLON14 == BB1=VIO correctamente apuntada a BB COMPLETO V10.19.7.23 v178 (misma que CLON3). Alerta creada el mismo dia que clon14 arranco (21/jul), primera y unica activacion con condicion extrema (poco frecuente). Los cero votos previos se explican por ser alerta nueva + condicion rara, no falla tecnica.

**Bug clon6 MACD — cerrado con fix aplicado:**
- Alerta "CLON 6 <<< MACD" estaba apuntada por error a la Condicion "MIN TEN CLON6" (el mismo script que la alerta separada MIN-TEN), mandando mensajes mit/MIN-TEND en vez de mac/MACD — mal configurada desde su creacion (18/may). **Fix aplicado y guardado en TradingView** (ventana 5, layout "5 CLONESS"): se corrigio el input del indicador MACD CLON6 (Cooldown entrada/salida de 550→120 segundos, valor confirmado por Code contra el codigo del servidor), lo que habilito la version correcta v14.0 en el dropdown; se reabrio la alerta, se selecciono esa version y se guardo, verificado persistido correctamente.

**Misterio sin resolver**: el desfase original de timing en el consenso boot/clon4 nunca se explico del todo — dos candidatos investigados y descartados (EMA200, BB1/VI0 combinado). Retorno decreciente de seguir investigando esto segun Code.
