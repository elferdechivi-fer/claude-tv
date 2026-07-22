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
