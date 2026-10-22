# Lab 4 — Respuestas de comprobación

Todas las rutas de evidencia son relativas a `docs/bitacora/lab4/evidencia/`.

## Puntos de control

### Parte B — ¿Cuántos archivos salieron como FUERA en el paso 3 del backup?

Ninguno: los 26 archivos del inventario salieron como `dentro` y el paso 6 dijo "ninguno: el volumen contiene toda la base de datos" (`script/20261022T155444Z_00-backup-lab3.script.log`, `archivos_fuera_del_volumen=0` en el manifiesto). No es que los datafiles de negocio estuvieran bien colocados: en mi imagen `$ORACLE_HOME/dbs` es un enlace simbólico a `/opt/oracle/oradata/dbconfig/FREE/dbs`, que está dentro del volumen. Por eso `readlink -f` resolvió `.../dbs/academia01.dbf` a `/opt/oracle/oradata/dbconfig/FREE/dbs/academia01.dbf` y lo dio por "dentro". El diagnóstico de la Parte C lo confirmó: "DENTRO del volumen, pero en carpeta no estándar" para los cinco (`script/20261022T155807Z_01-diagnostico.script.log`).

### Parte E — Explica ORA-01507 y ORA-01219 con tus palabras

En NOMOUNT la instancia solo ha leído el SPFILE: tiene la SGA y los procesos, y por eso `v$parameter` responde (los parámetros ya están en memoria). Pero `v$database` saca sus datos del control file, que todavía no está abierto, y falla con ORA-01507. En MOUNT ya se ha abierto el control file y Oracle conoce los 16 datafiles, pero no los ha abierto. `dba_tables` es una vista del diccionario, que vive en el tablespace SYSTEM (un datafile todavía cerrado), así que da ORA-01219; solo funcionan las vistas fijas V$, que leen memoria y control file (`script/20261022T160156Z_06-arranque-por-etapas.script.log`). Las cifras de "Total System Global Area" (1 603 398 928 bytes) son las mismas que da `v$sga` en la evidencia 08.

### Parte G — Interpreta el experimento de caché

Lo explico en la pregunta 17.

## Respaldo

### 1. ¿Por qué la copia de la Parte B es consistente? ¿Qué pasaría si se tomara con la base abierta?

`backup-estado.sh` hace `SHUTDOWN IMMEDIATE` antes de copiar ("Database closed / Database dismounted / ORACLE instance shut down" en la evidencia 00). Con un cierre limpio hay checkpoint final: todos los datafiles quedan con el mismo SCN de checkpoint, el control file lo sabe y no queda nada pendiente en el redo, así que al restaurar la base abre con `STARTUP` sin recuperación. Si se copiara con la base abierta, cada archivo se copiaría en un instante distinto mientras DBWn sigue escribiendo bloques: los datafiles tendrían SCN distintos y bloques a medio escribir. Esa copia sería inconsistente y solo serviría con el redo archivado para recuperarla (ARCHIVELOG + RMAN, Lab 5).

### 2. ¿Para qué sirven manifiesto.txt y SHA256SUMS? ¿Qué comprueba restaurar-estado.sh antes de tocar nada?

`manifiesto.txt` dice qué es la copia: etiqueta `lab3-final`, fecha `20261022T155725Z`, contenedor, volumen, imagen `container-registry.oracle.com/database/free:latest`, commit `b2a31d2`, tipo "copia en frio consistente" y 0 archivos fuera del volumen. `SHA256SUMS` guarda la huella de cada archivo (`volumen.tar.gz`, `inventario.txt`, `manifiesto.txt`, `extras.lst`). En la B.3, `sha256sum --check` dio `OK` en los cuatro. `restaurar-estado.sh` hace exactamente eso primero (`sha256sum --check --quiet SHA256SUMS`): si una sola suma no coincide, se para sin tocar el volumen. Después pide escribir el nombre del volumen para confirmar, salvo que se pase `--si`.

### 3. Define RPO y RTO con los datos de tu propia copia.

- RPO (cuántos datos puedo perder): todo lo hecho desde `20261022T155445Z`. Si hoy restaurase `backups/20261022T155445Z_lab3-final` perdería todo el trabajo del Lab 4 sobre la base: la reubicación de datafiles, V003, V002 (20 000 estudiantes), C##OPS_RO y la evidencia que dependa de ellos. Por eso el Lab 5 empieza con una copia `lab4-final`.
- RTO (cuánto tardo en volver a dar servicio): la copia tardó unos 2 min 40 s entre el inicio (15:54:45) y el manifiesto (15:57:25), y la base volvió a READ WRITE en 10 s. No he ejecutado todavía una restauración, así que mi RTO es una estimación del mismo orden: descomprimir 551 MB y arrancar, unos 3 minutos. El simulacro real con RTO medido es del Lab 5.

## Diagnóstico y correcciones

### 4. ¿Dónde creó Oracle los datafiles de V000 y por qué? ¿Qué parámetro habría evitado el problema?

En `/opt/oracle/product/26ai/dbhomeFree/dbs/` (FILE# 16 a 20 en el diagnóstico 01). V000 los creó con ruta relativa (`'academia01.dbf'`) y `db_create_file_dest` estaba vacío ("sin OMF" en la sección 5 del diagnóstico), así que Oracle usó su directorio por defecto, `$ORACLE_HOME/dbs`. En mi imagen ese directorio es un enlace al volumen (`/opt/oracle/oradata/dbconfig/FREE/dbs`), así que no se habrían perdido con `docker rm`, pero estaban mezclados con el SPFILE, el password file y los archivados, en una carpeta que nadie espera y que depende de un enlace de la imagen. Lo habría evitado usar una ruta absoluta en V000 o fijar `DB_CREATE_FILE_DEST='/opt/oracle/oradata'` (OMF).

### 5. ¿Por qué ALTER TABLESPACE ... RENAME DATAFILE no basta para mover un archivo?

Porque no mueve nada: solo cambia la ruta que guarda el control file. Si el archivo no está en la ruta nueva, `ONLINE` falla con ORA-01157/ORA-01110. Por eso `03-reubicar-datafiles.sh` hace OFFLINE NORMAL (checkpoint del tablespace), copia con `cp -p`, hace RENAME, pasa a ONLINE (Oracle valida la cabecera) y solo entonces borra el original. Resultado: los cinco en `/opt/oracle/oradata/FREE/FREEPDB1/` con AUTOEXTEND y MAX_MB 1024, y la segunda ejecución responde "Nada que reubicar" (`script/20261022T155827Z_03-reubicar-datafiles.script.log` y `..._03-reubicar-datafiles-idempotencia.script.log`).

### 6. ¿Por qué no se edita V000 y se crea V003? ¿Por qué invalidar una contraseña es mejor que borrarla del archivo?

V000 ya se aplicó: es historia. Si la edito, el repositorio deja de describir lo que se ejecutó y un entorno nuevo no quedaría igual que el mío. Las correcciones van hacia delante, en una migración nueva (forward-only migrations). Además, borrar las contraseñas del archivo no las quita del historial de Git: cualquiera con el repositorio puede verlas con `git log -p`. Lo que protege es que dejen de servir. V003 pone las cinco cuentas `ADMIN_*` en `NO AUTHENTICATION`, es decir, como cuentas de solo esquema (spool `20261022T155919Z_04-V003__...`). La prueba es que `admin_academia` con la contraseña de V000 da `ORA-01017` (`script/20261022T155919Z_04-login-v000-invalidado.script.log`).

### 7. ¿Qué demuestra la prueba de persistencia y por qué se ejecuta después de reubicar?

Demuestra que la base vive en el volumen y no en el contenedor. Paré la base, borré `oralab-26ai` y lo recreé con `05-crear-contenedor.sh` v2 ("se reutiliza (sin contraseñas)"). Antes y después aparecen los mismos 5 tablespaces ONLINE en `FREE/FREEPDB1` y las mismas tablas por esquema (3, 3, 2, 3, 3), y solo se publica `1521/tcp` (`script/20261022T155950Z_03-prueba-persistencia.script.log`). El HOST_NAME pasó de `6f6c5d9854bf` (evidencia 01) a `e38ca20e9401` (evidencia 05): es otro contenedor, con la misma base. Se ejecuta después de reubicar y del backup porque, en una imagen donde los datafiles estuvieran de verdad en la capa del contenedor, el `docker rm` los borraría.

## Instancia y arranque

### 8. Con tus evidencias: tres datos que pertenecen a la instancia y tres que pertenecen a la base de datos.

Instancia (`spool/20261022T160126Z_05-instancia-y-base.spool.log` y `script/..._05-instancia-so.script.log`):
- `STARTUP_TIME` 2026-10-22 16:00:00: cambia en cada arranque.
- `HOST_NAME` `e38ca20e9401`: es el contenedor y cambió al recrearlo.
- La SGA (1536 MB de `sga_target`) y los 66 procesos de background `db_*_FREE` que solo existen mientras está arrancada.

Base de datos:
- `DBID` 1509902213: no cambió al recrear el contenedor.
- `CREATED` 2026-07-05: cuando se construyó la base de la imagen, no cuando arrancó.
- Los archivos: 16 datafiles (3974 MB), 3 tempfiles, 3 miembros de redo y 2 control files. También `LOG_MODE` ARCHIVELOG, que está guardado en el control file.

### 9. ¿Qué lee Oracle en NOMOUNT, en MOUNT y en OPEN? Relaciónalo con ORA-01507 y ORA-01219.

- NOMOUNT: lee el SPFILE (`/opt/oracle/product/26ai/dbhomeFree/dbs/spfileFREE.ora`), reserva la SGA y arranca los procesos. No hay control file, así que `v$database` da ORA-01507.
- MOUNT: lee los control files (`control01.ctl` y `control02.ctl`) y conoce la estructura (16 datafiles), pero no los abre. El diccionario está en SYSTEM, cerrado, así que `dba_tables` da ORA-01219.
- OPEN: abre datafiles y redo online. El alert log muestra "Crash Recovery excluding pdb 2/3 which was cleanly closed", "Thread 1 opened at log sequence 4" y "Completed: ALTER DATABASE OPEN".

### 10. ¿Por qué FREEPDB1 se abre sola tras ALTER DATABASE OPEN? ¿Qué harías si no lo hiciera?

Porque tiene estado guardado: `dba_pdb_saved_states` muestra `FREEPDB1 FREE OPEN` (evidencia 06), y el script confirmó "FREEPDB1 abierta sola en READ WRITE: el estado guardado funciona". Al abrir el CDB, Oracle reabre cada PDB en el estado que se guardó con `SAVE STATE`. Si se quedara en MOUNTED (las aplicaciones recibirían ORA-01109), haría `ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;` y después `ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;` para que no se repita en el próximo arranque.

### 11. Clasifica ORA-00800 y ORA-20013 del alert log: ¿alerta real o ruido conocido? Justifícalo.

- `ORA-00800 [Set Priority Failed] [VKTM]` y `[LGWR]`: ruido conocido. Aparece en cada arranque (`script/20261022T160214Z_06-alert-log.script.log`) porque Oracle intenta subir la prioridad de esos procesos y el contenedor no tiene `CAP_SYS_NICE`. Es un error "soft" y la base abre igual.
- `ORA-20013 DBMS_QOPATCH`: en mi alert log no aparece ninguna vez (`grep -c` da 0). Si saliera, también sería ruido conocido: el inventario de parches no se puede leer en una imagen de contenedor.
- Otros que sí me salieron: `ORA-27300/27301/27302/27303 chdir()` y `ORA-07427: spstp: cannot change directory to dbs`. Fueron el 2026-07-05, cuando se construyó la imagen, no durante mis arranques, así que también los clasifico como conocidos. Lo mismo `ORA-27167 ... parse_df`, que sale porque Oracle no puede determinar el tipo de sistema de archivos del binario dentro del contenedor.

Cualquier ORA- que no esté en esta lista lo investigaría.

## Parámetros

### 12. Diferencia entre v$parameter y v$spparameter con el ejemplo de db_files de tu evidencia.

`v$parameter` es el valor que usa la instancia ahora, en memoria; `v$spparameter` es lo que está escrito en el SPFILE para el próximo arranque. En la evidencia 07 (`script/20261022T160255Z_07-parametros-cambios.script.log`), `ALTER SYSTEM SET db_files = 300 SCOPE = MEMORY` falló con ORA-02095 porque es estático. Con `SCOPE = SPFILE` quedó `db_files 200 (memoria) / 300 (spfile)`: el cambio está pendiente hasta reiniciar. Tras `ALTER SYSTEM RESET db_files SCOPE = SPFILE` volvió a `200 / (no especificado)`. La verificación final lo comprueba con `db_files sin línea pendiente en el SPFILE (FALSE)`.

### 13. ¿Qué hace SCOPE=BOTH y cuál es el SCOPE por defecto con SPFILE?

`SCOPE=BOTH` cambia el valor en memoria (efecto inmediato) y en el SPFILE (sobrevive al reinicio). En la evidencia 07 se ve `open_cursors 500 / 500` con el comentario `Lab4: prueba SCOPE=BOTH`, y después se deshizo a 300. Si la instancia arrancó con SPFILE, como la mía (sección 1 de `spool/..._07-parametros-lectura.spool.log`), el SCOPE por defecto es BOTH. Si arrancó con PFILE, es MEMORY.

### 14. ¿Por qué PROCESSES rechaza el cambio en memoria aunque v$parameter lo marque IMMEDIATE?

En mi `07-parametros-lectura` aparece `processes 200 ... SYS_MOD IMMEDIATE`, pero `ALTER SYSTEM SET processes = 250 SCOPE = MEMORY` dio ORA-02097 + ORA-02095. PROCESSES dimensiona estructuras de la SGA (tablas de procesos y sesiones, y SESSIONS se deriva de él: 322) que se reservan al arrancar y no se pueden redimensionar en caliente. En la práctica es estático: se cambia con `SCOPE=SPFILE` y reiniciando. La metadata orienta; la prueba decide.

### 15. ¿Para qué sirve versionar el PFILE? Explica "configuration drift".

El PFILE versionado (`config/initFREE_lab4.ora`, 32 líneas, y `config/initFREE_lab4_memoria.ora`, 122) es una foto legible de la configuración. Sirve para restaurar un SPFILE roto (`STARTUP PFILE=...` y `CREATE SPFILE FROM PFILE`) y para comparar con `git diff`. La configuration drift (deriva de configuración) es cuando la configuración real se va separando de la documentada por cambios que nadie registró. Comparar la foto de hoy con la anterior la detecta. Además, `07-copiar-pfile.sh` mostró que parámetros como `log_buffer`, `optimizer_mode` o `job_queue_processes` tienen valor en memoria sin estar en el SPFILE: son derivados o por defecto.

## Memoria y procesos

### 16. ¿Qué modelo de gestión de memoria usa tu instancia? Justifícalo con tres parámetros.

ASMM (Automatic Shared Memory Management). Según `spool/20261022T160324Z_08-memoria.spool.log`:
- `memory_target = 0` y `memory_size = 0`: no hay AMM, Oracle no reparte SGA y PGA juntas.
- `sga_target = 1536M`: Oracle reparte automáticamente esos 1536 MB entre buffer cache (1024 MB), shared pool (384 MB), Shared IO Pool (80 MB) y large pool (16 MB).
- `pga_aggregate_target = 512M`: la PGA se gestiona aparte, con límite duro `pga_aggregate_limit = 2G`.

`db_cache_size = 0` y `shared_pool_size = 0` significan "sin mínimo fijado; que ASMM decida".

### 17. Explica las cifras de tu experimento de caché: lecturas lógicas, físicas y tasa de acierto.

`ACA_ESTUDIANTES` tiene 20 000 filas en 244 bloques (`script/20261022T160325Z_08-experimento-cache.script.log`). Tras `FLUSH BUFFER_CACHE`:

| Ronda | Lecturas lógicas | Lecturas físicas | Tasa de acierto (1 − físicas/lógicas) |
|---|---|---|---|
| 1 (caché fría) | 359 | 260 | 1 − 260/359 = 27,6 % |
| 2 (caché caliente) | 247 | 0 | 1 − 0/247 = 100 % |

Toda lectura física acaba en una lógica: el bloque se trae a la caché y desde ahí se lee. La consulta en sí lee 247 bloques (los 244 de datos más la cabecera del segmento y los bloques que el recorrido completo consulta para localizar los extents), igual en las dos rondas. En mi ronda 1 salen 112 lógicas y 15 físicas más que en el PDF: el flush también vació los bloques del diccionario, y la primera ejecución tuvo que releerlos (SQL recursivo para resolver la tabla y las vistas `v$mystat`/`v$statname`). Al final, `v$bh` muestra 245 bloques de la tabla en caché: los 244 de datos más la cabecera del segmento. En la ronda 2 todo estaba ya en memoria: 0 lecturas a disco.

### 18. ¿Por qué no se suma el RSS de los procesos para saber la memoria de la instancia?

Porque el RSS de cada proceso incluye las páginas de la SGA compartida que ese proceso ha tocado, y la SGA es la misma para todos. En mi evidencia 09 cada proceso de background muestra entre 82 y 283 MB de RSS. Sumar 62 procesos daría varios GB, cuando `docker stats` dice que el contenedor usa 2.2 GiB en total (evidencia 05). La memoria real de la instancia es `v$sga` (≈1529 MB) más `v$pgastat` (`total PGA allocated` 394 MB).

### 19. Relaciona el SPID de LGWR de v$process con el PID de /proc. ¿Por qué docker top muestra otro número?

`09-procesos.sql` da `LGWR SPID 763` (`spool/20261022T160353Z_09-procesos.spool.log`) y `09-procesos-so.sh 763` encuentra en `/proc` del contenedor `763 103 db_lgwr_FREE` (`script/20261022T160354Z_09-procesos-so.script.log`): es el mismo proceso. `docker top` muestra otro número, en mi caso `4537 db_lgwr_FREE` (y `4489` para PMON, cuyo SPID es 715). El contenedor tiene su propio espacio de nombres de PID: el mismo proceso se llama 763 dentro y 4537 en el anfitrión. Oracle guarda el PID de dentro.

### 20. ¿Qué hace PMON y qué hace SMON? Pon un ejemplo de cada uno.

- PMON (process cleanup, SPID 715) limpia los procesos de usuario que mueren: deshace su transacción con el undo y libera sus bloqueos y recursos. Ejemplo: si se apaga el portátil de un usuario en mitad de un UPDATE, PMON detecta el proceso servidor huérfano, hace rollback y la sesión que esperaba esas filas puede seguir.
- SMON (System Monitor, SPID 767) hace la recuperación de instancia al arrancar tras una caída y limpia segmentos temporales. Ejemplo: tras un `SHUTDOWN ABORT` o un corte de luz, en el siguiente `ALTER DATABASE OPEN` SMON aplica el redo y deshace lo no confirmado. En mi alert log se ve "Crash Recovery excluding pdb ... which was cleanly closed": como el cierre fue limpio, no tuvo nada que recuperar.

## Seguridad y trabajo profesional

### 21. ¿Por qué C##OPS_RO y no SYSTEM para observar? ¿Qué no puede hacer C##OPS_RO?

Por mínimo privilegio (least privilege): para mirar no hace falta poder romper. SYSTEM puede crear, borrar y cambiar casi todo, y un error en una consulta de monitorización (o una credencial filtrada) tendría consecuencias. C##OPS_RO es un usuario común (existe en CDB$ROOT y FREEPDB1, `COMMON = YES`, evidencia 02) con solo `CREATE SESSION`, `SET CONTAINER` y `SELECT_CATALOG_ROLE`, más `CONTAINER_DATA = ALL` para ver las filas de todas las PDB desde la raíz. No puede modificar datos, crear o borrar objetos, hacer `ALTER SYSTEM`, parar o arrancar la base, ni leer las tablas de negocio (solo el diccionario y las vistas V$). Su contraseña está en `config/.env`, que no se versiona, y `ro.sh` se la pasa a SQLcl por la entrada estándar.

### 22. ¿Qué dos vías de exposición de contraseñas corrige este laboratorio respecto al Lab 3?

1. La contraseña de SYS en la línea de comandos: el 08 del Lab 3 ejecutaba `sqlplus "sys/$ORACLE_PWD@..."`, visible con `ps -ef` y en los eventos de Docker. El 08 v2 usa `/ as sysdba` (autenticación del sistema operativo), así que no hay contraseña que filtrar. Lo mismo hacen `sysdba.sh` y los demás scripts. Además, el 05 v2 ya no pasa `ORACLE_PWD` a `docker run` cuando el volumen ya tiene base, así que no aparece en `docker inspect`.
2. Las contraseñas literales de los propietarios de esquema en V000, que están en el historial de Git para siempre. V003 las invalida con `NO AUTHENTICATION`.

## Preguntas tipo examen

| Pregunta | Respuesta | Justificación |
|---|---|---|
| P1 | A, D | Tras SHUTDOWN IMMEDIATE todos los archivos comparten el SCN de checkpoint: la copia es consistente, no necesita redo y abre con STARTUP. No depende de ARCHIVELOG ni de RMAN. |
| P2 | B | En Standard Edition no hay MOVE DATAFILE online: OFFLINE, copia con el SO, RENAME (solo cambia el control file) y ONLINE, como en `03-reubicar-datafiles.sh`. |
| P3 | A, C | STARTUP_TIME es de la instancia y cambia en cada arranque; CREATED es de la base. El DBID no cambia al recrear el contenedor (1509902213 antes y después), una instancia monta una sola base y v$database responde ya en MOUNT. |
| P4 | A, C | En NOMOUNT solo hay SPFILE en memoria: v$parameter responde y se puede crear un control file. dba_users necesita OPEN, y ARCHIVELOG o renombrar SYSTEM necesitan MOUNT. |
| P5 | B | ALTER DATABASE ARCHIVELOG se hace en MOUNT: el modo se guarda en el control file y la base no puede estar abierta. |
| P6 | B | Con SPFILE, el SCOPE por defecto es BOTH: el cambio se aplica en memoria y en el SPFILE. |
| P7 | C | PROCESSES no es modificable a nivel de PDB (ISPDB_MODIFIABLE = FALSE): ORA-65040, igual que en mi evidencia 07. |
| P8 | A, C | STARTUP PFILE='init_bueno.ora' arranca con una copia buena y CREATE SPFILE FROM PFILE repara el SPFILE. |
| P9 | B | Con MEMORY_TARGET = 0 y SGA_TARGET > 0 es ASMM; un DB_CACHE_SIZE fijado actúa como mínimo, no como tamaño exacto. |
| P10 | A, D | En servidor dedicado, el área de ordenación y la memoria de sesión están en la PGA; shared pool, log buffer y buffer cache son SGA. |
| P11 | C | El COMMIT espera a que LGWR escriba el redo en los redo logs online; DBWn escribe los bloques después. |
| P12 | B, D | PMON y SMON son obligatorios; ARC0 solo existe en ARCHIVELOG, RECO es para transacciones distribuidas y CJQ0 para jobs. |

## Hallazgos adicionales de mi entorno

- La base está en ARCHIVELOG, no en NOARCHIVELOG como en el ejemplo del enunciado (evidencias 01, 05 y 06), y hay 4 procesos ARCn (evidencia 09). Como `db_recovery_file_dest` está vacío, los redo archivados (`arch1_*.dbf`) van a `$ORACLE_HOME/dbs`, es decir, a `/opt/oracle/oradata/dbconfig/FREE/dbs`. Están dentro del volumen pero sin FRA ni límite de espacio. Lo dejo anotado para el Lab 5 (ARCHIVELOG/FRA).
- En las evidencias generadas con `sqlplus` dentro del contenedor, las tildes salen como `??`: el `NLS_LANG` del contenedor no es UTF-8. Las generadas con SQLcl desde Linux se ven bien. No afecta a los datos.
