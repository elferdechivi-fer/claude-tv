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


2026-07-22 (sesion 5): Envie la aclaracion de Fer sobre el mecanismo EMA200 a Claude Code. Claude Code reviso su hipotesis anterior:

- Con el mecanismo correcto (mensaje llega cada 1 min, pero el valor que trae solo cambia cada 30 min al recalcularse en ese timeframe), Claude Code retiro parcialmente la conexion que habia hecho con el hueco de heartbeat de 12.6 horas ya documentado: si el mensaje llega cada minuto sin falta, un valor desactualizado hasta 30 min no alcanza para explicar un hueco de 12.6 horas por si solo. Sugiere que ese hueco podria ser el filtro funcionando correctamente durante un tramo bajista real y sostenido, no necesariamente un bug.
- - - Lo que si queda firme: cada voto de clon4 se evalua contra un valor de EMA200 que puede tener hasta 30 minutos de desactualizacion, lo cual alcanza para explicar el desfase de consenso de horas entre boot y clon4 en ventanas cortas.
    - - - Claude Code actualizo su propia memoria del proyecto (project_pine_script_hallazgos.md) para reflejar esta correccion.
        - - - Pendiente: preguntarle a Fer si el timeframe de 30m del EMA200 fue diseno intencional o descuido, antes de tocar nada en Pine.
            - 


2026-07-22 (sesion 6): Fer hablo directamente con Claude Code en su sesion compartida sobre el EMA200/clon4. Resumen de lo que verifique alli:

- Fer le conto a Code que la idea de clon14 nacio justamente para reemplazar a clon4 por este problema, y le pidio verificar en la base de datos real si el valor de EMA200 llega cada 1 minuto o no (en vez de suponer).
- - Code reviso la tabla historica real de EMA200 de clon4: encontro que en los datos viejos la mediana real de llegada era 60 minutos (no 1 minuto), con el mismo hueco de 12.6 horas ya documentado como bug de conectividad.
  - - Fer aclaro que esa info era vieja y pidio revisar el ultimo dia. Code confirmo: en las ultimas 24h ya no hay huecos grandes (maximo 89 minutos entre registros), la confiabilidad de entrega se reparo. Pero la frecuencia real sigue siendo aproximadamente cada 1 hora, no cada 1 minuto.
    - - Fer razono con Code: un EMA200 no tendria cambios importantes cada 1 minuto, por eso probablemente se decidio que la cadencia fuera cada 1 hora. Code coincidio: un EMA200 sobre velas de 30m no puede cambiar mas seguido que cada 30m, asi que mandarlo cada 1 minuto seria puro ruido repetido; pasar el heartbeat a 1h es una decision de diseno razonable para un filtro de regimen amplio, no algo que necesite precision al segundo.
      - - Conclusion que Code propuso guardar como cierre del hallazgo (no confirmada aun por Fer en el momento en que revise esto): frecuencia horaria = diseno correcto; el problema real y ya reparado era la confiabilidad de entrega (el hueco de 12.6h), no la frecuencia; sin conexion adicional con el hueco viejo salvo que era el mismo bug de conectividad, ya resuelto.
        - - No participe de esa conversacion puntual entre Fer y Code (fue directa entre ellos), solo la lei para mantenerme al tanto y dejar registro.
          - 


2026-07-22 (sesion 7 - CIERRE EMA200): Verifique directamente en TradingView el Pine script "EMA200 FILTRO N-- CLON4" (v20, activo) y la alerta configurada, en modo solo lectura (sin editar ni guardar nada). Hallazgo tecnico definitivo:
- El script no sondea cada 1 minuto ni cada 30 minutos de forma fija. Dispara alert() en 3 casos: (1) transicion inmediata a condBloqueo (mensaje "ema/no/HEARTBEAT/..."), (2) transicion inmediata a condLibre (mensaje "ema/si/HEARTBEAT/..."), ambos con alert.freq_once_per_bar y reseteando un timer interno; y (3) un heartbeat de respaldo cada 3600000 ms (1 hora) que solo se envia si no hubo cambio de estado real en la ultima hora.
- - La alerta en TradingView esta configurada con condicion "Cualquier llamada a la funcion alert()", intervalo 30m, sin mensaje fijo (el payload lo arma el propio alert() con ema200/distancia).
  - - Confirme esto con Code en su sesion compartida: coincide exactamente con lo que encontro en la base de datos (cadencia horaria estable en el ultimo dia, ya sin el hueco de 12.6h). Code cerro el hallazgo con un matiz importante: como los cambios de estado reales (bloqueo/libre) disparan de inmediato y no esperan al heartbeat horario, un cambio de regimen se refleja casi en tiempo real (atado al cierre de vela de 30m), y no con el retraso de 1 hora que se penso antes. Esto significa que este mecanismo probablemente NO explica por si solo el desfase de consenso de horas entre boot y clon4 investigado antes - eso queda como un misterio aparte, aun sin explicacion completa.
    - - Conclusion: cadencia horaria = diseno correcto e intencional (EMA200 sobre 30m no puede cambiar mas rapido). El problema real y ya reparado era la confiabilidad de entrega (gap de 12.6h), no la frecuencia. TEMA EMA200/CLON4 CERRADO, confirmado por Fer, por mi via inspeccion directa de Pine, y por Code via datos reales de DB.


     ## Sesion 8 - 2026-07-22 - Barrido rapido de alertas y hallazgo CLON12

    Contexto: Fer pidio optimizar los bots. Code pidio un barrido general de que alertas especiales tiene cada bot antes de profundizar en uno solo (candidato clon6, peor rendimiento en dolares). Barrido hecho totalmente de lectura sobre el panel de Alertas de TradingView (81 alertas totales, confirmado global por simbolo, no por layout/pantalla).

    Hallazgos del barrido: clon5 vota en 30m en vez de 1m y clon10 vota en 5m en vez de 1m, siendo los unicos dos bots aparte de EMA200/clon4 con timeframe de voto distinto a 1m. En cuanto a estructura de mensaje, boot, clon3, clon13 y clon14 combinan BB+1 y VI-0 en una sola alerta, mientras que clon2, clon4, clon5, clon6, clon8, clon9 y clon10 los mandan separados en dos alertas. Existe una alerta especial llamada RECOLECTOR 8 en 1 que manda las 8 senales juntas en un mensaje con formato distinto tipo va1/si/RECOLECTOR con todos los valores concatenados. clon2 tiene variantes BB+1 LIMPIO y VI-0 LIMPIO que no existen en ningun otro bot. Tambien hay problemas de nomenclatura ya sospechados: CLON 88 y CLON 99 en vez de clon8 y clon9, y un typo nuevo ARONN con doble N en la alerta Aroon de clon14.

    Hallazgo importante sobre CLON12, que aclara la duda anterior de que clon12 nunca recibio votos: Fer sugirio revisar a que URL de webhook mandan las alertas genericas sin prefijo de bot. Se abrio en modo solo lectura (Cancelar en todo, sin guardar cambios) la configuracion de Notificaciones de cada una de esas alertas genericas y se confirmo que 2 STF CONFIRMACION, OTROS, BB COMPLETO V10.19.7.23, AROON, ENVE V26 y MACD V10 V2 mandan todas su webhook POST al mismo path https alerta punto elferdechivi punto com slash clon12 slash webhook, cada una con un token distinto que no se expone aqui por seguridad. La conclusion es que clon12 si tiene alertas activas, un total de 6, pero estan mal nombradas porque les falta el prefijo CLON12 que usan los demas bots, y por eso el barrido anterior por nombre daba 0 resultados. Dato aparte: la alerta RECOLECTOR 8 en 1 no va a clon12, va a un endpoint distinto (path auditor2), que parece un agregador o auditor separado y no las alertas de voto de un bot especifico.

    Esto se reporto a Fer y a Code en su sesion compartida. Code tenia abierto un pedido de permiso para correr un chequeo de servidor de solo lectura (systemctl mas dos consultas sqlite) para verificar si clon12 recibio votos alguna vez, chequeo que sigue teniendo sentido para confirmar si esas 6 alertas mal nombradas realmente llegan y se procesan del lado del servidor. Ese permiso no fue otorgado por mi, ya que la regla de seguridad establece que solo Fer autoriza acciones con permiso real sobre el servidor, y quedo pendiente de su decision. Como sugerencia de optimizacion, pendiente de aprobacion, convendria renombrar esas 6 alertas para que empiecen con CLON12 como el resto de los bots, para que el monitoreo por nombre sea consistente en todo el sistema; esto implica editar y guardar alertas reales y no se realizo sin autorizacion explicita.


## Actualizacion sesion 8 - Rename de alertas CLON12 y cierre de dos candidatos

Fer autorizo explicitamente renombrar las 6 alertas genericas de clon12. Se hizo en TradingView editando unicamente el campo Nombre de la alerta (accesible desde Editar alerta, Mensaje, boton de volver a Editar mensaje) de cada una: 2 STF CONFIRMACION, OTROS, BB COMPLETO V10.19.7.23, AROON, ENVE V26 y MACD V10 V2 ahora empiezan con el prefijo CLON12 ///, igual que el resto de los bots. No se toco condicion, intervalo, mensaje dinamico ni webhook de ninguna alerta, solo el nombre visible. Verificado con busqueda en el panel de Alertas que las 6 muestran el prefijo nuevo y siguen activas. Reportado a Code.

Code por su lado confirmo del lado del servidor que clon12 esta activo con 10 operaciones reales y 4477 eventos de voto registrados, coincidiendo con el hallazgo de TradingView (nunca estuvo mudo, solo mal nombrado). Tema CLON12 queda cerrado.

Code tambien reviso el punto de BB1/VI0 combinado vs separado (candidato para explicar el desfase de consenso boot/clon4) mirando los mensajes reales que llegan al webhook, y lo descarto: aunque en TradingView BB1 y VI0 aparezcan agrupados bajo un mismo nombre de alerta en algunos bots, lo que llega al servidor son siempre dos mensajes separados y limpios (bb1/si/BB+1 y vi0/si/VI-0), el parser busca el patron id/si y no se confunde. Con esto van dos candidatos descartados (EMA200 y BB1/VI0 combinado) para explicar el misterio original de timing boot/clon4, que sigue sin explicacion completa pero con retorno decreciente de seguir invirtiendo tiempo ahi, segun Code.

Proximo paso propuesto por Code (pendiente de confirmar con Fer): pasar a revisar clon6 en profundidad, por ser el bot con peor rendimiento en dolares de todo el ecosistema (-1374.40 USD) y el mas complejo segun la documentacion del proyecto.


Sesion 8 - continuacion - Verificacion 100 por ciento de alertas y cierre de clon6 y del punto LIMPIO

Fer pidio confirmar si el 100 por ciento de las 81 alertas estaban identificadas. Se hizo un recuento completo scrolleando todo el panel de Alertas (solo lectura) y se confirmo que las 81 caen exactamente en un grupo conocido: BOOT 6, CLON2 7, CLON3 6, CLON4 7 mas su alerta especial EMA200 N CLON4 1, CLON5 7, CLON6 7, CLON 88 osea clon8 7, CLON 99 osea clon9 7, CLON10 7, CLON12 6 ya renombradas, CLON13 6 y CLON14 6, mas RECOLECTOR 8 en 1 que confirmamos antes que va al endpoint auditor2 y no a un bot de voto. La suma da 81 exacto, no sobra ni falta ninguna. Los unicos dos detalles pendientes son puramente de nomenclatura visual, sin impacto funcional: CLON 88 y CLON 99 en vez de CLON8 y CLON9, y el typo ARONN con doble N en CLON14 y CLON3.

Code reviso clon6 del lado del servidor antes de pedirme que mire Pine, y encontro que el problema ya estaba diagnosticado en el propio CONFIG.py del bot: concentracion de riesgo en la quinta compra del DCA, el mismo patron ya visto en boot y clon2, sin ningun filtro de Pine escondido de por medio. Ya existe un retro-test validado de antes que corta MAX_COMPRAS de 5 a 4 y mejora el resultado de clon6 en 291.52 dolares, de -1374.40 a -1082.88, siguiendo negativo pero mejor. Ese cambio nunca se aplico en produccion. Code le va a preguntar a Fer si autoriza aplicarlo, por ser un cambio de codigo real en el servidor; no es algo que yo pueda aprobar ni que se decida sin Fer.

Mientras tanto Code me sugirio revisar el punto 5 pendiente de la lista original, las variantes BB+1 LIMPIO y VI-0 LIMPIO que solo existen en clon2. Encontre que son versiones simplificadas de un solo umbral, sin las demas confirmaciones que tiene la logica normal de BB+1 y VI-0. Revise en modo solo lectura el webhook de ambas alertas y confirme que las dos apuntan correctamente a clon2/webhook con token propio. Encontre tambien un detalle cosmetico sin impacto real: el archivo Pine de BB+1 LIMPIO tiene comentarios internos viejos que dicen CLON3, y el archivo BB+1 usado por clon4 tiene su indicator() todavia titulado BB+1 visual boot; son resabios de copiar y pegar de versiones anteriores, no afectan el webhook ni la logica de ninguno de los dos bots. Reportado a Code, que dio por cerrado este punto sin cambios pendientes.

Queda pendiente la decision de Fer sobre aplicar el corte de MAX_COMPRAS en clon6.
