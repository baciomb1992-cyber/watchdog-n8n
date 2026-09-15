# Watchdog externo de n8n (REC-007 del Sistema Vigilante)

Vigila desde GitHub Actions que la instancia de n8n responda, cada 5 minutos. Si no responde 3 veces seguidas, la reinicia vía la API de EasyPanel, reverifica y avisa por Telegram. Vive fuera de n8n a propósito: todo lo demás del Vigilante corre dentro de n8n y no puede levantarlo si se cae.

- Avisa **solo cuando actúa**. Un check exitoso solo queda en el log de la corrida.
- **Nace en modo sombra** (`MODO_SOMBRA=true`): detecta y avisa lo que habría hecho, sin reiniciar. Se pasa a real cambiando esa variable del repo.
- Idempotente: una sola corrida a la vez (`concurrency`) y cooldown consultando `vigilante_autofix_log` (no reinicia dos veces en `COOLDOWN_MIN` minutos).
- Sin credenciales en el código: todo va en Secrets y Variables del repo.

Documentación completa en `projects/sistema-vigilante/` del repo Asistente_Empresarial.
