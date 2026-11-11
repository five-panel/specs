# Shopify #279: cómo lo implementaríamos

Qué piezas construiremos, qué código actual cambia y cómo se conectará todo. Complementa la [spec](279-create-integration-shopify.md) y el [plan técnico detallado](../docs/superpowers/plans/2026-11-11-shopify-integration.md).

## Qué aprovechamos y qué cambia

La base será el sistema actual de automatizaciones y su cola de Graphile, que organiza trabajos en segundo plano. Reutilizaremos `DataService` para guardar registros, las credenciales protegidas y la programación de tareas.

Haremos cambios concretos en estas partes:

| Parte actual | Cambio necesario |
| --- | --- |
| Configuración y credenciales | Ampliarlas para guardar conexiones y sus acciones, con credenciales propias por conexión. |
| Automatizaciones | Agregar operaciones compartidas y permitir continuar una importación en otra ejecución de forma controlada. |
| Guardado de registros | Proteger los campos elegidos para los IDs externos y permitir que la integración los complete. |
| Historial y ajustes | Agregar la actividad de la conexión y las pantallas del diseño. |

El impacto total es **grande**, principalmente por las funciones nuevas. Los cambios en servicios actuales serán puntuales y conservarán las integraciones existentes.

## Qué código agregamos

El código compartido manejará conexiones, solicitudes HTTP/GraphQL, mapeo, importación y sincronización. Cada acción guardada indicará a qué dirección llamar, qué enviar y cómo leer la respuesta. Agregar una acción no exigirá código nuevo.

Las operaciones principales serán:

| Operación | Qué hace |
| --- | --- |
| `integration.request` | Ejecuta una acción de la conexión usando sus credenciales. |
| `data.map` | Transforma datos entre campos cuando hace falta un paso separado. |
| `import.save_batch` | Guarda un lote y su avance. |
| `automation.enqueue` | Pone otra ejecución en la cola con los parámetros recibidos. |

Shopify tendrá código propio para autorizar la app, comprobar sus avisos y aplicar sus reglas de pedidos, pagos, envíos, devoluciones y stock. Usará las piezas compartidas para las solicitudes y el guardado.

## Dónde se guarda cada cosa

Ampliaremos los dos modelos actuales de configuración y credenciales. La configuración irá junta en JSON: dirección, acciones, mapeos, sistema principal por campo y avance. Las credenciales se guardarán cifradas aparte.

El ID de Shopify irá en el campo elegido del propio producto, pedido u otro registro. No crearemos una tabla de enlaces obligatoria.

Una tabla adicional recordará qué operaciones o movimientos ya se aplicaron. Esta información se consulta durante el procesamiento para evitar repetirlos. El historial reutilizará las ejecuciones guardadas.

## Cómo se conecta todo

1. **Conectar y configurar.** La autorización crea la conexión para la empresa actual. El usuario elige modelos, campos admitidos, ubicaciones y qué sistema manda. Cada solicitud comprueba que la conexión siga autorizada.

2. **Importar primero.** Shopify aporta los datos iniciales, incluso para campos que después controlará Five Panel. Cada lote ejecuta **leer → guardar → encolar**. El siguiente recibe la importación, dónde continuar y el tamaño/número de lote. Por ejemplo, 1.248 productos en grupos de 250 generan cinco ejecuciones bajo una sola importación.

3. **Guardar y continuar.** Los registros y su avance se guardan juntos; las llamadas a Shopify van aparte. Si falla guardar el lote 2, un reintento seguro conserva su ejecución y la lectura anterior. Si falla encolar, se reintenta ese paso sin guardar otra vez. El visor agrupa pasos por lote; cada paso tiene datos, errores e intentos.

4. **Revisar y activar.** Incorporamos los cambios llegados durante la importación y mostramos datos, reglas, programación y acciones. Los pedidos históricos no repiten descuentos de stock ni pagos. La sincronización saliente comienza con la activación expresa y el stock inicial confirmado.

5. **Sincronizar.** Los avisos de Shopify, cambios locales y revisiones programadas inician automatizaciones. Un precio de 25 controlado por Five Panel se restaura a 25 si Shopify lo cambia a 15. Para stock, procesamos la venta una vez antes de corregir cantidades: ambos pasan de 10 a 9, incluso con avisos repetidos. Si Shopify controla la cantidad, la importamos sin descontar otra vez el pedido.

6. **Ejecutar acciones.** Pedidos, cobros, envíos, devoluciones y reembolsos usan opciones revisadas por el usuario. Guardamos sus resultados y comprobamos los inciertos antes de permitir otro intento. Pausar detiene la sincronización automática saliente; desconectar bloquea nuevas solicitudes.

## En qué orden lo construiremos

Primero, conexiones y acciones compartidas; después, autorización e importación. Luego, sincronización y acciones de negocio. Las pantallas e historial se conectarán a esos servicios siguiendo el [diseño aprobado](279-shopify-designs/README.md).

Comprobaremos que la importación continúe tras un fallo, que una venta descuente stock una vez y que un pago incierto no se repita. El plan técnico conserva los archivos, tareas y pruebas concretas.
