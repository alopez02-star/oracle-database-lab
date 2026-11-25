# Lab 6 — Oracle DBA Operations · Respuestas · E041

Autor: Antonio López Ros

Todas las cifras de este documento salen directamente de mis evidencias en `docs/bitacora/lab6/evidencia/` (ejecución del 25/11/2026, contenedor `e38ca20e9401`, DBID `1509902213`). En los casos en los que mis datos no coinciden con los del enunciado, lo dejo indicado expresamente.

## 9. Preguntas de comprobación

### 9.1. Partes A-C · Rol operativo, parámetros y estados

**1. Errores ORA- del alert log en 24 h (Parte B).**
La lista de turno (`20261125T180321Z_01-turno.script.log`, sección 4) registró **17** errores ORA- en las últimas 24 horas, frente a los 202 que indica el enunciado. Al agruparlos con `V$DIAG_ALERT_EXT` para esa misma ventana salen: 12 × ORA-00800 («Set Priority Failed» de LGWR/VKTM), 2 × ORA-65019, 1 × ORA-01536, 1 × ORA-00031 y 1 mensaje de KILL SESSION. Los cinco últimos los provoqué yo mismo en la primera vuelta del laboratorio (el 24/11), antes de restaurar la copia `lab6-inicio`, ya que el alert log no se restaura junto con la base. Por su parte, ORA-00800 es ruido habitual en contenedores: Docker no permite a Oracle cambiar la prioridad de sus procesos, y el ADR lo agrupa todo bajo un único problema (`ORA 800`, con 36 incidentes en `20261125T180708Z_07-adr-trazas.script.log`). Por eso el recuento aislado no sirve como alarma: siempre hay que contrastarlo con la línea base y revisar qué errores concretos aparecen. Lo que realmente preocupa es un código nuevo o un repunte anormal, no el número en sí.

**2. Ventana de mantenimiento del miércoles.**
Según `20261125T180327Z_01-automatizacion.spool.log`, `WEDNESDAY_WINDOW` arranca a las 22:00 (`byday=WED;byhour=22`) y dura 4 horas, hasta las 02:00 del jueves. Durante esa ventana se ejecutan las tres tareas de AutoTask, todas en estado ENABLED: `auto optimizer stats collection` (estadísticas del optimizador), `auto space advisor` (asesor de espacio) y `sql tuning advisor`.

**3. V$SPPARAMETER frente a V$PARAMETER tras SCOPE = SPFILE.**
En la sección 5 de `20261125T180400Z_02-parametros.script.log`, tras ejecutar `ALTER SYSTEM SET db_files = 250 SCOPE = SPFILE`, `V$SPPARAMETER` ya mostraba 250 con `ISSPECIFIED = TRUE` (el valor guardado en el archivo), mientras que `V$PARAMETER` seguía marcando 200 (el valor activo en memoria). Como `db_files` es un parámetro estático (`ISSYS_MODIFIABLE = FALSE`; intentar cambiarlo en memoria devuelve ORA-02095), el cambio queda pendiente hasta el próximo reinicio. Si no hubiéramos hecho el RESET, en el siguiente arranque la instancia habría subido con `db_files = 250`, aplicando un cambio que nadie había validado.

**4. Por qué RESET valía para db_files y no para processes.**
En la copia de texto del SPFILE que saqué antes de tocar nada (`config/20261125T180401Z_02-init_desde_spfile.ora`) no aparece ninguna línea para `db_files`, lo que significa que no estaba fijado y usaba su valor por defecto. Por eso un RESET (que borra la entrada del SPFILE) lo devuelve exactamente a su estado original, tal como confirma la sección 6: «(sin fijar)», `ISSPECIFIED = FALSE`. Con `processes` ocurre lo contrario: sí estaba definido explícitamente en la línea 31 de ese mismo PFILE (`*.processes=200`). Si le aplicáramos un RESET, quedaría sin valor y Oracle pasaría a usar su valor por defecto en lugar del 200 original. Para dar marcha atrás con `processes`, lo correcto es volver a fijar 200 con `SET`.

**5. Reinicio sin estado guardado.**
En `20261125T180401Z_02-estados-pdb.script.log` (sección 4), tras reiniciar la instancia, FREEPDB1 se quedó en `MOUNTED` y «(sin estado guardado)», de modo que C##OPS_RO recibió `ORA-01109: database not open` al intentar conectar por el servicio `freepdb1`. Aunque la instancia informó «Database opened», ese mensaje se refiere únicamente a la CDB (la raíz), que sí abrió con normalidad. Las PDB solo se abren automáticamente si tienen su estado guardado (`SAVE STATE`); de hecho, en la sección 5, tras abrirla y guardar el estado, el siguiente reinicio ya la dejó en READ WRITE.

### 9.1.1. Partes D-E · Servicios, sesiones y diccionario

**6. SID, SPID y evento de la sesión de la aplicación.**
En `20261125T180459Z_03-sesiones.script.log` (sección 3) encontramos: SID 60, SERIAL# 25828, SPID 1680 (correspondiente al proceso `oracleFREE (LOCAL=NO)` del contenedor, en estado sleeping) y evento `PL/SQL lock timer`. El campo USERNAME solo nos dice APP_ACADEMIA, que es la cuenta compartida por toda la aplicación. En cambio, MODULE (`matriculas`) y ACTION (`cierre-periodo`) nos indican qué módulo y qué operación concreta están corriendo, y CLIENT_IDENTIFIER (`ana.garcia@ucam`) identifica a la persona real detrás de esa cuenta compartida. Con estos datos sabemos exactamente a quién afecta el problema y a quién avisar sin tener que adivinar.

**7. CANCEL SQL frente a KILL SESSION.**
En el mismo archivo (secciones 4 y 5), tras lanzar `CANCEL SQL '60,25828'`, la aplicación mantuvo **1** transacción abierta: se canceló el SLEEP (la aplicación recibió ORA-01013 y siguió respondiendo «sigo viva»), pero el UPDATE sobre la fila 101 seguía sin confirmarse y reteniendo el bloqueo. En cambio, tras `KILL SESSION '60,25828' IMMEDIATE` quedaron **0** sesiones vivas de la aplicación y, por tanto, 0 transacciones (la aplicación recibió ORA-00028 y después ORA-03114). La fila 101 volvió a su estado original `ACTIVO` porque, al matar la sesión, Oracle hace rollback automático de cualquier transacción no confirmada.

**8. common_user_prefix.**
En mi imagen su valor es **C##** (`20261125T180459Z_03-cuenta-app.script.log`), mientras que en el enunciado aparece vacío. Esto implica que cualquier cuenta común (creada en la raíz y visible en todas las PDB) debe empezar obligatoriamente por `C##`, como `C##OPS_RO`, mientras que una cuenta local de una PDB, como `APP_ACADEMIA` (COMMON = NO), no puede llevar ese prefijo.

**9. Tabla de las tres miradas** (`20261125T180529Z_04-tres-miradas.script.log`):

| Vista | C##OPS_RO en CDB$ROOT | C##OPS_RO en FREEPDB1 | APP_ACADEMIA en FREEPDB1 |
|---|---|---|---|
| ALL_TABLES de ADMIN_ACADEMIA | 0 | 0 | 1 |
| DBA_USERS | 37 | 47 | ORA-00942 |
| CDB_USERS | 37 (CON_ID 1) + 47 (CON_ID 3) | 47 (CON_ID 3) | ORA-00942 |
| V$DATAFILE | 17 | 10 | ORA-00942 |

Aquí destacan dos diferencias clave. Por un lado, desde la raíz CDB_USERS devuelve tanto los usuarios de la raíz como los de FREEPDB1 porque C##OPS_RO tiene `CONTAINER_DATA = ALL`; sin embargo, consultada desde dentro de la PDB, la misma vista se limita a su contenedor (CON_ID 3). Por otro lado, ALL_TABLES no lista todo lo que existe en la base, sino solo aquello sobre lo que el usuario tiene privilegios: por eso C##OPS_RO (que puede leer el diccionario pero no tiene permisos sobre las tablas de Academia) ve 0, mientras que APP_ACADEMIA ve 1 (la tabla sobre la que sí tiene permiso), aunque carece de acceso a DBA_USERS o V$DATAFILE.

**10. Delta de la carga (04-deltas).**
Según `20261125T180540Z_04-deltas.script.log` (sección 4), los 10 recorridos completos de PEDIDO consumieron 53 838 lecturas lógicas —es decir, **5384 por recorrido**— y **0,28 s de DB time** (0,27 s de DB CPU y 0,27 s de tiempo de reloj). El hecho de que DB time ≈ DB CPU indica que la carga prácticamente no tuvo esperas: casi todo el tiempo se empleó en CPU leyendo bloques que ya estaban en caché, sin esperas de E/S ni bloqueos. Si el DB time fuera muy superior al DB CPU, esa diferencia correspondería a tiempo de espera y habría que investigar qué la está provocando.

### 9.1.2. Partes F-G · Espacio y acceso

**11. Las tres medidas de TBS_ACADEMIA_HIST antes de la carga.**
En `20261125T180601Z_05-almacenamiento.script.log` (sección 2) tenemos: 10 MB asignados, 3,9 MB libres (dentro de lo asignado) y 40 MB de máximo (el techo del autoextend), con 776 bloques usados de 5120, lo que supone un 15,16 % del máximo. A la hora de alertar, la métrica útil es el porcentaje ocupado sobre el **máximo**: el espacio libre dentro de lo asignado puede llevar a engaño porque el archivo todavía puede crecer, y el asignado solo refleja el tamaño actual en disco. Lo que de verdad avisa de que el tablespace va a quedarse sin sitio es cuánto margen queda hasta el techo, que es justo lo que vigila el umbral de MMON que configuré (aviso al 80 %, crítico al 95 %, sección 7).

**12. ORA-01950 y ORA-01536.**
En las secciones 4 y 5 del mismo archivo, aunque el INSERT lo lanzó SYS (el script se ejecuta como SYSDBA), quien «pagó» el espacio fue CARGA_HIST, el dueño de la tabla, ya que la cuota se descuenta siempre al propietario del segmento y no a quien ejecuta la inserción. Con cuota 0 saltó ORA-01950; al subirla a 2 MB, la carga de 5000 filas falló con ORA-01536 en cuanto consumió esos 2 MB (256 bloques). Ampliar el tablespace no habría solucionado nada: espacio en el tablespace había de sobra (techo de 40 MB), pero el tope estaba en la cuota individual del usuario, que no cambia por mucho que crezca el tablespace.

**13. Carga reanudable suspendida.**
La carga permaneció suspendida alrededor de **14 segundos**. En el alert log quedan registradas las dos líneas exactas: «statement in resumable session 'carga-historico' was suspended due to» (ORA-01536) a las 18:06:03.323 y «… was resumed» a las 18:06:17.429 (las marcas de tiempo exactas las obtuve de `V$DIAG_ALERT_EXT`, y ambas líneas constan en `20261125T180602Z_05-resumable.script.log`). Durante esa pausa, `DBA_RESUMABLE` la mostraba en estado `SUSPENDED` con un timeout de 180 s, y la sesión 60 esperaba en `statement suspended, wait error to be cleared`. De cara a la aplicación, todo fue transparente y no vio ningún error: arrancó a las 18:06:03, terminó a las 18:06:17 e insertó las 20 000 filas en cuanto amplié la cuota del propietario a 30 MB.

**14. Antes y después del SHRINK** (`20261125T180617Z_05-hwm-shrink.script.log`):

| Momento | Filas | Bloques | Cuota usada |
|---|---|---|---|
| Antes del DELETE | 20 000 | 2944 | 23 MB |
| Tras el DELETE | 2000 | 2944 | 23 MB |
| Tras SHRINK SPACE | 2000 | 304 | 2,4 MB |

El DELETE no liberó espacio porque únicamente borra filas dentro de los bloques existentes: esos bloques siguen asignados al segmento, por debajo de la marca de agua alta (HWM), y siguen contando en la cuota. Lo que hace `SHRINK SPACE` es compactar las filas hacia el principio del segmento (de ahí que exija `ENABLE ROW MOVEMENT`), bajar la HWM y devolver los bloques sobrantes.

**15. Perfil DEFAULT y PRF_PERSONAS.**
En `20261125T180632Z_06-gobierno-acceso.script.log` (sección 1), el perfil DEFAULT tiene: FAILED_LOGIN_ATTEMPTS 10, PASSWORD_LOCK_TIME 1 (día), PASSWORD_LIFE_TIME 180, PASSWORD_GRACE_TIME 7, INACTIVE_ACCOUNT_TIME 365, PASSWORD_REUSE_MAX y PASSWORD_REUSE_TIME en UNLIMITED, PASSWORD_ROLLOVER_TIME 0 y PASSWORD_VERIFY_FUNCTION en NULL. Frente a esto, PRF_PERSONAS fija 3 intentos fallidos, bloqueo de 1 minuto (1/1440, solo para el laboratorio), caducidad a los 90 días, 7 días de gracia, 90 días de inactividad y la función `ora12c_verify_function`. Dos motivos claros para aplicarlo a una persona de secretaría:
- **3 intentos fallidos**: Un usuario legítimo que se equivoca al teclear suele fallar una o dos veces; dejar 10 intentos da demasiado margen a alguien que esté probando contraseñas. En `06-probar-acceso`, al tercer fallo Ana quedó en estado `LOCKED(TIMED)`.
- **Función de verificación**: Sin ella (DEFAULT la tiene a NULL) cualquiera podría elegir una contraseña trivial; al activarla, el intento con `corta` fue rechazado con ORA-28003 («password length less than 8 characters»).

**16. Ana y ACA_MATRICULAS.**
En `20261125T180633Z_06-probar-acceso.script.log`, Ana pudo consultar las 60 000 filas de ACA_MATRICULAS sin que nadie le concediera permisos explícitos sobre esa tabla concreta. Esto se debe a que su rol `ROL_SECRETARIA_ACADEMIA` incluye `ROL_LECTURA_ACADEMIA`, el cual tiene el privilegio de esquema `SELECT ANY TABLE` sobre `ADMIN_ACADEMIA`, cubriendo todas las tablas actuales y futuras de ese esquema. Sin embargo, al intentar modificar el nombre de un alumno recibió `ORA-41900: missing UPDATE privilege on "ADMIN_ACADEMIA"."ACA_ESTUDIANTES"`, ya que su rol solo le permite hacer UPDATE sobre la columna ESTADO; de hecho, cambiar el estado sí funcionó (1 fila afectada, deshecha después con rollback).

### 9.1.3. Partes H-I · Objetos, diagnóstico y guardia

**17. Objetos invalidados por cada cambio de columna** (`20261125T180700Z_07-objetos.script.log`):
- **Añadir una columna nueva**: No se invalidó ningún objeto (los tres se mantuvieron VALID).
- **Ampliar `email_institucional`**: Solo se invalidó `V_CONTACTO_LAB6`, que es la única que referencia esa columna.
- **Ampliar `nombres`**: Se invalidó `V_ACTIVOS_LAB6` (que usa la columna) y, en cascada, `F_TOTAL_ACTIVOS_LAB6`, ya que la función depende de esa vista (como muestra la cadena de dependencias de la sección 4).

Añadir una columna nueva no provocó invalidaciones porque Oracle gestiona las dependencias a nivel de columna (dependencia de grano fino): una vista o función solo queda inválida si se altera algo que realmente utiliza, y ningún objeto podía estar usando una columna que aún no existía.

**18. Advisors de memoria** (`20261125T180703Z_07-memoria.script.log`, sección 5):
No merece la pena ampliar la SGA: duplicarla (de 1536 a 3072 MB, factor 2) mantiene `ESTD_DB_TIME_FACTOR` en 1 y las lecturas físicas estimadas en 24 972, exactamente igual que ahora, así que no mejoraría el DB time. En cuanto a la PGA, el advisor estima 0 sobreasignaciones a partir de 384 MB, pero ya marca 2 tanto a 256 MB como a 128 MB; por tanto, bajar de **384 MB** provocaría sobreasignaciones (actualmente el objetivo está en 512 MB).

**19. Lecturas lógicas de la consulta de la aplicación según la traza.**
En `20261125T180708Z_07-adr-trazas.script.log`, la consulta `SELECT COUNT(*) … WHERE estado = 'ACTIVO'` requirió **247** lecturas lógicas. Este dato aparece en la línea `FETCH` de la traza en bruto bajo `cr=247` (lecturas consistentes), y tkprof lo resume en la columna `query` de la fila Fetch (247), con 0 lecturas físicas de disco.

**20. Incidentes de la guardia** (detalle completo en `docs/bitacora/lab6-guardia.md`):
- **INC-601**: Evidencia: la sesión 176 (ANA_GARCIA) esperaba en `enq: TX - row lock contention`, bloqueada por la sesión 19 (APP_ACADEMIA), inactiva con 1 registro de undo. Intervención: `ALTER SYSTEM KILL SESSION '19,31982' IMMEDIATE`. Validación: 0 sesiones bloqueadas y el alumno 101 pasó a ACTIVO.
- **INC-602**: Evidencia: el servicio `academia_app` figuraba con ARRANCADO = NO y el listener solo anunciaba `freepdb1`. Intervención: `DBMS_SERVICE.START_SERVICE('academia_app')` y `SAVE STATE`. Validación: el listener anunció el servicio en estado READY y `app.sh` conectó por `academia_app`.
- **INC-603**: Evidencia: APP_ACADEMIA estaba `LOCKED` (no TIMED) y la auditoría unificada registró un `ALTER USER app_academia ACCOUNT LOCK` ejecutado por SYS a las 18:09:22. Intervención: `ALTER USER app_academia ACCOUNT UNLOCK`. Validación: la cuenta quedó OPEN y `app.sh` volvió a entrar sin problemas.

## 10. Simulacro de examen 1Z0-182

| Pregunta | Respuesta | Dónde lo comprobé |
|---|---|---|
| S1 | B · MMON | Partes B y F (umbrales de espacio de MMON) |
| S2 | C | Parte C (`sort_area_size`, DEFERRED) |
| S3 | B · CREATE PFILE FROM MEMORY | Parte C (`init_desde_memoria.ora`) |
| S4 | B y C | Parte C (OPEN READ WRITE sin cerrar dio ORA-65019; con FORCE funcionó) |
| S5 | C · DBA_PDB_SAVED_STATES | Parte C (columna ESTADO_GUARDADO) |
| S6 | B · ORA-01035 | Parte C (C##OPS_RO en modo RESTRICTED) |
| S7 | A · DBMS_SERVICE | Parte D (creación de `academia_app`) |
| S8 | B · PADDR = ADDR | Parte D (SID → SPID 1680) |
| S9 | C | Parte D (CANCEL SQL dejó 1 transacción abierta) |
| S10 | C · PUBLIC | Parte D (sección 6: `USER is "PUBLIC"`) |
| S11 | B · V$CON_SYSMETRIC | Parte E (04-deltas, sección 1) |
| S12 | A y C | Parte E (DB time = CPU + esperas no Idle de primer plano) |
| S13 | B · HR | Parte F (pagó CARGA_HIST, el dueño, aunque insertó SYS) |
| S14 | A y C | Parte F (ORA-30041 sobre TEMP) |
| S15 | A y C | Parte F (DBA_RESUMABLE en SUSPENDED, timeout 180) |
| S16 | B | Parte F (ENABLE ROW MOVEMENT antes del SHRINK; TBS_ACADEMIA_HIST con segmentos AUTO) |
| S17 | B · LOCKED(TIMED) | Parte G (Ana tras tres fallos) |
| S18 | B | Parte G (cascada: el de sistema de U6_B sobrevive) |
| S19 | A y B | Parte H (MOVE dejó `IX_EST_LAB6_ESTADO` UNUSABLE) |
| S20 | B · DBMS_MONITOR.SESSION_TRACE_ENABLE | Parte H (traza de la sesión 214) |
| S21 | C · DIAGNOSTIC_DEST | Parte H (ADR Base `/opt/oracle`) |
| S22 | C | Parte I (INC-601: primero el bloqueador) |
