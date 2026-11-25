# Guardia del Lab 6 · Postmortems sin culpables · E041

Resumen de la guardia del 25/11/2026 sobre la base `oralab-26ai` (contenedor `e38ca20e9401`, DBID `1509902213`). Como norma de trabajo, todo el diagnóstico se realizó con el usuario de solo lectura `C##OPS_RO` (`scripts/lab6/08-diagnostico.sql`), reservando SYSDBA únicamente para ejecutar la sentencia de intervención. Todas las evidencias están guardadas en `docs/bitacora/lab6/evidencia/script/` (archivos desde `20261125T1807…` hasta `20261125T1809…`).

## INC-601 · La pantalla de secretaría se queda colgada
- Detección: A las 18:07 UTC, Ana García (secretaría) abre un ticket avisando de que, al intentar guardar la ficha del alumno 101, la pantalla se le queda colgada.
- Impacto: Ana no pudo guardar los cambios en la ficha del alumno 101 entre las ~18:07:55 y las 18:08:13. Reiniciar su equipo no habría solucionado nada, porque el bloqueo estaba en la base de datos.
- Diagnóstico: Al lanzar `08-diagnostico` a las 18:07:58 vimos enseguida lo que ocurría:
  - En la sección 1, la sesión 176 (ANA_GARCIA, `secretaria`/`ficha-alumno`) estaba esperando en `enq: TX - row lock contention` (clase Application), bloqueada por la sesión 19 (APP_ACADEMIA, `matriculas`/`cierre-periodo`), que se encontraba inactiva en `PL/SQL lock timer` (Idle).
  - La sección 2 confirmó que el conflicto estaba en la tabla `ACA_ESTUDIANTES` (archivo 1024, bloque 780, fila 5).
  - En la sección 3 comprobamos que la sesión 19 retenía el bloqueo TX en modo 6 con `BLOCK = 1`, mientras la 176 lo solicitaba también en modo 6.
  - Por último, la sección 4 mostró que la transacción de la sesión 19 era mínima: apenas 1 bloque y 1 registro de undo.
- Causa raíz: La aplicación de matrículas modificó la fila del alumno 101 y se quedó esperando sin hacer commit ni rollback. Al dejar la transacción abierta, mantuvo bloqueada la fila mientras la sesión no hacía nada.
- Intervención: A las 18:08:13 ejecutamos `ALTER SYSTEM KILL SESSION '19,31982' IMMEDIATE;`. Era el cambio más quirúrgico posible: solo cerrábamos una sesión (sin tocar la instancia) y únicamente se deshacía 1 fila. Usar `CANCEL SQL` no habría servido, ya que cancela la llamada en curso pero deja vivos tanto la transacción como el bloqueo (Parte D).
- Validación: En `08-incidente1-intervencion` comprobamos que ya había 0 sesiones bloqueadas y que el alumno 101 pasó a estado `ACTIVO` (el UPDATE de Ana se completó). El `08-diagnostico` posterior (18:08:21) salió limpio de esperas, y ASH dejó registrada la espera de la sesión 176 hasta las 18:08:12.
- Acción preventiva: Asegurar que la aplicación confirme o deshaga siempre antes de entrar en cualquier espera (nunca dejar una transacción abierta durante un SLEEP) y configurar una alerta cuando una sesión lleve más de N segundos bloqueando a otra.

## INC-602 · La aplicación no conecta: ORA-12514
- Detección: A las 18:08 UTC, el equipo de matrículas avisa de que ninguna instancia de la aplicación consigue conectar (error ORA-12514) y pregunta si la base de datos está caída.
- Impacto: Ninguna instancia de matrículas pudo conectarse a través del servicio `academia_app` desde las 18:08 hasta aproximadamente las 18:09:02, momento en que se arrancó el servicio.
- Diagnóstico: Empezamos pasando la lista de comprobación del turno (`08-incidente2-turno`, 18:08:34): la instancia estaba OPEN, con LOGINS ALLOWED, la base en READ WRITE y FREEPDB1 abierta. Quedaba descartado que la base estuviera caída. En el `08-diagnostico` de las 18:08:38 (sección 6) vimos el problema: el servicio `academia_app` existía pero figuraba con ARRANCADO = NO, y `lsnrctl status` solo anunciaba `freepdb1`. Reprodujimos el fallo desde el lado del cliente con `app.sh`, obteniendo el mensaje exacto: «Service academia_app is not registered with the listener».
- Causa raíz: El servicio `academia_app` estaba detenido. El error ORA-12514 lo devuelve el listener porque no conoce ese servicio, no porque la base de datos esté caída.
- Intervención: Desde FREEPDB1, arrancamos el servicio con `EXEC DBMS_SERVICE.START_SERVICE('academia_app')` y guardamos el estado con `ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;` para garantizar que arranque automáticamente tras futuros reinicios (Parte C). No hizo falta reiniciar nada.
- Validación: Según `08-incidente2-intervencion`, el listener volvió a anunciar `academia_app` en estado READY y `app.sh` conectó sin problemas por `academia_app` (FREEPDB1, APP_ACADEMIA). El `08-diagnostico` posterior confirmó ARRANCADO = SI.
- Acción preventiva: Actualizar la lista de turno para que verifique que todos los servicios definidos están arrancados y avise si alguno no lo está. Además, guardar siempre el estado de la PDB cada vez que se realicen cambios en sus servicios.

## INC-603 · La aplicación no entra: ORA-28000
- Detección: A las 18:09 UTC, el equipo de matrículas informa de que, tras un despliegue, la aplicación devuelve ORA-28000 pese a que no han tocado la contraseña. En las notas del turno de tarde consta que hoy se ejecutó el script trimestral de bajas.
- Impacto: La aplicación no pudo iniciar sesión con la cuenta APP_ACADEMIA desde las 18:09:22 hasta que se procedió a su desbloqueo.
- Diagnóstico: En el `08-diagnostico` de las 18:09:22 (sección 7) vimos que APP_ACADEMIA figuraba como `LOCKED` (y no `LOCKED(TIMED)`) desde las 18:09, bajo el perfil DEFAULT (que permite hasta 10 intentos fallidos). Si el bloqueo hubiera sido automático por contraseñas erróneas, habría aparecido como `LOCKED(TIMED)`, así que la cuenta no se bloqueó sola. La auditoría unificada (`08-incidente3-intervencion`) nos dio la confirmación definitiva: SYS había ejecutado `ALTER USER app_academia ACCOUNT LOCK` a las 18:09:22.
- Causa raíz: El script trimestral de bajas no distingue entre cuentas personales y cuentas de aplicación, por lo que acabó bloqueando el usuario de la aplicación.
- Intervención: Tras guardar primero la evidencia de auditoría, ejecutamos `ALTER USER app_academia ACCOUNT UNLOCK;` en FREEPDB1. No tocamos la contraseña: la aplicación ya tiene la correcta, así que nos ahorramos cualquier redespliegue.
- Validación: La cuenta volvió a estado `OPEN` y `app.sh` pudo entrar de nuevo por `academia_app` (FREEPDB1, APP_ACADEMIA). El `08-diagnostico` posterior confirmó que ya no quedaban cuentas bloqueadas.
- Acción preventiva: Limitar el script de bajas para que solo actúe sobre cuentas con el perfil de personas (`PRF_PERSONAS`), y asignar un perfil propio a las cuentas de aplicación para que queden excluidas siempre.
