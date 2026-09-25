**1. ¿Por qué un Issue sin criterios de aceptación es un problema, aunque la descripción general parezca clara?**

Sin criterios de aceptación no existe una forma objetiva de saber cuándo el trabajo está terminado. Cada persona interpreta la descripción a su manera, el autor puede entregar menos de lo esperado y el reviewer no tiene una lista contra la que comprobar el diff. Los criterios convierten una intención vaga en un contrato verificable. En este laboratorio, los cuatro criterios del Issue #1 fueron la checklist que usó el reviewer antes de aprobar.



**2. Explica la diferencia entre "Refs #N" y "Closes #N" en un mensaje de commit o en la descripción de un Pull Request.**

Refs #N solo crea una referencia cruzada: el commit o PR aparece enlazado en el Issue, pero el Issue sigue abierto. Closes #N (igual que Fixes #N o Resolves #N) es una palabra clave de cierre: GitHub cierra el Issue automáticamente cuando el commit o PR que la contiene llega a la branch principal. Por eso durante el trabajo se usa Refs en los commits, y Closes en la descripción del PR. El cierre ocurre en el merge, no en el momento de escribir la palabra.



**3. ¿Qué ocurre exactamente si intentas hacer git push directamente sobre una branch main protegida? ¿Es un error tuyo o un fallo del sistema?**

El commit se crea en local, pero GitHub rechaza el push con el error "GH006: Protected branch update failed for refs/heads/main. Changes must be made through a pull request", y main en el remoto no cambia. No es un fallo del sistema: es la regla de protección funcionando como se configuró. Es un error del proceso de quien lo intenta, y la solución nunca es forzar el push, sino crear una branch y abrir un Pull Request. Hay que deshacer el commit local con git reset --hard HEAD~1.



**4. Un compañero te dice: "he aprobado el PR sin mirar los archivos, total ya me fío". ¿Qué riesgo tiene esa forma de revisar?**

Es "rubber-stamping": la aprobación deja de ser una comprobación y se convierte en un trámite. Pueden entrar en main errores, datos sensibles o cambios accidentales que nadie ha leído, y además con la apariencia de haber sido revisados, lo que es peor que no tener revisión. En nuestro PR #2 se coló una línea de comando (cat > CONTRIBUTING.md << 'EOF') en el primer commit; solo se detecta si alguien mira de verdad el diff en Files changed.



**5. Si el reviewer pide un cambio y tú ya habías hecho push de tu branch, ¿tienes que abrir un Pull Request nuevo? Explica qué ocurre técnicamente con el PR existente cuando haces un nuevo commit.**

No. Un Pull Request no guarda una copia de los cambios, sino que apunta a la head branch. Al hacer commit y push sobre la misma branch, el nuevo commit aparece automáticamente en el mismo PR, en las pestañas Commits y Files changed se actualiza el diff, y la conversación y el historial de revisión se conservan. Si está activada la regla de descartar aprobaciones obsoletas, una aprobación previa se invalida porque el diff ha cambiado.



**6. Describe con tus palabras la diferencia entre Merge commit, Squash and merge y Rebase and merge. ¿Cuál usarías para una branch con commits "wip", "fix", "fix2", "ok ya"?**

Merge commit conserva todos los commits de la branch y añade un commit de fusión extra. Squash and merge combina todos los commits de la branch en uno solo sobre main. Rebase and merge reaplica los commits uno a uno sobre main, sin commit de fusión, dejando un historial lineal. Para una branch con commits "wip", "fix", "fix2", "ok ya" usaría Squash and merge, porque esos commits no aportan valor individual y ensuciarían el historial de main; lo importante es un único commit que describa el cambio completo.



**7. ¿Por qué borrar una branch después del merge no elimina el trabajo realizado en ella?**

Porque una branch es solo un puntero a un commit, no una copia de los archivos. Tras el merge, el trabajo ya forma parte del historial de main (en nuestro caso, en el commit de squash 23e9711), así que borrar la branch elimina únicamente ese puntero que ya no hace falta. Los commits integrados siguen en main y el PR conserva el registro de todo lo que ocurrió.



**8. ¿Qué información debería contener siempre la descripción de un Pull Request, como mínimo?**

Qué hace el cambio (Summary), qué se ha modificado (Changes), cómo se ha probado (Testing) y qué necesidad resuelve con el enlace al Issue (Related Issue, por ejemplo Closes #1). Cuando el cambio es visual, también evidencia del antes y el después. Con esto el reviewer entiende el PR sin tener que preguntar, lo que reduce las idas y vueltas en una revisión asíncrona.



**9. Un reviewer escribe solo "esto está mal" como comentario. ¿Qué le falta a ese comentario para ser útil? Reescríbelo tú con un ejemplo inventado.**

Le falta decir qué observa, por qué importa, qué propone para solucionarlo y si bloquea o no el merge. Tal como está, genera una respuesta defensiva y el autor no sabe qué hacer. Ejemplo reescrito: "issue (blocking): esta consulta asume que la tabla customers nunca está vacía; si count(*) = 0 la división da error. Propongo comprobar el recuento antes de calcular la media."



**10. ¿Qué diferencia hay entre que main esté protegida y que simplemente el equipo "se ponga de acuerdo" en no hacer push directo?**

Un acuerdo es una política que depende de la disciplina y la memoria de cada persona: basta un despiste o una prisa para saltárselo. Una branch protegida es la misma política aplicada técnicamente: GitHub rechaza el push directo y bloquea el merge hasta que haya PR y aprobación, sin excepciones. Lo comprobamos en la Parte H, donde el push directo a main fue rechazado con el error GH006.



**11. Explica con un ejemplo propio la diferencia entre un comentario issue: (blocking) y uno nitpick: (if-minor) en formato Conventional Comments.**

Un issue (blocking) señala un problema que debe resolverse antes de aprobar. Por ejemplo: "issue (blocking): el script de migración no tiene su rollback correspondiente en database/rollback/; sin él no podemos deshacer el cambio en producción." Un nitpick (if-minor) es un detalle de estilo que no debería bloquear y se deja a criterio del autor. Por ejemplo: "nitpick (if-minor): el resto de títulos del documento usan mayúscula inicial; este podría seguir el mismo estilo."



**12. Si tu próximo commit es "feat!: cambia la firma de la función principal de la API", ¿qué tipo de versión SemVer se dispara y por qué?**

Se dispara una versión MAJOR (por ejemplo, de 2.1.0 a 3.0.0). Aunque el tipo sea feat, el signo ! indica un BREAKING CHANGE: cambiar la firma de la función principal rompe la compatibilidad con el código que ya la usa, y SemVer exige incrementar la versión mayor cuando un cambio no es compatible hacia atrás.



**13. ¿Por qué abrir un Draft PR desde el primer commit estructural puede ahorrar tiempo al equipo, aunque parezca más lento para el autor individual?**

Porque permite validar el enfoque general antes de invertir horas en la lógica detallada y los tests (Fail Fast). Si el diseño es equivocado, se detecta cuando corregirlo todavía es barato, en lugar de al final, cuando el autor tiende a defender un mal diseño por las horas ya invertidas (falacia del coste hundido). Dedicar un rato a recibir feedback temprano evita reescribir días de trabajo después.
