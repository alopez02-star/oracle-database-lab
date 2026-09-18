1. **¿Cuál es la diferencia entre Working Directory, Staging Area y Local Repository? Da un ejemplo de un archivo pasando por las tres.**

El Working Directory es el sistema de archivos local donde se editan los documentos. El Staging Area es el índice donde se registran los cambios seleccionados para el próximo commit. El Local Repository es la base de datos local donde se almacena el historial de versiones. Un ejemplo consiste en crear el archivo archivo.txt en el Working Directory. Al ejecutar git add archivo.txt, el archivo se indexa en el Staging Area. Al ejecutar git commit, el estado del archivo se registra de forma permanente en el Local Repository.



**2. Si modificas un archivo pero no haces git add, ¿aparece ese cambio en tu próximo commit? Explica por qué.**

El cambio no se incluye en el próximo commit. Git genera el commit exclusivamente a partir del estado del Staging Area. Si el archivo modificado no se indexa mediante el comando git add, la modificación permanece en el Working Directory y es omitida durante la creación del commit.



**3. ¿Por qué git status no mostraba las carpetas vacías que creaste en la Parte C? ¿Qué truco usamos para solucionarlo?**

Git modela el sistema de control de versiones basándose en archivos, no en directorios. Un directorio vacío carece de archivos para rastrear, por lo que el sistema lo ignora por defecto. La solución técnica consiste en crear un archivo oculto, denominado .gitkeep, dentro del directorio vacío. La existencia de este archivo obliga al sistema a registrar la estructura de directorios subyacente.



**4. Explica con tus palabras qué es HEAD.**

HEAD es un puntero de referencia interno de Git que indica el commit actual sobre el cual se basa el estado del Working Directory. Usualmente apunta a la referencia de la rama activa. En situaciones de revisión del historial, puede apuntar directamente al hash de un commit previo.



**5. ¿Qué diferencia hay entre crear una branch con git switch -c y crear una carpeta nueva con mkdir? ¿Cómo lo comprobamos en la Parte G?**

El comando mkdir instruye al sistema operativo para crear un directorio físico en el sistema de archivos. El comando git switch -c crea un nuevo puntero en el historial de Git sin modificar la estructura de directorios del sistema operativo. Esto se comprueba al alternar entre ramas. Los archivos del Working Directory se actualizan in situ para reflejar el estado de la rama seleccionada sin requerir navegación a diferentes rutas del disco.



**6. Durante el conflicto de la Parte H, ¿qué representaba el contenido entre <<<<<<< HEAD y =======? ¿y entre ======= y >>>>>>>?**

El bloque de código situado entre los marcadores de conflicto de HEAD y el separador de igualdad representa el estado en la rama actual o destino de la fusión. El bloque situado entre el separador de igualdad y el marcador de cierre representa el estado del código en la rama entrante que se intenta integrar.



**7. ¿Por qué NO se debe hacer git commit --amend sobre un commit que ya se subió con git push?**

La ejecución de un amend altera el objeto commit y genera un nuevo hash criptográfico, reescribiendo el historial. Si el commit original ya ha sido propagado al repositorio remoto, la alteración local provocará una divergencia temporal. Esto generará conflictos de integración para cualquier otro desarrollador que haya sincronizado la versión previa.



**8. Si borras por accidente la carpeta .git de tu proyecto, ¿qué se pierde exactamente? ¿Se pierde también el código fuente que está en el disco?**

La eliminación del directorio .git destruye la base de datos de objetos, las referencias de ramas y los registros de configuración del repositorio. El código fuente presente en el Working Directory no se ve afectado. Los archivos físicos permanecen en el sistema de almacenamiento, pero el directorio deja de poseer control de versiones.



**9. Explica con tus propias palabras la diferencia entre Git y GitHub, sin usar la palabra nube.**

Git es el sistema de control de versiones distribuido que se ejecuta localmente en el equipo para gestionar el historial de código. GitHub es una plataforma de alojamiento web externa que proporciona servidores para almacenar repositorios Git remotos. Esta arquitectura facilita la colaboración distribuida y la sincronización entre múltiples equipos.



**10. ¿Por qué no se debe subir un archivo .env con contraseñas reales a un repositorio, aunque el repositorio sea privado?**

Incluir archivos de configuración con credenciales en texto plano expone los datos a todos los usuarios con permisos de lectura actuales o futuros. Existe el riesgo operativo de que el repositorio altere su nivel de privacidad a público de forma accidental. Las credenciales deben inyectarse de forma segura mediante variables de entorno del sistema host.



**11. Un compañero te dice: hice push y ahora GitHub me rechaza el segundo push con el error non-fast-forward. ¿Qué ha ocurrido probablemente y qué comando ejecutarías primero?**

El error non-fast-forward indica que el puntero de la rama remota ha avanzado independientemente y contiene commits ausentes en el repositorio local. Para resolver esta divergencia, el primer comando a ejecutar es git pull para descargar el estado remoto e integrarlo en el historial local antes de intentar una nueva propagación.



**12. ¿Qué tipo de Conventional Commit usarías para: añadir un índice de rendimiento a una tabla, corregir una restricción mal definida, y actualizar el README?**

Para añadir un índice de rendimiento a una tabla se debe usar la etiqueta perf. Para corregir una restricción mal definida se debe usar la etiqueta fix. Para actualizar el archivo README se debe usar la etiqueta docs.

