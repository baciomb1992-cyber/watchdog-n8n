# Watchdog externo de n8n (REC-007 del Sistema Vigilante)

Vigila desde GitHub Actions que la instancia de n8n responda, cada 5 minutos. Si no responde 3 veces seguidas, la reinicia vía la API de EasyPanel, reverifica y avisa por Telegram. Vive fuera de n8n a propósito: todo lo demás del Vigilante corre dentro de n8n y no puede levantarlo si se cae.

- Avisa **solo cuando actúa**. Un check exitoso solo queda en el log de la corrida.
- **Nace en modo sombra** (`MODO_SOMBRA=true`): detecta y avisa lo que habría hecho, sin reiniciar. Se pasa a real cambiando esa variable del repo.
- Idempotente: una sola corrida a la vez (`concurrency`) y cooldown consultando `vigilante_autofix_log` (no reinicia dos veces en `COOLDOWN_MIN` minutos).
- Sin credenciales en el código: todo va en Secrets y Variables del repo.

Documentación completa en `projects/sistema-vigilante/` del repo Asistente_Empresarial.

## PARA TODO (spec 001 del Sistema Vigilante)

Antes de reiniciar n8n, cada corrida lee el interruptor **PARA TODO** (`vigilante_config.autofix_pausado`) en el paso «Leer interruptor PARA TODO»:

- **Prendido** → no reinicia; deja constancia (`estado='pausado'`, `modo='ninguno'`, `resultado.motivo='PARA TODO'`) y avisa por Telegram («PAUSADO»). El enfriamiento (`COOLDOWN_MIN`) hace que sea **un aviso por enfriamiento** mientras dure la caída. Al apagar la pausa, reinicia en la siguiente corrida si n8n sigue caído, sin intervención.
- **Apagado** → como siempre.
- **No se puede leer** (Supabase no responde o la fila no existe) → usa el **último valor leído**: busca hacia atrás, en los logs de corridas previas, la marca que imprime cada lectura sana. Si hay valor recordado `false` actúa y el aviso lo dice; si es `true`, o **nunca se ha leído** → no reinicia y avisa (`pausado_ilegible`).
- Marcas: las corridas reales imprimen `PAUSA=<valor>`; las de prueba, `PAUSA_PRUEBA=<valor>`. Cada modo solo lee las suyas: una prueba nunca contamina el valor recordado de producción. (En el log aparecen con un prefijo técnico; el YML las arma con variables para que el fuente del paso impreso no dé falso positivo.)

### Limitación documentada: cuánto recuerda

La búsqueda recorre como máximo `MAX_CORRIDAS_MEMORIA = 60` corridas completadas hacia atrás (variable de entorno del propio YML, no es un secret ni una variable del repo) = **unas 5 h de cron** (12 corridas/h). Si la base lleva caída más de ~5 h sin una sola lectura sana en esa ventana, REC-007 deja de actuar y avisa «sin valor recordado» hasta que la base vuelva (decisión: desconocido → no actuar y avisar). El «≈ 8 h» del plan era optimista; se midió (spec 001, T8, corridas `37493364110` y `37493713018` en modo prueba):

- Leer el log de una corrida previa cuesta **0,8–1,1 s** (100 corridas en 77 s y en 111 s) y **1 llamada** contada a la API por corrida (102 llamadas por 100 corridas).
- El paso corta a los 170 s y el job tiene `timeout-minutes: 8`: 100 corridas caben en tiempo, pero **una caída continua** repite la búsqueda en las 12 corridas de cada hora: con 100 serían ~1 220 llamadas/h, por encima del límite documentado de 1 000/h por repositorio para el `GITHUB_TOKEN` (la cabecera `x-ratelimit-limit` del runner mostró 5 000; se toma el documentado, más conservador).
- Con **60**: ~46–67 s por búsqueda y 12 × 62 ≈ **744 llamadas/h** en el peor caso (74 % del límite documentado; 15 % del que mostró el runner).

### Modo prueba (solo `workflow_dispatch`)

Inputs nuevos, todos opcionales y con valor por omisión «normal» (el cron externo no los usa):

| Input | Para qué |
|---|---|
| `modo_prueba` (`true`/`false`, def. `false`) | `true`: **nunca llama a EasyPanel** (en vez de reiniciar imprime `HABRIA REINICIADO (modo prueba)` y termina `sombra_prueba`), imprime `PAUSA_PRUEBA=` y antepone `[PRUEBA] ` a todo Telegram (que además queda impreso en el log). |
| `tabla_config` (def. `vigilante_config`) | tabla del interruptor. Con `modo_prueba=true` debe contener `_prueba`; si no, «Validar configuración» falla. |
| `tabla_log` (def. `vigilante_autofix_log`) | bitácora de intentos y del enfriamiento. Misma regla. |
| `forzar_estado` | además de `caido` e `inalcanzable`: `caido_persistente` (fuerza también la reconfirmación a «caído»; el VPS sigue alcanzable) y `sin_conexion_persistente` (fuerza la reconfirmación a HTTP 000 y que tampoco se alcance EasyPanel), **solo con `modo_prueba=true`**. |

Fuera de modo prueba, cambiar las tablas o usar `caido_persistente` / `sin_conexion_persistente` también falla en «Validar configuración». Ejemplo:

```
gh workflow run watchdog-n8n.yml --ref spec-001-para-todo -f modo_prueba=true -f forzar_estado=caido_persistente \
  -f tabla_config=vigilante_config_prueba -f tabla_log=vigilante_autofix_log_prueba
```

Las pruebas viven en `projects/sistema-vigilante/herramientas/pruebas/casos_rec007.py` del repo Asistente_Empresarial. No agrega secrets ni variables al repo.

### Con la pausa prendida: qué pasa según lo que ve la reconfirmación (vuelta 1 de la spec 001)

La pausa se evalúa **después** de que la reconfirmación final distingue el caso (paso «Reiniciar n8n vía EasyPanel»):

| Reconfirmación | Pausa prendida / ilegible sin valor recordado |
|---|---|
| HTTP 200 | `recuperado_solo`, no se toca nada |
| **HTTP 000** (ni conecta) | se **conserva el camino de siempre**: se sondea EasyPanel de solo lectura (un `GET`, sin reiniciar nada). Si tampoco conecta → `restart_sin_conexion` (regla de 2 strikes, aviso 🟠 «No alcanzo el VPS»; la pausa **no** interviene y **no** se avisa «n8n no respondió… PARA TODO»). Si EasyPanel sí responde, el VPS es alcanzable y aquí es donde se habría reiniciado → `pausado` (o `pausado_ilegible`) |
| HTTP real 4xx/5xx | como antes: `pausado` / `pausado_ilegible` |

Con la pausa **apagada** nada cambia: sigue el reinicio normal y, si EasyPanel tampoco conecta, `restart_sin_conexion`.

El aviso de Telegram, cuando el interruptor no se pudo leer y se usó el valor recordado, **siempre** (en `pausado`, `sombra`, `resuelto`…) lleva: «No pude leer el interruptor PARA TODO (Supabase no responde); actué con el último valor leído: true/false».

### Un solo aviso de PARA TODO aunque la bitácora esté caída: los logs como memoria

El enfriamiento normal se lee de la bitácora (`vigilante_autofix_log`). Si la bitácora no responde (no se puede leer el enfriamiento o falla el `POST` de la constancia), el paso «Memoria de la bitácora» usa los **logs de las corridas previas** (misma técnica que el valor recordado del interruptor), dentro de `COOLDOWN_MIN`:

- Cuando se avisa un `pausado` / `pausado_ilegible` **con la bitácora degradada**, la corrida imprime `PAUSADO_AVISADO=<epoch>` (en prueba, `PAUSADO_AVISADO_PRUEBA=`). La siguiente corrida, si la bitácora sigue degradada y encuentra esa marca dentro de `COOLDOWN_MIN`, **no repite el Telegram** y escribe «constancia pendiente de la bitácora, aviso ya dado». Resultado: un aviso por caída/enfriamiento, nunca uno cada 5 min.
- La constancia que no se pudo escribir se imprime como `CONSTANCIA_PENDIENTE=<json corto>` (con la tabla y la fila). **Al inicio de cada corrida**, si hay una pendiente (de la misma tabla) y la bitácora ya responde, se escribe **primero** (con su hora original, así también cuenta para el enfriamiento) y se imprime `CONSTANCIA_REPUESTA=<hora original>` para no repetirla. Mientras la bitácora siga caída, la pendiente se vuelve a imprimir en cada corrida y vive en los logs.
- Límite: la memoria es la ventana de `COOLDOWN_MIN` (15 min hoy). Una constancia pendiente más vieja que eso y sin bitácora de vuelta se pierde del log (el aviso de Telegram ya se dio; la hora y el motivo van también en ese mensaje).
- Cuando la bitácora funciona nada de esto actúa: el enfriamiento sigue siendo la fila de la bitácora.

Las marcas se arman con variables en el YML (GitHub imprime el fuente de cada paso; una marca literal daría un falso positivo en cada corrida).
