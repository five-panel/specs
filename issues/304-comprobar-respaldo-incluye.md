# Issue #304: bug: Comprobar que el respaldo incluye los últimos cambios

Source issue: https://github.com/benitogonzalezh/five-panel/issues/304

## TL;DR

Comprobar que un respaldo conserva los cambios guardados antes de iniciarlo y permite recuperarlos. Seguir la misma copia desde su generación hasta su restauración para detectar datos omitidos, archivos antiguos o restauraciones incompletas.

El reporte original sigue sin un archivo ni un cambio concreto identificado. Los fallos encontrados deben distinguirse de la causa de ese reporte.

## What will change

**Impacto de código: Medium.** Cambios enfocados en los scripts de respaldo y restauración, la selección de copias y sus pruebas.

Reutilizar el CLI raíz (`npm run cli`), los scripts de `ops/`, PostgreSQL y el cliente de Spaces existentes. El inventario se contrasta con `backend/src/db/types.ts`, las migraciones y los servicios que guardan datos. Debe cubrir empresa y configuración; modelos y registros; usuarios y accesos; vistas, navegación y permisos; integraciones y credenciales; tareas programadas y su historial; plantillas; automatizaciones y sus ejecuciones; historial de cambios; e información de archivos adjuntos.

El respaldo por empresa omite plantillas, automatizaciones, ejecuciones de automatizaciones e historial de cambios. Corregir las plantillas en esta tarea. La corrección de automatizaciones e historial pertenece a [#290](https://github.com/five-panel/five-panel/issues/290), pero su conservación también se verifica aquí; completar esa comprobación depende de resolverla.

Corregir la lectura del respaldo completo `.tar.gz`: el generador guarda los SQL dentro de una carpeta y el restaurador los busca fuera. Una prueba aislada del lector confirmó que anuncia éxito sin aplicar esos SQL; todavía falta la prueba con una base real. Revisar también los errores que puedan acabar anunciando una copia completa o una restauración correcta sin serlo.

Añadir pruebas de comportamiento en `ops/test/` para los fallos corregidos y la selección de la copia más reciente.

## How it would work

1. Preparar bases de origen y destino aisladas con Docker Compose. Guardar valores reconocibles de los grupos del inventario y comprobarlos en el origen. Registrar empresa, contenedor, base, hora y valores esperados. Evitar escrituras durante esta comprobación.
2. Generar las copias con los flujos existentes: `ops/backup-tenant.sh`, su ejecución desde `ops/backup-all-tenants.sh` y `ops/backup-full-db.sh`. Comparar el contenido generado con el origen, sin dar por completa la lista fija actual de tablas.
3. Cuando se use Spaces, comprobar que se sube exactamente el archivo recién generado y que la descarga conserva el mismo contenido. En `ops/tenant-cli.ts`, comprobar todo el listado antes de elegir “Latest backup”: usar la fecha de generación del nombre, filtrar por tipo y empresa, y mostrar el archivo elegido. Volver a subir una copia antigua no la convierte en la más reciente. Las copias sin fecha reconocible quedan para selección manual.
4. Restaurar mediante el CLI raíz y los scripts existentes `ops/restore-tenant.sh` y `ops/restore-full-db.sh`. Probar archivos locales y descargados, los dos formatos completos (`.sql.gz` y `.tar.gz`), y la restauración por empresa con un destino vacío y con limpieza de datos anteriores. Comparar valores, accesos y configuración; usar la clave de prueba correspondiente para comprobar las credenciales guardadas.
5. Documentar dónde se pierden los cambios o se elige otro archivo. Las pruebas nuevas deben demostrar el fallo antes de su corrección y verificar los valores recuperados después. Conservar la compatibilidad existente con copias anteriores.

## Done when

- Hay un caso documentado con origen, empresa, archivo, horas, valores esperados y recuperados, y el paso donde falla el recorrido.
- Una prueba real de respaldo y restauración conserva los datos del inventario, incluidas plantillas, automatizaciones e historial; la parte por empresa vinculada a #290 está verificada tras su corrección.
- La copia subida y descargada de Spaces coincide con el archivo generado. “Latest backup” elige la copia correcta con varias empresas, tipos, fechas y páginas de resultados, incluso si se vuelve a subir una copia antigua.
- Ambos formatos completos recuperan los valores esperados. Un archivo sin SQL utilizable o un fallo que impida exportar, descargar o restaurar datos devuelve un error y no anuncia éxito.
- La restauración por empresa con limpieza conserva los usuarios compartidos y los datos de las demás empresas. Las copias anteriores compatibles siguen pudiendo restaurarse.
- Las pruebas enfocadas de los fallos corregidos y las comprobaciones existentes de ops pasan.

## Out of scope

- Implementar la corrección de automatizaciones e historial de #290 o cambiar cómo se ejecutan las automatizaciones.
- Incluir archivos físicos de `uploads`, claves del servidor o configuración del entorno en el respaldo de base de datos. La información de los adjuntos sí se comprueba.
- Recuperar sesiones de acceso, enlaces temporales de recuperación de contraseña o reanudar tareas que estuvieran en ejecución.
- Garantizar cambios realizados mientras se genera el respaldo, rediseñar el sistema, modificar scripts históricos de empresas o cambiar datos de producción.
