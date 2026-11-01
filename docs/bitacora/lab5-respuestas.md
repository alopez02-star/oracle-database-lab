# Lab 5 — Oracle DBA Workflow II · Respuestas

Autor: Antonio López Ros

Todas las cifras proceden de mis evidencias en `docs/bitacora/lab5/evidencia/` (salvo la copia lab3-final, en `docs/bitacora/lab4/evidencia/script/`). Cuando mi cifra no coincide con la del enunciado, lo indico.

## 8. Preguntas de comprobación

### 8.1. Punto de partida y banco de medida (A, B)

**1. Paso 6 de lab4-final frente a lab3-final.**
En mi caso el paso 6 dice lo mismo en las dos copias: «ninguno: el volumen contiene toda la base de datos» (`20261022T155444Z_00-backup-lab3.script.log` y `20261101T162428Z_00-backup-lab4.script.log`), y los dos manifiestos tienen `archivos_fuera_del_volumen=0`. La diferencia está en el inventario (paso 2): en lab3-final los cinco datafiles de aplicación (`academia01.dbf`, `clinica01.dbf`, `retail01.dbf`, `logistica01.dbf`, `fintech01.dbf`) estaban en `/opt/oracle/product/26ai/dbhomeFree/dbs/`, y en lab4-final están en `/opt/oracle/oradata/FREE/FREEPDB1/`. El script los dio por «dentro» en lab3 porque, en la imagen de Oracle Free, `dbs/` es un enlace que apunta al volumen, así que no salió el aviso de cinco archivos fuera que muestra el enunciado. Aun así, la diferencia demuestra que la reubicación del Lab 4 funcionó: ahora los datafiles están en su ruta canónica del volumen y no dependen de un enlace dentro del ORACLE_HOME. De paso, el volumen pasó de 4977 MB a 5089 MB.

**2. Factor de bloqueo de CLIENTE y PEDIDO.**
Con `20261101T162911Z_01-perfil-tiendasur.script.log`: CLIENTE tiene r = 100 000 y b = 1517, así que bfr = 100 000 / 1517 ≈ 66; PEDIDO tiene r = 1 000 000 y b = 5320, así que bfr ≈ 188. El enunciado da 1467 bloques y bfr 68 para CLIENTE; el mío es un poco menor, y PEDIDO coincide. PEDIDO cabe a casi el triple de filas por bloque (188/66 ≈ 2,85) porque su fila media ocupa 32 bytes frente a 102 de CLIENTE: son columnas numéricas, una fecha y códigos cortos, y no tiene cadenas largas como email, apellidos o ciudad.

**3. ¿Por qué se mide solo la segunda ejecución?**
La primera ejecución carga trabajo que no se repite: el análisis duro del cursor, la lectura física de los bloques que todavía no están en la caché y la limpieza diferida de bloques de las cargas anteriores. Medirla mezclaría ese coste único con el coste real de la consulta. En la segunda, en caliente, las cifras son estables y comparables: en `20261101T162912Z_02-anatomia-indice.script.log` la búsqueda de `id_pedido = 4711` da Buffers = 4 (3 del índice + 1 de la tabla) y 0 lecturas físicas, y `prueba_pk` en `20261101T162912Z_01-herramienta-medir.script.log` da 4 lecturas lógicas y 0 físicas.

### 8.1.1. Anatomía y tipos de índice (C, D)

**4. Altura, hojas y fan-out de IX_PED_CLIENTE.**
Según `20261101T162912Z_02-anatomia-indice.script.log`, la altura es 3, con 2216 hojas y 6 ramas. El fan-out real es (2216 + 6 − 1) / 6 = 370 y el máximo es 8028 / 14,8 = 543. Las hojas están llenas al 90 % (451 entradas de 503 posibles) porque el índice se creó después de la carga con PCTFREE 10. Con mil millones de filas harían falta unas 2 208 969 hojas y la altura sería 4, es decir, 5 lecturas para encontrar una fila. Con el fan-out real de 370 sale lo mismo: log₃₇₀(2,2·10⁶) ≈ 2,5, que redondeado da 3 niveles de rama más la hoja. La altura crece con el logaritmo del número de filas.

**5. ROWID en la hoja de PK_CLIENTE.**
La primera entrada de la hoja (`row#0`, clave `c1 02` = 1) guarda `data:(6): 00 00 03 1b 00 00` (`20261101T162912Z_02-anatomia-indice.script.log` y el volcado `config/20261101T162912Z_02-volcado-indice.trc.txt`). Los 4 primeros bytes son el bloque, 0x31B = 795, y los 2 últimos la ranura, 0. La consulta de comprobación devuelve lo mismo para el cliente 1: bloque 795, ranura 0. No aparece el número de archivo porque el tablespace es bigfile: tiene un solo datafile, que ya se conoce por el segmento, y los 4 bytes se usan enteros para el número de bloque. Tampoco hace falta el número de objeto, porque un índice no particionado apunta siempre a la misma tabla.

**6. Tres pruebas de la Parte D** (`20261101T162914Z_03-tipos-indice.script.log`):
- **Único (Q13).** Pasa de 1486 lecturas con recorrido completo a 3 con INDEX UNIQUE SCAN de UX_CLI_DNI. Gana porque baja por el árbol y para en la primera coincidencia. Además, la base rechaza el duplicado con ORA-00001. El enunciado da 1436 para el recorrido completo.
- **Compuesto y cubriente (Q7).** Con `ix_ped_cliente` cuesta 15 lecturas y un SORT; con `(id_cliente, fecha)` 7 lecturas y sin SORT; con la versión cubriente 3 lecturas, sin tocar la tabla. Primero va la igualdad y después el rango, así que el índice ya devuelve las filas ordenadas; si además contiene todas las columnas que pide la consulta, se ahorra el acceso a la tabla.
- **Bitmap frente a recorrido (Q14).** Pasa de 4073 lecturas a 528 con BITMAP AND de cuatro bitmaps. Los bitmaps de baja cardinalidad se combinan con operaciones de bits antes de ir a la tabla, y ocupan poco: BX_VH_REGION tiene 114 hojas frente a 1954 del B-tree equivalente.
- **Extra (Q15).** El clúster hash cuesta 1 lectura, la IOT 3 y el montículo con PK 4. Pero la limpieza por rango sobre el hash cuesta 2770 lecturas, porque el hash no sirve para rangos.

### 8.1.2. Agrupamiento, equilibrio y sargabilidad (E, F)

**7. CF de IX_PED_FECHA frente a IX_PED_CLIENTE.**
En `20261101T163018Z_04-agrupamiento-equilibrio.script.log`, IX_PED_FECHA tiene CF = 5220, muy cerca de los 5320 bloques de la tabla, e IX_PED_CLIENTE tiene CF = 999 041, casi igual al millón de filas. El cálculo a mano da exactamente lo mismo. Los pedidos se insertaron en orden de fecha, así que fechas consecutivas están en bloques consecutivos. En cambio, los pedidos de un mismo cliente están repartidos por toda la tabla, y casi cada entrada del índice salta a otro bloque. Un REBUILD no lo arreglaría: el CF depende del orden físico de las filas en la tabla, no del índice. Solo bajaría reordenando la tabla (por id_cliente), y eso empeoraría el CF de la fecha.

**8. Punto de equilibrio medido.**
Según la curva de `20261101T163018Z_04-agrupamiento-equilibrio.script.log`:
- **Índice con CF malo.** Lee 4875 bloques al 0,5 % y 9920 al 1 %, frente a 5232 del recorrido completo. En lecturas lógicas deja de compensar entre el 0,5 % y el 1 %, lo que cuadra con la fórmula de la Unidad 3: k/r < 1/bfr = 1/188 ≈ 0,53 %. El optimizador cambia antes, ya en el 0,2 % (2004 lecturas, recorrido completo), más cerca del umbral en disco de la fórmula con c_sec/c_aleat = 1/5, que da 0,11 %.
- **Índice con CF bueno.** Sigue ganando al 50 % (3938 lecturas) y solo pierde al 100 % (7875). El cruce queda en torno a 5232/7875 ≈ 66 %, porque lee unas k/bfr hojas y bloques en lugar de un bloque por fila.

**9. Q4b ('Gonz%').**
En `20261101T163034Z_05-sargabilidad.script.log`, Q4b devuelve 2056 filas con INDEX RANGE SCAN de IX_CLI_APELLIDOS y lee 2024 bloques. Un recorrido completo de CLIENTE cuesta unas 1486 lecturas en mis medidas (Q1 rota, Q5, Q13), y Q4a, con recorrido completo, 1523. El enunciado da 1472. El predicado es sargable porque un prefijo es un rango del B-tree, pero selecciona el 2,1 % de los clientes con un índice de CF malo, así que hace casi una lectura de tabla por fila. Eso supera el punto de equilibrio de CLIENTE (1/bfr = 1/66 ≈ 1,5 %). Sargable no significa que usar el índice compense: aquí lo correcto habría sido el recorrido completo. El CF de IX_CLI_APELLIDOS no aparece en mi evidencia; el «CF malo» se deduce de las 2024 lecturas para 2056 filas.

### 8.1.3. Coste de escritura y mantenimiento (G, H)

**10. Coste del INSERT y propuesta del compañero.**
Según `20261101T163039Z_06-indices.script.log`, un INSERT modifica 10,2 bloques por fila con 4 índices (hoy), 20,2 con 9 (propuesta del compañero) y 12,2 con 5 (V005). Eso son unos 2 bloques más por cada índice. Del compañero rechacé:
- IX_PED_CANAL: 3 valores, el 33 % de la tabla por valor, no viable y sin uso según MONITORING USAGE.
- IX_PED_ESTADO: 4 valores, el 25 % por valor. Q12 lo usaba, pero lo sustituí por el índice parcial IX_PED_PENDIENTES, de 54 hojas frente a las 2933 del índice (estado, fecha) de `20261101T162914Z_03-tipos-indice.script.log`.
- IX_PED_FEC_CLI: sin uso.
- IX_PED_CLI_FEC: sí se usaba (Q7, 7 lecturas), pero lo reemplazó el cubriente IX_PED_CLI_FEC_CUB (3 lecturas).

Solo conservé IX_PED_CP, que se usa y da un 0,009 % de filas por valor. Además, V005 quita IX_PED_IMPORTE (USA = NO; invisible, Q2 sigue en 326 lecturas) e IX_PED_CLIENTE (es prefijo de IX_PED_CLI_FEC_CUB). Coincide con la tabla del enunciado (10,1 / 20,2 / 12,2), con una pequeña diferencia en «hoy» (10,2 en mi caso).

**11. Patrones en los gráficos de la Parte H** (`20261101T163107Z_07a-crecimiento.script.log` y sus gráficos):
- **50/50.** IX_AZAR: 595 divisiones, ninguna 90/10; ocupación del 75,4 %, histograma repartido entre el 40 % y el 99 % y mapa irregular (`=`, `-`, `#` mezclados). La teoría da unos 69 % (ln 2); medí algo más.
- **90/10.** IX_CRECIENTE: 398 divisiones, todas 90/10; ocupación del 99,9 % y mapa uniforme de `#`. Como referencia, IX_INVERTIDA (REVERSE) reparte las claves: 511 divisiones 50/50 y un 77,9 %.
- **Patrón cola.** IX_COLA antes de reparar (`20261101T163107Z_07b-grafico-IX_COLA-antes.script.log`): 471 hojas, 270 de ellas vacías, 83 988 entradas borradas y solo un 42,5 % de ocupación viva. El mapa está vacío a la izquierda y lleno a la derecha (`[   ...   -#####]`).

**12. Remedio para el patrón cola en horario de servicio.**
Elegiría COALESCE. Según `20261101T163107Z_07c-remedios.script.log`, deja el índice en 201 hojas (antes 471) y baja la lectura del «siguiente mensaje» de 272 a 2. Se hace en línea, sin bloquear la tabla y con poco redo. Si además hace falta devolver espacio al tablespace, SHRINK SPACE también es en línea y deja el segmento en 1,69 MB (antes 4). REBUILD ONLINE da 223 hojas, pero necesita espacio doble y genera mucho redo, así que no lo usaría en horario de servicio sin una causa medida. La solución de fondo es de diseño: particionar por fecha y vaciar particiones enteras.

### 8.1.4. SCN, control files, redo y FRA (I a L)

**13. ¿Qué compara Oracle al abrir?**
Compara el SCN de checkpoint que el control file guarda para cada datafile (`v$datafile.checkpoint_change#`) con el que figura en la cabecera del propio archivo (`v$datafile_header.checkpoint_change#`). Si no coinciden, ese archivo necesita recuperación de medios. Si coinciden pero la base no se cerró limpiamente, hace recuperación de instancia con el redo online. En `20261101T163158Z_08-scn-checkpoint.script.log`, antes del checkpoint los dos valían 5 240 040, y después de ALTER SYSTEM CHECKPOINT ambos pasaron a 5 246 725, con current_scn = 5 247 166. Los archivos de PDB$SEED se quedan en 2 098 101 porque están en solo lectura.

**14. Multiplexar el control file en el mismo volumen.**
Según `20261101T163231Z_09-multiplexar-controlfile.script.log`, tengo tres copias idénticas (control01, 02 y 03, de 18 792 448 bytes cada una), todas en `/opt/oracle/oradata/FREE`. Esto protege frente a la pérdida o corrupción de un archivo concreto, como un borrado accidental o un bloque dañado. El simulacro B (`20261101T172648Z_14b-perdida-controlfile.script.log`) lo demuestra: se borró control03.ctl, la base se quedó en NOMOUNT con ORA-00205 y se reparó copiando una copia sana. No protege frente a la pérdida del disco o del volumen, porque caerían las tres a la vez. En producción, cada copia iría en un disco o controladora distintos, por ejemplo una en +DATA y otra en +FRA con ASM, y además se haría copia con RMAN (autobackup del control file).

**15. Estados del redo y borrados rechazados.**
Según `20261101T172530Z_10-redo.script.log`:
- **CURRENT:** el grupo en el que escribe LGWR (al principio, el grupo 1, secuencia 13).
- **ACTIVE:** ya no es el actual, pero aún hace falta para una recuperación de instancia, porque el checkpoint no ha cubierto sus cambios. Tras el primer cambio de log, el grupo 1 pasó a ACTIVE y el 2 a CURRENT; tras el segundo, los grupos 1 y 2 estaban ACTIVE y el 3 CURRENT.
- **INACTIVE:** sus cambios ya están en los datafiles (y, en ARCHIVELOG, ya está archivado), así que se puede reutilizar. Es el caso de los grupos 2 y 3 al principio.

Los tres borrados del paso 4 se rechazan:
- Borrar el grupo CURRENT da ORA-01623, porque LGWR está escribiendo en él.
- Borrar un miembro del grupo CURRENT da ORA-01609.
- Borrar otro grupo me dio ORA-01624 («log 1 needed for crash recovery»), no la ORA-01567 del enunciado, porque en ese momento el grupo 1 estaba ACTIVE y Oracle lo rechaza por eso antes de comprobar el mínimo de dos grupos.

Además, en mi evidencia redo01b.log seguía INVALID tras los dos cambios de log, porque el grupo 1 no había vuelto a ser CURRENT. El comentario del script dice que ya no queda ninguno.

**16. Cadena de causas de la FRA llena.**
Según `20261101T172533Z_11-fra-llena.script.log`:
1. Con la FRA reducida a 2 MB, el alert log avisa con ORA-19815 («db_recovery_file_dest_size of 2097152 bytes is 100.00% used»).
2. Al intentar archivar llega ORA-19809 (límite superado).
3. ARCn no puede copiar el grupo: ORA-16038, «log 2 sequence# 17 cannot be archived». Los grupos 2 y 3 quedan con ARC = NO.
4. En ARCHIVELOG, un grupo no archivado no se puede reutilizar. Cuando LGWR completa la vuelta y necesita ese grupo, el cambio de log se detiene: las sesiones esperan en «log file switch (archiving needed)», las conexiones nuevas reciben ORA-00257 y ninguna DML puede confirmar.

En mi ejecución devolví espacio (paso 5) antes de que LGWR cerrara el círculo. ARCn reintentó y archivó las secuencias 17 y 18, con 2,7 MB usados, así que mi evidencia no contiene el evento de espera ni ORA-00257: ese último eslabón lo explico con la teoría de la Parte L.3, no con una cifra propia.

### 8.1.5. Undo, WAL y simulacros (M a P)

**17. «Lectores no bloquean a escritores».**
Según `20261101T172535Z_12-lectura-consistente.script.log`, la sesión A actualizó la fila 101 a EGRESADO sin confirmar, con bloqueo TX en modo 6 sobre la fila, TM en modo 3 sobre la tabla y 1 bloque y 1 registro de undo (segmento 9).
- **Paso 1.** La sesión B leyó la fila sin esperar y vio ACTIVO, el valor confirmado, reconstruido a partir del undo de A.
- **Pasos 2 y 3.** B consulta en `v$transaction` y `v$lock` la transacción y los bloqueos de A.
- **Paso 4.** B solo choca con A cuando intenta bloquear la misma fila con `FOR UPDATE WAIT 3`, que da ORA-00054.

Es decir, una lectura normal no toma bloqueos ni espera, y la escritura de A tampoco espera a B: solo compiten dos escritores sobre la misma fila.

**18. Undo de DELETE frente a INSERT, y redo del ROLLBACK.**
En `20261101T172605Z_12-undo-y-redo.script.log`, con 10 000 filas, el INSERT dejó 143 bloques de undo (1144 KB) y el DELETE 427 (3416 KB), unas tres veces más. Para deshacer un INSERT basta con apuntar «borra este ROWID», pero para deshacer un DELETE hay que guardar la fila entera y sus entradas de índice. El ROLLBACK también genera redo (355 KB tras el INSERT y 4701 KB tras el DELETE) porque deshacer consiste en aplicar cambios nuevos a los bloques de datos e índice, y todo cambio de bloque se protege con redo para poder rehacerlo si la instancia cae a mitad del ROLLBACK.

**19. Garantía de durabilidad tras COMMIT y tras checkpoint.**
Según `20261101T172606Z_13-lgwr-dbwn.script.log`:
- **Justo después del COMMIT.** LGWR había escrito 1779 KB de redo en 2 escrituras, DBWn 0 bloques, y quedaban 231 bloques sucios de la tabla en la caché. La garantía está en el redo log online, ya multiplexado en dos miembros; los datafiles todavía no tienen el cambio.
- **Después de ALTER SYSTEM CHECKPOINT.** DBWn escribió 350 bloques y quedaron 0 sucios. Los datos ya están en los datafiles, el SCN de checkpoint avanza en el control file y en las cabeceras, y ese redo deja de ser necesario para una recuperación de instancia (aunque sigue haciendo falta, archivado, para una recuperación de medios).

El enunciado muestra 7685 KB, 10 escrituras y 580 bloques. Mis cifras son menores, porque son de toda la instancia, pero el patrón es el mismo.

**20. Simulacro A: roll forward y roll back.**
Según `20261101T172636Z_14a-caida-instancia.script.log`, antes del SHUTDOWN ABORT confirmé la fila 102 → EGRESADO y dejé sin confirmar la 103 → INACTIVO. Al arrancar, el alert log registra «Beginning crash recovery of 1 threads», «Recovery starting at checkpoint rba (logseq 19 block 42337), scn 5251459», «Completed redo application» y «Completed crash recovery». Lo que pasó fue esto:
- **Roll forward.** SMON aplicó el redo online desde el último checkpoint. Ese redo contenía los dos cambios y también su undo.
- **Roll back.** Después se deshizo, con ese undo, la transacción que no tenía COMMIT.

Resultado: la 102 sobrevivió como EGRESADO y la 103 volvió a ACTIVO, sin que yo interviniera. Se aplicaron «0.00MB» de redo porque había pasado muy poco tiempo desde el checkpoint de la Parte N.

**21. RTO y RPO de los simulacros C y D.**
- **C** (`20261101T172658Z_14c-simulacro-restauracion.script.log`): RTO 59 s y RPO igual a todo lo hecho en los 2 s entre la copia y el fallo. Se restaura el volumen entero en frío, así que se para toda la base y se pierde lo posterior a la copia (en producción, horas o días).
- **D** (`20261101T174856Z_14d-perdida-datafile.script.log`): RTO 11 s desde OFFLINE hasta ONLINE (12 s desde el fallo) y RPO 0. La fila 104, cambiada después de la copia RMAN, apareció como EGRESADO.

Son tan distintos porque D restaura solo un datafile (38 MB) y RECOVER rehace con el redo archivado y el online todo lo posterior a la copia. C es una vuelta atrás completa a un punto fijo, sin redo que aplicar. En mi caso Oracle cerró FREEPDB1 (ORA-63999), así que durante esos 12 s estuvo caída toda la PDB y no solo TBS_ACADEMIA.

**22. Errores ORA- del bloque 5 del runbook** (`20261101T175043Z_15-healthcheck.spool.log`):
- **ORA-01116, ORA-01110, ORA-01113 (file 21 needs media recovery) y ORA-63999:** efecto del laboratorio, del simulacro D (borrado de academia01.dbf y RESTORE/RECOVER). Los tres primeros son los del síntoma del runbook y ORA-63999 es el cierre protector de la PDB.
- **ORA-205 y ORA-00202 sobre control03.ctl:** efecto del laboratorio, del simulacro B.
- **ORA-00800 «Set Priority Failed» de LGWR y VKTM:** ruido conocido. El contenedor Docker no tiene permiso para subir la prioridad de esos procesos a tiempo real; aparece en cada arranque y no afecta al funcionamiento.
- **Incidentes reales:** ninguno. Todos los errores coinciden en hora con un simulacro o con un arranque.

## Simulacros: RTO y RPO medidos

| Simulacro | Qué se rompe | Recuperación | RTO medido | RPO real |
|-----------|--------------|--------------|------------|----------|
| A (`14a`) | Caída de instancia (SHUTDOWN ABORT) | Automática: STARTUP y recuperación de instancia por SMON | ≈12 s (el script entero, de 17:26:36 a 17:26:48) | 0: lo confirmado sobrevive |
| B (`14b`) | Se borra control03.ctl | Copiar una copia sana, MOUNT y OPEN | ≈10 s (el script entero, de 17:26:48 a 17:26:58) | 0 |
| C (`14c`) | DROP TABLE PURGE de ACA_MATRICULAS | Restaurar la copia completa en frío del volumen | 59 s | Lo posterior a la copia (2 s) |
| D (`14d`) | Se borra el datafile de TBS_ACADEMIA | OFFLINE (o PDB montada), RESTORE y RECOVER con RMAN, ONLINE | 11 s (12 s desde el fallo) | 0 |

Los scripts A y B no imprimen el RTO; lo he sacado de la marca de tiempo de las evidencias consecutivas, así que es una cota superior.

## Retrospectiva del game day (sin culpables)

- **Qué fue bien.** Los cuatro simulacros recuperaron los datos y `16-verificacion-final.sh` dio TODO OK (`20261101T175045Z_16-verificacion-final.script.log`). La multiplexación convirtió la pérdida de un control file en una copia de un archivo, y ARCHIVELOG + RMAN llevó el RPO de D a 0.
- **Qué me sorprendió.**
  - En D, Oracle cerró la PDB entera (ORA-63999): el RTO afectó a toda FREEPDB1 y no solo al tablespace.
  - La copia lab3-final ya decía «ninguno» en el paso 6, porque `dbs/` es un enlace dentro del volumen (ver la pregunta 1).
  - La base ya estaba en ARCHIVELOG antes de la Parte L.
- **Qué cambiaría.**
  1. Medir el RTO de A y B dentro de los propios scripts, como hacen C y D.
  2. Hacer una copia RMAN programada (no solo la del simulacro), para que el bloque 7 del runbook nunca salga vacío.
  3. En producción, poner las copias del control file y los miembros del redo en discos distintos.
- **Comparación C frente a D.** Restaurar todo es unas 5 veces más lento y pierde datos; restaurar solo lo roto con redo archivado no pierde nada.

## Nota sobre el punto de partida

El Lab 4 lo hice con la primera versión del enunciado (Partes A–J, scripts `lab4/01–10`), no con la v3. Por eso no existía `scripts/lab4/16-verificacion-final.sh`, y la comprobación A.1 no se pudo hacer. Para este laboratorio traje del kit v3 solo lo imprescindible:
- `scripts/ops/grafico-indice.sh`, la versión de `ro.sh` con comprobaciones y `08-aplicar-migraciones.sh` con `MIG_EVID`.
- `scripts/lab4/11-ampliar-ops-ro.sql`, que concede a C##OPS_RO lectura sobre `V$DIAG_ALERT_EXT` y lo exigen el runbook de salud y la verificación final.

Las Partes I–N del Lab 4 v3 quedan sin hacer.

## 9. Simulacro de examen 1Z0-182

| Nº | Respuesta | Justificación | Parte |
|----|-----------|---------------|-------|
| S1 | B | Con un CF cercano al número de filas, casi cada entrada del índice apunta a un bloque distinto (IX_PED_CLIENTE: CF 999 041 para 10⁶ filas). | E |
| S2 | A, C | El índice funcional o reescribir la consulta aplicando la función al dato y no a la columna (Q1: de 1486 lecturas a 4 en ambos casos). | D, F |
| S3 | B | Oracle añade TO_NUMBER("CODIGO_POSTAL") a la columna y el índice queda inutilizado (filtro visto en el plan de Q3). | F |
| S4 | B | El bitmap encaja con baja cardinalidad, carga nocturna y consulta diurna; con OLTP concurrente bloquea rangos de filas (Q14: de 4073 a 528 lecturas). | D |
| S5 | B | Sin la primera columna, el optimizador puede hacer INDEX SKIP SCAN (Q6 solo ciudad: 144 lecturas con SKIP SCAN). | D |
| S6 | B | INVISIBLE hace que el optimizador lo ignore pero lo sigue manteniendo, y se vuelve atrás al instante con VISIBLE (probado con IX_PED_IMPORTE). | G |
| S7 | A, C | COALESCE y SHRINK SPACE son en línea y eliminan las hojas vacías (de 471 a 201 hojas, siguiente mensaje de 272 a 2 lecturas). | H |
| S8 | B | REVERSE reparte las inserciones (511 divisiones 50/50), pero el rango pasa a recorrido completo (de 4 a 878 lecturas). | H |
| S9 | C | Un UPDATE de una columna no indexada solo toca la tabla (UPDATE estado sin índice: 2 bloques por fila; con IX_PED_ESTADO, 6). | G |
| S10 | B | La altura depende del fan-out, que depende del tamaño de la clave: altura = 1 + ⌈log_fo(hojas)⌉ (fan-out de 370 a 543, altura 3). | C |
| S11 | A, C | En una IOT las filas viven en el índice de la PK y la búsqueda no salta a la tabla (Q15: IOT 3 lecturas frente a 4 del montículo). | D |
| S12 | B | Compara el SCN de checkpoint del control file con el de la cabecera del datafile (5 240 040 = 5 240 040, «sincronizado»). | I |
| S13 | C | CKPT anota el checkpoint en el control file y en las cabeceras; DBWn solo escribe los bloques. | I |
| S14 | B, E | En NOMOUNT solo hay instancia (SGA y parámetros): `v$instance` respondió STATUS = STARTED tras ORA-00205; `v$database` y `v$logfile` necesitan el control file montado y `dba_users` la base abierta. | O (simulacro B) |
| S15 | B | ARCHIVELOG se activa en estado MOUNT tras una parada limpia. Mi base ya estaba en ARCHIVELOG al llegar a la Parte L («no hace falta reiniciar»), así que esta secuencia la repasé en la teoría, no la ejecuté en este lab. | L |
| S16 | B | Un grupo INACTIVE y archivado no hace falta para recuperar: CLEAR LOGFILE lo recrea (si no estuviera archivado, sería CLEAR UNARCHIVED LOGFILE). No lo ejecuté; lo relaciono con los estados que observé. | K |
| S17 | A, C | Lectura consistente (B vio ACTIVO mientras A tenía EGRESADO sin confirmar) y Flashback Query AS OF (la fila de hace 40 s). | M |
| S18 | B | El undo vive en bloques de datos y debe poder rehacerse antes de deshacer (redo que describe el undo: 3262 de 6626 KB en el DELETE). | M |
| S19 | A | Tras el COMMIT, el redo está en el log online: 1779 KB escritos por LGWR, 0 bloques de DBWn y 231 bloques sucios aún en memoria. | N |
| S20 | A | Un MTTR objetivo menor fuerza checkpoints incrementales más frecuentes y una recuperación más corta (en mi base fast_start_mttr_target = 0 y ESTIMATED_MTTR = 39 s). | I |
| S21 | B | Ese evento indica que ARCn no puede archivar, típicamente por la FRA llena (ORA-19809 y ORA-16038 en mi evidencia 11). | L |
| S22 | B | Primero se pone OFFLINE el datafile dañado para que el resto siga en servicio; después se hace RESTORE y RECOVER. | O (simulacro D) |
| S23 | B | SYSTEM no puede ir OFFLINE con la base abierta: hay que montar, restaurar, recuperar y abrir. | O (teoría del simulacro D) |
| S24 | B | Sin redo archivado no hay RECOVER completo: solo se puede volver a la última copia en frío y se pierde lo posterior (como en el simulacro C, que restauró la copia entera). | O (simulacro C) |
| S25 | B, D | Más retención y más tamaño de undo, y RETENTION GUARANTEE si la consulta es crítica (undo_retention = 900; ORA_01555 = 0 en v$undostat). | M |
| S26 | A | V$RECOVERY_AREA_USAGE muestra el uso por tipo de archivo (ARCHIVED LOG 0,02 %, 2 archivos). | L |

Notas:
- Las respuestas S15, S16 y S23 no las he comprobado con mis propias manos en este lab (S15 se hizo en un lab anterior; S16 y S23 son teoría). Son las que más debo repasar.
- S22 lo comprobé en el simulacro D (`20261101T174856Z_14d-perdida-datafile.script.log`), aunque en mi caso Oracle ya había cerrado la PDB (ORA-63999) y se reparó con ella montada. S24 se apoya en el simulacro C (`20261101T172658Z_14c-simulacro-restauracion.script.log`: RTO 59 s, se pierde lo posterior a la copia), aunque mi base estaba en ARCHIVELOG.
