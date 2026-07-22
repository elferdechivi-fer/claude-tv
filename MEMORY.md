# Memoria — Extensión Claude en Chrome (TradingView)

Este repositorio actúa como espacio de memoria persistente para la extensión de Claude en Chrome que opera sobre TradingView. La extensión no conserva contexto propio entre reinicios de Chrome, por lo que usa este repo para guardar y consultar avances, decisiones y estado de trabajo entre sesiones.

## Estado actual

Creación inicial del espacio de memoria (2026-07-22). Aún no hay avances técnicos registrados.

## Cómo usar este archivo

Cada sesión de la extensión debe:
1. Leer este archivo al iniciar para recuperar contexto previo.
2. Añadir una entrada nueva en la sección "Registro de sesiones" al finalizar trabajo relevante, con fecha y resumen breve.

## Registro de sesiones

- **2026-07-22**: Creación inicial del espacio de memoria. Sin avances técnicos todavía.


2026-07-22 (sesión 2): Primer contacto establecido con Claude Code (sesión "Fix consensus exit minimum profit floor"). Contexto acordado con Fer y con Claude Code:

- Roles: yo (Claude/extensión de Chrome, alias "Claude Chrome") opero visualmente TradingView — leo y edito Pine Script, reviso el editor, gráficos y alertas. Claude Code maneja el servidor y los bots (no tiene acceso directo a TradingView). Fer coordina y es quien autoriza cualquier acción de riesgo real de cualquiera de las dos IAs.
- - Regla de seguridad acordada con Claude Code: preguntas e intercambio de información fluyen libremente entre las dos IAs; cualquier acción con consecuencias reales (tocar servidor, disparar webhooks, cambiar configuración de bots) requiere confirmación explícita de Fer, no alcanza con que la otra IA la pida. Los tokens/credenciales (ej. webhook de clon14) no deben circular por el chat entre IAs; los pasa Fer directamente.
  - - Sistema del proyecto (según resumen de Claude Code): "boot" es el bot original (Binance API Demo, dinero virtual, opera SOL/USDT). Los clones (clon2 a clon14) son variantes del mismo bot con distinto TP/SL, distinta cantidad de votos requeridos y filtros extra (EMA150, EMA200, salida anticipada, SL dinámico). Cada estrategia (RSI, MACD, ARO, OTR, MIT, ENV, BB1, VI0, etc.) manda alertas desde Pine Script a un webhook propio de cada bot cuando "vota"; al juntarse suficientes votos activos a la vez (5 o 6 según el bot) se considera consenso y el bot compra. Cada voto tiene una "vida" que expira si no se renueva.
    - - Hallazgo ya reportado por Claude Code: el consenso de votos no muestra ventaja predictiva real sobre entrar al azar; el win rate alto observado se explica por la estructura de riesgo (TP chico / SL grande), no por calidad de señal.
      - - Pendiente clon14: instalado con infraestructura completa (nginx, Cloudflare, systemd) y filtro EMA150 activo, pero sin alertas de TradingView configuradas apuntando a su webhook. Token ya guardado en el .env del servidor; falta que Fer me pase esa URL/token directamente para cargar las alertas en TradingView.
        - - Tarea abierta que Claude Code me pidió investigar (prioridad: pregunta 1): boot y clon4 a veces reciben la misma alerta al mismo segundo (visto en logs del servidor) pero completan consenso en momentos muy distintos, sin relación con estar ocupados en una operación. Preguntas a resolver mirando TradingView/Pine directamente:
          -   1. ¿Cada bot (boot, clon3, clon4, clon6, clon8, clon9, clon14) tiene su propia instancia de Pine Script por estrategia, o hay un único script compartido que manda la misma alerta a varios webhooks a la vez?
              2.   2. Si son instancias separadas, ¿la lógica de cada estrategia es idéntica entre bots o difiere en parámetros/condiciones/momento de evaluación (cierre de vela vs tiempo real)?
                   3.   3. ¿Hay configuración de "una alerta por barra" vs "una alerta por cada vez que se cumple la condición" que explique el desfase de timing?
                        4. - Próximo paso: aún no empecé a revisar el código Pine en TradingView para responder estas preguntas; queda pendiente para la próxima sesión.
                           - 


2026-07-22 (sesion 3): Primera investigacion tecnica en TradingView sobre la pregunta 1/2/3 que pidio Claude Code (boot vs clon4 desfase de consenso). Hallazgos con evidencia directa (solo lectura, no edite nada):

- Mapeo de pantallas de TV dado por Fer: pantalla '0 ORIGINAL' aloja alertas de boot, clon, clon2, clon3, clon12, clon13, clon14; pantalla '4' aloja clon4; pantalla '5' aloja clon5, clon6, clon8, clon9, clon10. Son 3 pantallas casi identicas en indicadores, cada una dedicada solo a mandar alertas de un subconjunto de bots.
- - Confirmado: NO hay un script Pine compartido. Cada bot tiene su propia copia de archivo por estrategia (ej. 'AROON CLON4' v14.0 editado 19.05.2026, 'AROON CLON6' v6.0 editado 19.05.2026, 'AROON SOLO' v13.0 editado 12.05.2026, probablemente esta ultima es la de boot). Mismo patron para RSI ENVE, MIN TEN, OTROS, BB+1, VI-0 (todas con sufijo CLON4 como archivos separados).
  - - Las copias divergen en version y fecha de edicion entre si, lo que confirma que no se actualizan de forma sincronizada.
    - - Confirmado el comportamiento de 'foto fija': las alertas de CLON 5 en el panel de Alertas tienen fechas de creacion/edicion muy distintas (16, 21 y 22 de julio 2026) para el mismo bot.
      - - Diferencia de timeframe real: CLON 5 corre en 30m; BOOT, CLON2, CLON3, CLON4, CLON13, CLON14 y CLON88 corren en 1m. CLON4 tiene ademas una alerta separada 'EMA200 N CLON4' en 30m, distinta de sus otras alertas de voto en 1m, posible causa concreta del desfase de consenso boot/clon4 que reporto Claude Code (el filtro EMA200 se re-evalua cada 30 min mientras los votos llegan cada 1 min).
        - - Duda sin confirmar: existen alertas nombradas 'CLON 88' y 'CLON 99' en vez de clon8/clon9, no se sabe si es nomenclatura real o error de tipeo antiguo. Pendiente confirmar con Fer.
          - - Reporte todo esto a Claude Code en su sesion (Fix consensus exit minimum profit floor) y quede a la espera de que indique si sigo comparando codigo fuente linea por linea (AROON CLON4 vs AROON SOLO) o si primero confirmamos con Fer la duda del EMA200/30m.
            - 


2026-07-22 (sesion 4): Claude Code confirmo la causa raiz cruzando mi hallazgo de TradingView con el codigo del servidor (WEBHOOK.py, lineas 536-538):

- Cuando llega un voto de cualquier bot, el servidor chequea CONFIG.FILTRO_EMA200_ACTIVO. Si esta en True, el voto se rechaza de inmediato (status: blocked_ema200) en vez de sumarse al conteo de consenso.
- - Ese flag solo se actualiza cuando llega la alerta especial EMA200 (la que corre en 30m), mientras que los votos normales llegan cada 1m. Esto puede dejar pasar hasta 30 minutos con un valor de flag desactualizado, descartando votos que en teoria deberian contar.
  - - Claude Code conecto esto con un bug ya documentado antes: un hueco de 12.6 horas en el heartbeat del filtro EMA200 de clon4, que se habia dejado sin tocar a proposito como base de comparacion limpia. La hipotesis es que el desfase de timeframe (30m vs 1m) es la causa de ese hueco.
    - - Aclaracion de Fer sobre el mecanismo real: la alerta de EMA200 se recibe en clon4 cada 1 minuto, pero esta activada/calculada en el timeframe de 30m. Clon4 actua en consecuencia segun el valor que tenga esa alerta en cada momento (no es que la alerta en si solo llegue una vez cada 30 minutos).
      - - Claude Code recomienda priorizar preguntarle a Fer si el timeframe de 30m en la alerta EMA200 fue una decision de diseno intencional o un descuido, antes de tocar nada en Pine. Bajo prioridad por ahora: comparar linea por linea AROON CLON4 vs AROON SOLO, ya que la explicacion mecanica ya quedo confirmada.
        - - No toque ni edite ningun script ni alerta, solo lectura y coordinacion.
          - 
