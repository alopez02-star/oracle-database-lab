# Runbook: pérdida de un datafile de aplicación

Para el compañero de guardia. Probado en el simulacro D del Lab 5 (`scripts/lab5/14d-perdida-datafile.sh`): RTO 12 s, RPO 0.

## 1. Síntoma

Las consultas sobre un tablespace de aplicación fallan con:

```
ORA-01116: error in opening database file <n>
ORA-01110: data file <n>: '/opt/oracle/oradata/FREE/FREEPDB1/<archivo>.dbf'
ORA-27041: unable to open file
```

Si en el alert log aparece **ORA-63999**, Oracle ha cerrado la PDB entera para protegerla (queda `MOUNTED`) y todo FREEPDB1 está caído, no solo ese tablespace.

## 2. Comprobaciones previas (no tocar nada todavía)

```sql
-- sqlplus / as sysdba (docker exec -it oralab-26ai sqlplus / as sysdba)
SELECT log_mode FROM v$database;                    -- debe ser ARCHIVELOG
SELECT name, open_mode FROM v$pdbs;                 -- ¿FREEPDB1 READ WRITE o MOUNTED?
SELECT file#, name FROM v$datafile WHERE file# = <n>;
```

```
RMAN> LIST BACKUP OF DATAFILE <n>;                  -- debe haber al menos una copia
```

- **NOARCHIVELOG o sin copia**: PARAR y escalar. Solo queda restaurar la última copia en frío (`scripts/ops/restaurar-estado.sh`), perdiendo todo lo posterior.
- Si el datafile es de **SYSTEM o undo de la raíz**, este runbook no sirve: `SHUTDOWN ABORT`, `STARTUP MOUNT`, `RESTORE`, `RECOVER`, `OPEN`.

## 3. Reparación

**Camino A: la PDB sigue en READ WRITE** (solo cae ese tablespace y el resto del servicio sigue funcionando):

```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
ALTER DATABASE DATAFILE <n> OFFLINE;
```
```
RMAN> RESTORE DATAFILE <n>;
RMAN> RECOVER DATAFILE <n>;
```
```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
ALTER DATABASE DATAFILE <n> ONLINE;
```

**Camino B: la PDB está MOUNTED (ORA-63999)**. Hay que esperar a que quede completamente `MOUNTED`, porque un cierre a medias y un `RESTORE` a la vez se pisan. Después:

```
RMAN> RESTORE DATAFILE <n>;
RMAN> RECOVER DATAFILE <n>;
```
```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
ALTER DATABASE DATAFILE <n> ONLINE;
ALTER SESSION SET CONTAINER = CDB$ROOT;
ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;
```

## 4. Verificación

```sql
SELECT name, open_mode FROM v$pdbs;                          -- FREEPDB1 READ WRITE
SELECT file#, status FROM v$datafile WHERE file# = <n>;      -- ONLINE
SELECT COUNT(*) FROM <tabla de control del tablespace>;      -- responde sin error
```

En el simulacro, la fila de control (estudiante 104), cambiada **después** de la copia, apareció con su valor nuevo: `RECOVER` aplicó el redo archivado y no se perdió nada (RPO 0).

## 5. A quién avisar

1. Al empezar: al responsable de la aplicación del tablespace, indicando qué está caído (solo el tablespace o toda la PDB).
2. Al terminar: al equipo de sistemas, porque un archivo que desaparece apunta a un disco o volumen defectuoso. Hay que revisar el almacenamiento antes de que se repita.
3. En las 24 h siguientes: hacer una copia RMAN nueva y escribir la retrospectiva sin culpables, con el RTO, el RPO y la causa raíz.
