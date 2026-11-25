He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 6 · E041

## Entrada 1
- **Herramienta:** Claude Code (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
Estoy haciendo una guardia en el Lab 6 de Oracle 26ai (CDB FREE, PDB FREEPDB1, contenedor Docker).
Ticket: Ana García, de secretaría, dice que al guardar la ficha del alumno 101 la pantalla se queda colgada.

Mi diagnóstico de solo lectura (08-diagnostico.sql con C##OPS_RO) muestra:
- La sesión 176 (ANA_GARCIA) espera en "enq: TX - row lock contention", clase Application.
- BLOCKING_SESSION = 19. La sesión 19 es APP_ACADEMIA, serial 31982, módulo "matriculas", acción "cierre-periodo".
- La 19 está dormida (evento "PL/SQL lock timer", Idle) y tiene una transacción abierta con 1 registro de undo.
- V$LOCK: la 19 tiene el TX en lmode 6 con block 1; la 176 lo pide con request 6.
- La fila bloqueada es de ADMIN_ACADEMIA.ACA_ESTUDIANTES (archivo 1024, bloque 780, fila 5).

Necesito:
1. La sentencia mínima, como SYSDBA, para liberar el bloqueo sin tocar la instancia ni otras sesiones.
2. Por qué esa y no ALTER SYSTEM CANCEL SQL, y qué pasa con la transacción de la sesión 19.
3. Dos consultas de validación: cuántas sesiones siguen bloqueadas en la instancia, y el estado de la fila
   del alumno 101 en admin_academia.aca_estudiantes (ojo: la tabla está en FREEPDB1 y yo conecto en la raíz).

Explícame cada línea.
```
(Contexto: se lo pregunté en la guardia de la Parte I, después de que el diagnóstico 08 mostrara que la sesión 19 de APP_ACADEMIA bloqueaba a la 176 de ANA_GARCIA en la fila 101.)
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
ALTER SYSTEM KILL SESSION '19,31982' IMMEDIATE;

SELECT COUNT(*) AS sesiones_bloqueadas FROM v$session WHERE blocking_session IS NOT NULL;
ALTER SESSION SET CONTAINER = FREEPDB1;
SELECT estudiante_id, estado FROM admin_academia.aca_estudiantes WHERE estudiante_id = 101;
```
- **Dónde lo usé (parte, script o pregunta):** Parte I, incidente INC-601 (intervención y validación)
- **Propósito (para qué lo necesitaba):** quitar el bloqueo que dejaba colgada la ficha de Ana y comprobar que se resolvía
- **Qué hace (con mis palabras; línea a línea si es SQL corto):**
  1) KILL SESSION termina solo la sesión 19 de APP_ACADEMIA, que tenía la fila bloqueada (TX en modo 6) y hacía esperar a Ana (SID 176). Pongo SID y serial (19,31982) para no matar otra sesión que hubiera reutilizado el SID 19. Con IMMEDIATE Oracle no espera al cliente: deshace la transacción sin confirmar de la 19 (su registro de undo) y suelta el bloqueo; ese cambio se pierde y la aplicación tendrá que repetir el «cierre-periodo». Usé KILL y no CANCEL SQL porque la 19 estaba dormida sin ejecutar nada: CANCEL solo corta la llamada en curso, deja la transacción abierta y la fila seguiría bloqueada;
  2) cuenta, en toda la instancia y desde la raíz, las sesiones que esperan a otra: 0, ya no hay bloqueos;
  3) cambio a FREEPDB1, porque la tabla está en la PDB y yo conecto en la raíz;
  4) miro la fila 101: está ACTIVO, así que la espera de Ana terminó y su cambio se guardó
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** docs/bitacora/lab6/evidencia/script/20261125T180813Z_08-incidente1-intervencion.script.log

## Entrada 2
- **Herramienta:** Claude Code (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
Sigo con la guardia del Lab 6 (Oracle 26ai, CDB FREE, PDB FREEPDB1).
Ticket: el equipo de matrículas dice que ninguna instancia de la aplicación conecta y el error es ORA-12514.
Preguntan si la base está caída.

Lo que he comprobado:
- Mi lista de turno dice: instancia OPEN, LOGINS ALLOWED, base READ WRITE, FREEPDB1 en READ WRITE.
- La sección 6 del diagnóstico (CDB_SERVICES frente a V$ACTIVE_SERVICES) muestra academia_app con ARRANCADO = NO.
- lsnrctl status solo anuncia el servicio freepdb1, no academia_app.
- Reproduzco el error con app.sh, que conecta como APP_ACADEMIA por //localhost:1521/academia_app.

Necesito:
1. El SQL, como SYSDBA, para arrancar academia_app sin reiniciar la instancia ni la PDB.
2. Que el servicio siga arrancado después del próximo reinicio de la instancia.
3. Una consulta que lanzaré con app.sh para comprobar por qué servicio, en qué contenedor
   y con qué usuario entra la aplicación.

Explícame por qué el ORA-12514 lo da el listener y no la base, y qué hace cada línea.
```
(Contexto: se lo pregunté en la guardia de la Parte I, después de que el diagnóstico 08 mostrara academia_app con ARRANCADO = NO y lsnrctl solo anunciara freepdb1.)
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
EXEC DBMS_SERVICE.START_SERVICE('academia_app')
ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;

SELECT SYS_CONTEXT('USERENV', 'SERVICE_NAME') AS servicio, SYS_CONTEXT('USERENV', 'CON_NAME') AS contenedor, USER AS usuario FROM dual;
```
- **Dónde lo usé (parte, script o pregunta):** Parte I, incidente INC-602 (intervención y validación con app.sh)
- **Propósito (para qué lo necesitaba):** que el listener volviera a anunciar academia_app sin reiniciar nada, y que siguiera arrancado después de un reinicio
- **Qué hace (con mis palabras; línea a línea si es SQL corto):**
  El ORA-12514 lo da el listener, no la base: la petición llega al listener pidiendo academia_app y él solo conoce los servicios que la instancia le ha registrado. La base estaba abierta (instancia OPEN, FREEPDB1 READ WRITE en mi lista de turno), pero el servicio estaba definido y parado (ARRANCADO = NO), así que nadie lo anunciaba y lsnrctl solo mostraba freepdb1.
  1) me pongo en FREEPDB1, donde está definido el servicio;
  2) START_SERVICE arranca academia_app sin reiniciar la instancia ni la PDB; la instancia lo registra y el listener lo vuelve a anunciar;
  3) SAVE STATE guarda el estado de la PDB con el servicio ya arrancado, para que tras el próximo reinicio vuelva igual (como en la Parte C);
  4) esta consulta la lancé con app.sh, que entra como la aplicación por academia_app, y me dice por qué servicio, en qué contenedor y con qué usuario entra: academia_app, FREEPDB1, APP_ACADEMIA
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** docs/bitacora/lab6/evidencia/script/20261125T180902Z_08-incidente2-intervencion.script.log

## Entrada 3
- **Herramienta:** Claude Code (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
Tercer incidente de la guardia del Lab 6 (Oracle 26ai, PDB FREEPDB1).
Ticket: tras el despliegue la aplicación no entra y da ORA-28000; dicen que no han cambiado la contraseña.
Nota del turno: hoy se ejecutó el script trimestral de bajas.

Mi diagnóstico muestra:
- APP_ACADEMIA en estado LOCKED (no LOCKED(TIMED)), con la fecha de bloqueo de hoy.
- Su perfil es DEFAULT, con FAILED_LOGIN_ATTEMPTS = 10.

Antes de tocar nada quiero preservar la evidencia. Necesito:
1. Una consulta sobre UNIFIED_AUDIT_TRAIL, dentro de FREEPDB1, que me dé el último ALTER USER que bloqueó
   APP_ACADEMIA: hora, usuario de base de datos, acción, código de retorno y los primeros 60 caracteres del SQL.
2. Una consulta que una DBA_USERS y DBA_PROFILES y me muestre el estado de APP_ACADEMIA, su perfil
   y su límite de FAILED_LOGIN_ATTEMPTS, para descartar que se bloqueara sola.
3. La sentencia mínima para desbloquearla sin cambiar la contraseña, y una consulta para comprobar que queda OPEN.

Explícame la diferencia entre LOCKED y LOCKED(TIMED) y qué hace cada línea.
```
(Contexto: se lo pregunté en la guardia de la Parte I, después de que el diagnóstico 08 mostrara APP_ACADEMIA en LOCKED y app.sh devolviera ORA-28000.)
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
SELECT TO_CHAR(event_timestamp, 'YYYY-MM-DD HH24:MI:SS') AS hora, dbusername, action_name, return_code,
       CAST(SUBSTR(sql_text, 1, 60) AS VARCHAR2(60)) AS sql_text
  FROM unified_audit_trail
 WHERE action_name = 'ALTER USER' AND UPPER(sql_text) LIKE '%APP_ACADEMIA%LOCK%'
 ORDER BY event_timestamp DESC FETCH FIRST 1 ROWS ONLY;
SELECT u.username, u.account_status, u.profile, p.limit AS failed_login_attempts
  FROM dba_users u JOIN dba_profiles p ON p.profile = u.profile AND p.resource_name = 'FAILED_LOGIN_ATTEMPTS'
 WHERE u.username = 'APP_ACADEMIA';

ALTER USER app_academia ACCOUNT UNLOCK;
SELECT username, account_status FROM dba_users WHERE username = 'APP_ACADEMIA';
```
- **Dónde lo usé (parte, script o pregunta):** Parte I, incidente INC-603 (evidencia, intervención y validación)
- **Propósito (para qué lo necesitaba):** saber quién bloqueó la cuenta de la aplicación y cuándo antes de tocarla, y desbloquearla sin cambiar la contraseña
- **Qué hace (con mis palabras; línea a línea si es SQL corto):**
  0) me pongo en FREEPDB1, porque la auditoría de la cuenta está en la PDB;
  1) busca en la auditoría unificada el último ALTER USER que bloqueó APP_ACADEMIA, con hora, usuario, acción, código de retorno y los primeros 60 caracteres del SQL: lo hizo SYS a las 18:09:22 con ACCOUNT LOCK y código 0, es decir, el script de bajas y no unos intentos fallidos. Lo guardo antes de tocar nada para no perder la evidencia;
  2) une la cuenta con su perfil: LOCKED (no LOCKED(TIMED)) con perfil DEFAULT y 10 intentos, así que no pudo bloquearse sola. LOCKED(TIMED) es el bloqueo automático por superar FAILED_LOGIN_ATTEMPTS, que se quita solo pasado PASSWORD_LOCK_TIME; LOCKED a secas es un ACCOUNT LOCK manual y dura hasta que alguien hace UNLOCK;
  3) ACCOUNT UNLOCK quita solo el bloqueo; la contraseña no la toco porque la aplicación la tiene configurada y nadie la había cambiado;
  4) compruebo que la cuenta ha quedado OPEN; después app.sh entra por academia_app
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** docs/bitacora/lab6/evidencia/script/20261125T180944Z_08-incidente3-intervencion.script.log

## Entrada 4
- **Herramienta:** Claude Code (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
hazme un borrador y lo reviso yo
```
(Contexto: se lo pedí después de terminar los tres incidentes, para los postmortems de docs/bitacora/lab6-guardia.md.)
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
no aplica
```
- **Dónde lo usé (parte, script o pregunta):** Parte I, docs/bitacora/lab6-guardia.md (los tres postmortems)
- **Propósito (para qué lo necesitaba):** tener una primera versión de los tres postmortems con la plantilla de I.5 y los datos de mis evidencias 08
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** me dio un borrador de texto de los postmortems. Lo comparé con mis evidencias y lo reescribí con mis palabras; lo que entrego es mi versión
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** no aplica

## Entrada 5
- **Herramienta:** Claude Code (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
si empezamos con la parte J
```
(Contexto: con este prompt empezamos la Parte J; la IA preparó un borrador de las respuestas que después reescribí.)
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
no aplica
```
- **Dónde lo usé (parte, script o pregunta):** Parte J, docs/bitacora/lab6-respuestas.md
- **Propósito (para qué lo necesitaba):** tener una primera versión de las respuestas con la evidencia de la que sale cada cifra
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** me dio un borrador de texto de las respuestas indicando de qué evidencia sale cada dato. Comprobé las cifras en mis evidencias y reescribí las respuestas con mis palabras; lo que entrego es mi versión
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** no aplica

## Entrada 6
- **Herramienta:** Claude Code (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
Estoy respondiendo las preguntas del Lab 6 (Oracle 26ai, CDB FREE).
Mi lista de turno, ejecutada el 2026-11-25 a las 18:03:21 UTC, contó 17 errores ORA- en el alert log
en las últimas 24 horas, y quiero saber qué errores son.
Además, en la Parte F lancé una carga reanudable (sesión 'carga-historico') que se suspendió por ORA-01536
y se reanudó al ampliar la cuota, entre las 18:05:50 y las 18:06:30 UTC del 25/11. Quiero las horas exactas.

Necesito dos consultas sobre V$DIAG_ALERT_EXT:
1. Para la ventana del 2026-11-24 18:03:21 al 2026-11-25 18:03:21 UTC, solo los mensajes que contengan "ORA-":
   extraer el código ORA-NNNNN y contar cuántas veces aparece cada uno, de más a menos.
2. Para la ventana de 18:05:50 a 18:06:30 UTC del 25/11, los mensajes que contengan "resumable"
   (sin distinguir mayúsculas), con la hora en milisegundos.

Explícame qué es V$DIAG_ALERT_EXT y qué hace cada línea.
```
(Contexto: se lo pregunté al preparar las respuestas 1 y 13 de la Parte J, para desglosar los errores ORA- de mi lista de turno y fechar la carga reanudable de la Parte F.)
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
SELECT REGEXP_SUBSTR(message_text, 'ORA-[0-9]+') error, COUNT(*) veces
FROM v$diag_alert_ext
WHERE originating_timestamp BETWEEN TIMESTAMP '2026-11-24 18:03:21 +00:00'
                                AND TIMESTAMP '2026-11-25 18:03:21 +00:00'
  AND message_text LIKE '%ORA-%'
GROUP BY REGEXP_SUBSTR(message_text, 'ORA-[0-9]+') ORDER BY 2 DESC;

SELECT TO_CHAR(originating_timestamp, 'YYYY-MM-DD HH24:MI:SS.FF3') ts, message_text
FROM v$diag_alert_ext
WHERE originating_timestamp BETWEEN TIMESTAMP '2026-11-25 18:05:50 +00:00'
                                AND TIMESTAMP '2026-11-25 18:06:30 +00:00'
  AND LOWER(message_text) LIKE '%resumable%';
```
- **Dónde lo usé (parte, script o pregunta):** Parte J, preguntas 1 y 13 de lab6-respuestas.md
- **Propósito (para qué lo necesitaba):** saber qué errores formaban los 17 ORA- de mi lista de turno y a qué hora exacta se suspendió y se reanudó la carga reanudable de la Parte F
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** V$DIAG_ALERT_EXT es el alert log en forma de vista: lee el log.xml del ADR y da una fila por mensaje, con su hora (originating_timestamp) y su texto (message_text), así que puedo filtrarlo con SQL en vez de leer el fichero.
  1) la primera consulta fija la ventana en UTC con TIMESTAMP '... +00:00' (las 24 h anteriores a mi lista de turno, 18:03:21 del 25/11), se queda con los mensajes que tienen «ORA-», saca el código con REGEXP_SUBSTR y con GROUP BY y ORDER BY 2 DESC cuenta cuántas veces sale cada uno, de más a menos: 12 ORA-00800, 2 ORA-65019, 1 ORA-01536, 1 ORA-00031 y 1 ORA-00000 (el mensaje del KILL SESSION), que son los 17. Los tres últimos son de la primera pasada del 24/11 por la noche, por eso caen dentro de la ventana;
  2) la segunda busca entre las 18:05:50 y las 18:06:30 los mensajes con «resumable» (LOWER para no distinguir mayúsculas) y saca la hora con FF3, en milésimas: suspendida a las 18:06:03.323 y reanudada a las 18:06:17.429, unos 14 s
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** docs/bitacora/lab6/evidencia/script/20261125T184746Z_09-alert-log.script.log
