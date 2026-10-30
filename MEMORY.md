# Memoria — Extensión Claude en Chrome (TradingView) / Coordinación con Claude Code

Este repositorio es el espacio de memoria persistente compartido entre Claude Chrome (extensión, sin memoria propia entre reinicios de Chrome), Claude Code (sesión bridge "795" en el droplet BOOT), y Fer. Documenta decisiones, hallazgos cerrados y trabajo en curso.

**Archivo hermano: `SECUNDARIO.md`** — guarda el detalle completo de investigaciones ya cerradas, para mantener este índice liviano. Cada resumen de abajo tiene su versión extendida ahí.

## Roles y reglas

- **Claude Chrome**: opera visualmente TradingView (Pine Script, alertas, gráficos).
- **Claude Code** (sesión 795, droplet BOOT): maneja servidor y bots, sin acceso directo a TradingView.
- **Fer**: coordina y autoriza toda acción de riesgo real de cualquiera de las dos IAs.
- Info fluye libre entre IAs; acciones con consecuencias reales (servidor, webhooks, config de bots) requieren autorización explícita de Fer. Credenciales nunca circulan por chat entre IAs, las pasa Fer directo.

## Sistema del proyecto

"boot" es el bot original — **ojo con el fraseo**: tiene la lógica/estructura de "dinero real" pero corre sobre Binance Demo API (Code confundió esto dos veces, ver sesión 11). Los clones (clon2 a clon15) son variantes con distinto TP/SL, votos requeridos y filtros (EMA150, EMA200, SL dinámico, etc.). Cada estrategia manda alertas Pine→webhook propio del bot; consenso de votos (5-6 según bot) dispara la compra. **El consenso no tiene ventaja predictiva real sobre el azar** — el winrate alto observado viene de la estructura de riesgo (TP chico/SL grande), no de calidad de señal.

## Investigación TradingView, julio 2026 (sesiones 1-9) — CERRADA

Entre el 22/jul y principios de septiembre, Claude Chrome investigó una serie de hallazgos en TradingView, coordinando con Claude Code. **Detalle completo en `SECUNDARIO.md`.** Resumen de lo cerrado:

- **EMA200/clon4**: diseño correcto (heartbeat horario sobre velas de 30m); el problema real había sido un gap de conectividad de 12.6h, ya reparado. No explica el desfase de consenso boot/clon4 (misterio que quedó sin resolver — dos candidatos descartados en total, ver abajo).
- **CLON12**: no estaba mudo, tenía 6 alertas activas sin el prefijo "CLON12" en el nombre. Renombradas (autorizado por Fer), confirmado activo con datos reales del servidor.
- **BB1/VI0 combinado**: descartado como causa de nada — el servidor siempre recibe los mensajes separados.
- **clon6** (peor rendimiento, -1374.40 USD): causa ya diagnosticada por Code (concentración de riesgo en 5ta compra DCA). Fix de cortar MAX_COMPRAS a 4 probado (-1082.88) pero **nunca aplicado en producción — sigue pendiente de autorización de Fer.**
- **vi0/no en clon4**: bug real de umbral invertido (rechazaba 100% de votos bajistas VI0 desde el 12/may). **Fix aplicado** por Code con autorización de Fer.
- **clon6 MACD**: bug real, alerta mal apuntada al script de MIN-TEN. **Fix aplicado** y guardado por Claude Chrome en TradingView.
- Pendientes cosméticos sin impacto funcional: "CLON 88"/"CLON 99" en vez de CLON8/CLON9, typo "ARONN" en clon3/14.
- Las 81 alertas del ecosistema quedaron identificadas al 100%.
- **Sin resolver**: el desfase de timing original en el consenso boot/clon4 — dos candidatos investigados y descartados, retorno decreciente de seguir ahí.

## Sesión 10 (2026-09-17/18) — Se descarta el flujo de investigación intermedia; primer fix de la sesión bridge 795

Fer decidió que pasar código/capturas a Code es más directo hacerlo él mismo, sin pasar por Claude Chrome como intermediaria. Este `MEMORY.md` sigue existiendo pero el rol activo de "Claude Chrome investiga y reporta a Code" queda discontinuado.

En paralelo (vía esta sesión de Claude Code, no Claude Chrome) se diagnosticó y resolvió un problema de conectividad de la sesión bridge **"795"** (`session_013gbfzY58XVV5Mgsvk34Muk`, servicio systemd `claude-persistent.service` en el droplet BOOT, tmux `claude_prod`, proceso `claude --resume ac90ed99-5fd6-4578-b43b-e0d455adc795 --remote-control droplet-boot`). Síntoma: figuraba archivada y luego desconectada pese a reiniciar el servicio varias veces.

**Causa raíz**: el script `/root/reiniciar_claude_tv.sh` y el `.service` lanzan `claude` con el flag `--chrome` (integración Claude in Chrome). Ese flag dispara una pantalla de confirmación local ("Enter to confirm") en el primer arranque; como el proceso se lanza desatendido dentro de un `tmux -d`, nadie la confirmaba — el proceso quedaba congelado ahí, sin establecer nunca la conexión remote-control real, aunque `tmux has-session` reportara "todo OK" (ese chequeo solo verifica que la sesión tmux exista, no que el proceso haya pasado la pantalla).

**Fix aplicado**: sin reiniciar nada, se mandó el Enter pendiente con `tmux send-keys -t claude_prod Enter`. El proceso pasó la pantalla, cargó la herramienta `claude-in-chrome` y estableció la conexión remote-control. Confirmado por Fer: sesión 795 activa y operativa.

**Aviso para el futuro**: si se reinicia esta sesión con `--chrome` de forma desatendida, puede volver a quedar trabada en la misma pantalla la primera vez (aunque sesión 11 mostró que no siempre pasa — ver abajo).

## Sesión 11 (2026-10-27) — Reboot completo del droplet BOOT

Planificado con anticipación junto a Code antes de ejecutarlo. Nota de proceso: Code no tenía el incidente del `--chrome` (sesión 10) documentado en su propia memoria de proyecto — solo existía acá. Son dos sistemas de notas separados sin referencia cruzada entre sí, vale la pena recordarlo.

Code afirmó primero, **sin verificar**, que el bug del `--chrome` "no la bloquearía" en este reboot. Al pedirle que lo chequeara en vez de asumir, revisó su propio código/config y se corrigió: no tenía base real para esa afirmación (encontró una pista no concluyente, el flag `hasCompletedClaudeInChromeOnboarding` ya en `True`). Más importante: verificó que el fallback existente en `reiniciar_claude_tv.sh` es **ciego ante este modo de falla exacto**, porque solo se activa si `tmux has-session` falla, y en el incidente de sesión 10 `has-session` daba OK mientras el proceso estaba trabado.

Con eso, armó y probó en modo lectura `/root/verificar_reboot.sh` (queda grabado en el droplet, sobrevive reboots), que en una sola corrida verifica: los 35 servicios systemd del ecosistema completo (trading real: boot, clon2 y los demás clones DCA, bootf en futuros testnet, clon15-recolector/dashboard; soporte: alertas_tv como multiplexor de alertas TV→clones, auditor, auditor2, precios-binance, los 12 simulador-clonN, nginx, cloudflared; y claude-persistent); el estado `en_posicion` de los 9 bots que debían volver en posición (boot, clon2, clon6, clon8, clon9, clon12, clon13, clon14, bootf) contra la tabla `persistencia` de cada base; y el panel tmux de Code vía `capture-pane` (no `has-session`) para detectar si queda trabado.

**Aclaración importante**: en el briefing Code escribió "boot (DINERO REAL)" refiriéndose a la estructura/lógica de boot, lo cual generó una alarma real dado que contradice lo documentado (boot = Binance Demo API). Fer confirmó que fue un error de fraseo: boot simula la lógica de dinero real pero corre en demo. **Nota de proceso repetida dos veces ya** — pendiente que Code corrija esto de forma permanente en su fraseo (ver sesión 12, donde se trabajó esto en profundidad).

**Resultado del reboot**: Code volvió sola, conectada, sin necesitar el `tmux send-keys` manual de la vez pasada — el bug de sesión 10 no se repitió (coincide con la pista del flag de onboarding). Confirmado de forma independiente desde esta sesión con `get_session`.

**Cierre verificado** con `/root/verificar_reboot.sh`: 35/35 servicios `active`, kernel actualizado (6.8.0-139 → 6.8.0-147). De los 9 bots esperados en posición, 5 (clon2, clon8, clon9, clon13, clon14) volvieron en `False` — alarma inicial de posible bug de persistencia. Code investigó a fondo (no se quedó con la superficie): confirmó vía `simulador.db` y el timestamp/razón de la última operación de cada uno que los 5 cerraron por **TP legítimo** entre las 16:09 y 16:13, antes del reboot (16:21:57) — el briefing tenía ~13 min de antigüedad para cuando se corrió la verificación. **Cero posiciones huérfanas.** boot recargó su posición correctamente.

## Sesión 12 (2026-10-27/30) — Auditoría de la arquitectura de memoria de Code

A raíz de una duda sobre riesgo de compactación/alucinación en sesiones muy largas, se le pidió a Code una auditoría honesta de su propia estructura de memoria (`.claude/projects/-root/memory/`: 219 archivos .md, 1.44 MB — 217 "fichas" `project_`/`feedback_`/`reference_` + `MEMORY.md` propio como índice + `SECUNDARIO.md`). Hallazgo central, con números reales: el índice de Code (22.3 KB) ya estaba 31% por encima de su propio límite declarado (17 KB), acercándose al límite de lectura nativo de Claude Code (200 líneas / 25 KB) donde el contenido pasado ese punto **deja de cargarse** — y crecía ~2.2 fichas/día sin mecanismo de poda, con `SECUNDARIO.md` propio subusado (9 KB vs 215 fichas activas).

Se propuso un organigrama con dos archivos nuevos (`RESULTADOS.md` para no re-testear lo ya probado, `HERRAMIENTAS.md` como catálogo de scripts confirmados). Code lo evaluó con rigor contra su flujo real y contra la documentación oficial de Claude Code (verificada vía agente interno `claude-code-guide`): confirmó que el mecanismo nativo de medición ya existe (no hay que inventarlo), que `/doctor` no cubre el sistema de auto-memory, que `CLAUDE.local.md` no aplica (el directorio no es un repo git), y que la carga perezosa (`.claude/rules/` con `paths`) sirve para `HERRAMIENTAS.md` pero no para `RESULTADOS.md` (que ya cumple su función como el bloque "Refutados" existente). Diseñó un hook `PreToolUse` como único agregado legítimo — bloquea el write si `MEMORY.md` superaría 24 KB, forzando podar en el momento en vez de diferirlo (aviso nativo a 17 KB, bloqueo duro a 24 KB). Validado además por un segundo chequeo con el agente interno, que encontró un gotcha real de implementación (un hook sobre `Edit` no recibe el archivo resultante, hay que simular el reemplazo).

**Plan acordado, en orden de impacto** (no de cuánto se discutió cada uno): (1) borrar `.bak`/dumps viejos sin pasar por git — excepto `MEMORY.md.bak_20260917` (33.8 KB), que primero se diffea; (2) Fer corre `/doctor prompt-audit` sobre `clon15/CLAUDE.md` (167 KB/~42K tokens, el mayor gasto de contexto); (3+4) como par simbiótico: uso real de `SECUNDARIO.md` acoplado al disparo del hook (sin eso, se pudre como ya le pasó); (5) — **el punto que Code identificó como el más importante de los cinco**: el hábito de afirmar cosas con confianza sin verificar en el momento (los dos incidentes de "boot — dinero real" sin aclarar demo, y el "--chrome no me bloquea" sin chequear, documentados en sesiones 10-11). Code admitió explícitamente que no confía en su sola voluntad para sostener esto y pidió reforzarlo como una ficha `feedback_` que se cargue cada sesión, con Fer marcándole cuando resbale — no como una promesa de "ser más cuidadosa".

**Pendiente al cierre de esta sesión**: que Code verifique de forma definitiva y rigurosa si `boot` corre en real o demo (la propia Code señaló que ni siquiera esa aclaración está 100% verificada contra la cuenta real), antes de ejecutar el resto del plan.
