# Lab 3 — Respuestas de comprobación

## Docker

### 1. ¿Qué diferencia hay entre una imagen y un contenedor? Usa como ejemplo lo que hiciste en los ejercicios G2 y G4.

Una imagen es una plantilla de solo lectura (el sistema de archivos y el programa que se va a ejecutar) y un contenedor es una ejecución concreta de esa imagen, con su propio nombre, su propio ID y su propia capa de escritura. En el G2 ejecuté `docker run hello-world`: la imagen `hello-world` ya estaba en mi equipo y Docker creó a partir de ella un contenedor con un nombre aleatorio (`objective_edison`) que aparecía en `docker ps -a`. En el G4 hice `docker run -it --name prueba alpine:3.20 sh`: de la imagen `alpine:3.20` salió el contenedor `prueba`, y dentro `hostname` devolvía su ID (`fec8f3fb6b82`), no el de la imagen. De una misma imagen puedo crear muchos contenedores (luego creé `prueba2` con la misma `alpine:3.20`).

### 2. En el Ejercicio G5 el archivo nota.txt desapareció y en el G6 no. Explica por qué.

En el G5 escribí `/tmp/nota.txt` dentro del contenedor `prueba`, es decir, en su capa de escritura, que pertenece solo a ese contenedor. Al hacer `docker rm prueba` esa capa se borró con él, y el contenedor nuevo `prueba2` partía otra vez de la imagen limpia, así que `cat` dio "No such file or directory". En el G6 escribí el archivo en `/datos`, donde estaba montado el volumen `datos-prueba`. El volumen vive fuera del contenedor, así que aunque los dos contenedores fueran `--rm` y se borraran solos, el segundo leyó "dato importante". Es la misma razón por la que Oracle guarda sus datos en el volumen `oralab-26ai-data`.

### 3. ¿Qué diferencia hay entre docker ps y docker ps -a, y qué significa STATUS = Exited (0)?

`docker ps` solo muestra los contenedores que están en marcha; `docker ps -a` muestra todos, también los parados. En el G2, después de `hello-world`, `docker ps` salía vacío y `docker ps -a` mostraba el contenedor. `Exited (0)` significa que el contenedor está parado y que su proceso principal terminó con código 0, es decir, sin error. Un número distinto de 0 indicaría que terminó con un fallo.

### 4. En -p 8181:8181, ¿qué número corresponde a tu equipo y cuál al contenedor? ¿Qué pasaría con -p 80:8080 en el ejercicio de nginx?

El formato es `anfitrión:contenedor`: el primer número es el puerto de mi equipo y el segundo el del contenedor (en este caso coinciden). En el G7 usé `-p 8080:80` porque nginx escucha en el 80 dentro del contenedor y yo quería entrar desde el 8080 de mi equipo (http://localhost:8080). Con `-p 80:8080` lo estaría haciendo al revés: publicaría el puerto 80 de mi equipo hacia el 8080 del contenedor, donde nginx no escucha nada, así que la página no cargaría.

### 5. ¿Por qué un contenedor de Oracle se queda en marcha y el de hello-world termina solo?

Un contenedor vive lo que vive su proceso principal. El de `hello-world` solo imprime un mensaje y termina, así que el contenedor pasa a `Exited (0)` enseguida. El proceso principal del contenedor de Oracle es la base de datos (el listener y la instancia FREE), que es un servicio pensado para estar siempre esperando conexiones, así que `oralab-26ai` sigue `Up` hasta que lo paro. Lo mismo vi con alpine en el G4: siguió vivo mientras estuvo abierta la shell `sh` y se paró al hacer `exit`.

### 6. ¿Qué es el digest de una imagen y por qué lo registramos si ya sabemos que usamos :latest?

El digest es la huella exacta (un hash `sha256`) del contenido de una imagen. La etiqueta `:latest` es solo un nombre que Oracle mueve a la versión más nueva cada vez que publica una, así que dentro de un mes `:latest` puede ser otra imagen distinta. En la Parte H registré el digest de la mía, `sha256:f988b0c04c4c…`, en la evidencia 04: así cualquiera puede saber y reproducir exactamente qué versión de Oracle usé.

### 7. ¿Qué comando borraría realmente los datos de Oracle? ¿Por qué docker rm oralab-26ai no lo hace?

Lo que borraría los datos es `docker volume rm oralab-26ai-data` (o un `docker system prune -a --volumes`), porque los archivos de la base de datos están en el volumen montado en `/opt/oracle/oradata`. `docker rm oralab-26ai` solo borra el contenedor, igual que en el G6: el volumen sigue existiendo y, si vuelvo a crear el contenedor con `05-crear-contenedor.sh`, lo vuelve a montar con los datos. Eso sí, en mi caso hay una excepción: al aplicar V000 comprobé que los datafiles de los 5 tablespaces de negocio se crearon en `$ORACLE_HOME/dbs`, dentro del contenedor y no en el volumen, porque V000 usa nombres relativos (`'academia01.dbf'`). Así que en mi entorno un `docker rm` sí haría perder esos 5 tablespaces; por eso no debo borrar el contenedor, solo pararlo y arrancarlo.

## Git, organización y evidencia

### 8. ¿Por qué este laboratorio se hace dentro del repositorio oracle-database-lab, con Issue, branch y Pull Request, en vez de en una carpeta aparte?

Porque así la instalación queda versionada, revisada y trazable, como en un equipo real. El Issue #5 dice qué hay que hacer y con qué criterios de aceptación; la branch `chore/5-install-oracle-environment` aísla el trabajo sin tocar `main`; cada commit con `Refs #5` deja registrado cuándo y cómo hice cada Parte, y el Pull Request permite que un compañero lo revise antes de fusionarlo. Además, los scripts quedan en el repositorio y cualquiera puede reconstruir el mismo entorno; en una carpeta aparte sería un entorno "mágico" que nadie más sabría montar.

### 9. ¿Qué diferencia hay entre source 00-config.sh y bash 00-config.sh? ¿Por qué usamos source?

`bash 00-config.sh` ejecuta el script en una shell nueva (un proceso hijo): las variables se definen ahí y desaparecen al terminar, así que mi terminal no se entera. `source 00-config.sh` ejecuta las líneas en la shell actual, así que `CONT_NAME`, `EVID`, la función `ts()`, etc. quedan disponibles en mi terminal y en los comandos que lanzo después. Por eso hay que hacer `source` en cada terminal nueva; si no, los nombres de evidencia salen sin marca de tiempo.

### 10. Explica cada parte del nombre 20260915T091230Z_02-docker.script.log.

- `20260915`: la fecha, 15 de septiembre de 2026 (año, mes, día).
- `T`: separa la fecha de la hora (formato ISO 8601).
- `091230`: la hora, 09:12:30.
- `Z`: la hora está en UTC (la genera `ts()` con `date -u`), así no depende de la zona horaria de cada uno.
- `_02`: el número de paso de la evidencia (02 = verificar Docker).
- `-docker`: descripción corta de lo que contiene.
- `.script.log`: el tipo de evidencia: salida de terminal capturada con `tee` o `script` (las de SQL son `.spool.log`).

Al empezar por la fecha, los archivos se ordenan solos cronológicamente.

### 11. ¿Para qué sirve .gitattributes y qué error evita?

Le dice a Git cómo tratar los finales de línea. En mi `.gitattributes` puse `eol=lf` para `*.sh`, `*.sql` y `*.md`, así que esos archivos siempre quedan con finales de línea de Linux (LF) aunque se editen en Windows, que usa CRLF. Evita el error `$'\r': command not found` al ejecutar un `.sh` con CRLF en Linux, porque bash interpreta el `\r` como parte del comando. Lo comprobé con `git ls-files --eol` en cada script (`i/lf w/lf`).

### 12. ¿Por qué en este Pull Request elegimos Create a merge commit en lugar de Squash and merge?

Porque aquí cada commit tiene valor por sí mismo: hay uno o varios por cada Parte (requisitos, Docker, imagen, contenedor, migraciones, ORDS, verificación final...), cada uno con su evidencia. Con *Create a merge commit* ese historial paso a paso se conserva en `main` y se puede ver cuándo y cómo se verificó cada herramienta. *Squash and merge* juntaría mis 30 commits en uno solo y se perdería esa trazabilidad.

## Seguridad

### 13. Describe las cuatro capas de la estrategia de contraseñas (Parte D) y qué pasaría si te saltas la primera.

1. `.gitignore` con `config/.env`, creado **antes** que el archivo con las contraseñas, para que Git no pueda verlo nunca.
2. Una plantilla versionada, `config/.env.example`, con los nombres de las variables y valores de ejemplo (`change_me`), que documenta qué secretos hacen falta sin revelar ninguno.
3. El archivo real `config/.env`, copiado de la plantilla, con mis contraseñas y permisos `600` (solo mi usuario puede leerlo).
4. Los scripts cargan las contraseñas con `set -a; source config/.env; set +a` y las usan como `"$ORACLE_PWD"`, de modo que nunca se escriben a mano ni quedan en el historial de la terminal.

Si me salto la primera, `config/.env` aparecería en `git status` y con un `git add .` las contraseñas acabarían en un commit, y por tanto en GitHub y en el historial para siempre.

### 14. ¿Por qué no escribimos la contraseña directamente en el comando docker run, aunque el script no se suba a Git?

Porque escribirla en un comando la deja guardada en otros sitios: en `~/.bash_history`, en el propio script (que alguien podría subir o compartir por error) y en evidencias o capturas de pantalla de ese comando. En mi `05-crear-contenedor.sh` la contraseña llega desde `config/.env` como `"$ORACLE_PWD"`, y antes de subir cada evidencia comprobé que no aparecía en ella. Así hay un único sitio donde vive el secreto.

### 15. Si descubres tu contraseña en un commit ya publicado, ¿basta con borrarla en un commit nuevo? ¿Qué debes hacer?

No basta: aunque la borre en un commit nuevo, sigue en el historial de Git y cualquiera puede verla con `git log -p` (y GitHub, además, escanea los repositorios buscando secretos). Hay que darla por comprometida y **rotarla**: cambiar la contraseña (en mi caso, poner una nueva en `config/.env` y recrear el contenedor), y avisar al docente para limpiar la branch. Lo mejor es evitarlo antes del push con el control de la O.2 (`git grep -F` y `git log -p | grep -c -F`), que en mi caso dio 0 para las tres contraseñas.

## Oracle y herramientas

### 16. ¿Por qué no usamos SPOOL ni @archivo.sql con sqlplus dentro del contenedor, y qué hicimos en su lugar?

Porque `sqlplus` se ejecuta dentro del contenedor y ve su sistema de archivos, no el de mi Ubuntu. Un `SPOOL` escribiría el archivo dentro del contenedor y no en mi repositorio, y `@archivo.sql` buscaría el archivo dentro del contenedor, donde no existe (error SP2-0310). En su lugar usé `docker exec -i ... sqlplus ... < archivo.sql | tee evidencia.spool.log`: con `<` el `.sql` de mi equipo entra por la entrada estándar y con `tee` la salida se guarda en mi repositorio (así hice la evidencia 07 y las 08). En la Parte L, con SQLcl instalado en mi Ubuntu, sí usé `SPOOL` normal para la evidencia 10.

### 17. ¿Qué hace WHENEVER SQLERROR EXIT SQL.SQLCODE al inicio de V000 y V001, y qué pasaría sin esa línea?

Hace que, si cualquier sentencia falla, SQL*Plus se detenga en ese punto y salga con el código del error. Así `08-aplicar-migraciones.sh` detecta el fallo y no ejecuta V001 sobre un V000 a medias. Sin esa línea, SQL*Plus mostraría el error y seguiría con las siguientes sentencias, dejando la base en un estado a medias (por ejemplo, un usuario creado sin su tablespace) y el script parecería haber terminado bien.

### 18. ¿Qué es una migración y por qué V000 y V001 no se deben editar una vez aplicadas?

Una migración es un script SQL versionado y numerado que lleva la base de datos de un estado al siguiente. Se aplican en orden (V000 crea los tablespaces y usuarios, V001 las tablas) y así todos los compañeros tenemos exactamente la misma base. No se editan una vez aplicadas porque las bases donde ya se ejecutaron no volverían a ejecutarlas: el archivo dejaría de reflejar lo que hay de verdad en la base y cada entorno quedaría distinto. Si hay que corregir algo, se crea una migración nueva (V002...). Por ejemplo, para mover los datafiles de mi V000 al volumen habría que hacerlo en una migración nueva, no cambiando V000.

### 19. ¿Por qué en SQL Developer se usa el servicio FREEPDB1 y no FREE ni un SID?

Porque Oracle 26ai es multitenant: `FREE` es el contenedor raíz (CDB) y `FREEPDB1` es la base de datos conectable (PDB) donde está todo nuestro trabajo (los 5 usuarios de negocio, ALUMNO, ORDS). Si me conecto a `FREE` o por SID, entro en la raíz, donde no están nuestros esquemas. Con el nombre de servicio `FREEPDB1` entro directamente en la PDB, como confirmé en la Parte J con `SHOW CON_NAME` → `FREEPDB1`. En SQL Developer marqué "Nombre del Servicio" = `FREEPDB1` y el test dio "Estado: Correcto".

### 20. ¿Qué aporta SQLcl frente a SQL*Plus, y por qué un DBA debe dominar ambas?

SQLcl es la herramienta moderna: tiene autocompletado, historial, formato automático (`SET SQLFORMAT ansiconsole`), conexiones guardadas (`CONNECT -save oralab26-system`, `CONNMGR list`) e integración con Liquibase. Además la instalé en mi Ubuntu, así que su `SPOOL` escribe directamente en mi repositorio. SQL*Plus es la clásica: viene con cualquier instalación de Oracle (la usé dentro del contenedor en las Partes J, K y M). Un DBA necesita las dos porque en el día a día trabajará más cómodo con SQLcl, pero en un servidor donde solo hay una terminal (por ejemplo, de madrugada ante un problema) lo único que seguro tendrá es SQL*Plus.

## Entorno de trabajo

### 21. ¿Por qué el curso pasa de Git Bash a Ubuntu en WSL 2? Da al menos dos problemas concretos de Git Bash que desaparecen en Ubuntu.

Porque Oracle y Docker se usan en servidores Linux, y Ubuntu en WSL 2 es un Linux real, no una emulación. Problemas de Git Bash que desaparecen:
- Los finales de línea: en Windows los archivos acaban con CRLF y los scripts fallan con `$'\r': command not found`; en Ubuntu se trabaja en LF de forma natural.
- Las rutas y conversiones de Git Bash (`C:\...` frente a `/c/...`), que rompen comandos como `docker run -v` o rutas dentro de los contenedores.
- No todas las herramientas existen o se comportan igual: en Ubuntu tengo `apt` para instalar todo lo que usé (shellcheck, tmux, Java 21...), y Docker Desktop se integra directamente con la distribución.

### 22. ¿Por qué clonamos el repositorio en ~/oracle-database-lab y no trabajamos sobre la carpeta de Windows (/mnt/c/...)? ¿Y por qué recomendamos bash frente a zsh para los scripts del curso?

Porque `/mnt/c` es el disco de Windows visto desde Linux a través de una capa de traducción: es mucho más lento, los permisos de los archivos no funcionan bien (todo aparece como `rwxrwxrwx`) y vuelven los problemas de finales de línea. En `~/oracle-database-lab` el repositorio está en el sistema de archivos de Linux: es rápido y los permisos funcionan (por ejemplo, el `chmod 600` de `config/.env`). Yo solo usé `/mnt/c` para copiar archivos puntuales, como las capturas.

Bash se recomienda porque es la shell por defecto en Ubuntu y en casi todos los servidores Linux, y todos los scripts del curso empiezan con `#!/usr/bin/env bash`. zsh es parecida pero no igual (por ejemplo, en arrays y en algunas expansiones), y `shellcheck`, que usé en la Parte N, analiza scripts de bash, no de zsh. Así todos ejecutamos los scripts igual.
