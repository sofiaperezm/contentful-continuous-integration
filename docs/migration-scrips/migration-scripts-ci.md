# Migration scripts dentro de procesos de CI/CD

La presente documentación consiste en describir una serie de pasos requeridos y recomendaciones para implementar lógica que permita correr los migration scripts de Contentful dentro de procesos de Continuous Integration.

## 1. Inicialización del entorno

Antes de ejecutar cualquier migración, se requiere configurar adecuadamente el entorno en el que se llevará a cabo. Este paso asegura que todos los recursos y parámetros necesarios estén disponibles y correctamente configurados.

### 1.1 Obtener configuración inicial

El proceso de migración generalmente requiere credenciales o configuraciones específicas como por ejemplo: `SPACE_ID`, `DELIVERY_API_KEY`, y `MANGEMENT_API_KEY`. Estos datos pueden provenir de diferentes fuentes, como variables de entorno o archivos de configuración. Es importante validar que todos estos parámetros estén correctamente establecidos.

### 1.2 Validar configuración inicial

Una vez que se ha obtenido la configuración, es esencial verificar que todos los parámetros requeridos estén presentes. Si falta alguno, se debe abortar el proceso y lanzar o loggear un mensaje de error descriptivo.

```js
if (!config.deliveryApiKey || !config.spaceId || !config.managementApiKey) {
  throw new Error('There are one or more credentials missing');
}
```

## 2. Obtención de archivos de migración

Una vez configurado el entorno, el siguiente paso es identificar y obtener los archivos que contienen las migraciones. En este paso se recopilan todos los scripts o archivos que contienen las especificaciones de la(s) migracion(es).

El proceso de migración implica la ejecución de varios scripts de migración. Estos archivos generalmente están almacenados en una carpeta específica (por ejemplo, `/migrations`) dentro del codebase. El proceso consiste en localizar estos archivos y organizarlos de manera que se ejecuten en el orden correcto.

Los archivos de migración deberían tener una estructura clara y estar nombrados de manera que facilite su orden de ejecución. Un enfoque común es usar una marca temporal, como por ejemplo un timestamp y el content type asociado(`2024-12-30-01-createContentType.js`) o los id de los ambientes fuente y objetivo(`2024-12-31-from-sourceEnvironmentId-to-targetEnvironmentId.js`).

```js
console.log("Read all the available migrations from the file system");
const availableMigrations = (await readdirAsync(MIGRATIONS_DIR))
    .filter((file) => /^\d+?\.js$/.test(file))
    .map((file) => getVersionOfFile(file));
```

Los archivos de migración podrían ser generados de una forma específica según la estrategia que se esté usando para determinar cuáles migration scripts deben ser ejecutados, esto depende si se está usando la enfoque basado en el content type como gestor de versiones versionTracking o en la Merge App de Contentful para identificar la diferencia entre ambientes. Más detalle en [esta documentación](./migration-scripts-tracking.md).

## 3. Ejecución de los scripts y actualización del versionTracking

Este paso es el corazón de este tipo de implementaciones, en el cual se deberá ejecutar los scripts de migración. Contando con la configuración y los archivos a ejecutar, se puede iniciar con la migración.

```js
const { runMigration } = require('contentful-migration')

const options = {
    filePath: '<path-to-migration-script>',
    spaceId: config.spaceId,
    accessToken: config.managementApiKey
}

runMigration(options)
    .then(() => console.log('Migration Done!'))
    .catch((e) => console.error(e))
```

## Validación y manejo de errores:

La siguiente serie de recomendaciones se deben tener en cuenta en todos los pasos necesarios para ejecutar los script de migración, no corresponden a un paso per se. Al ejecutar un script, es importante capturar cualquier error que ocurra, esto puede incluir fallos de red o errores en la lógica del script.

### Manejo de errores:

#### Try-cath
Utilizar bloques `try-catch` para detectar errores durante la ejecución y registrarlos para su posterior análisis.

#### Retries
Implementar un mecanismo para reintentar la ejecución de un script en caso de errores temporales. Esto es especialmente útil para problemas de red o fallos intermitentes de la API de Contentful, por ejemplo.

```js
const maxRetries = 3;
let attempts = 0;
let success = false;

while (attempts < maxRetries && !success) {
    try {
        await runMigration(options);
        success = true;
        console.log('Migration succeeded');
    } catch (e) {
        attempts++;
        console.error(`Attempt ${attempts} failed for ${file}:`, error);
    }
}
if (!success) {
  console.error(`All attempts failed for ${file}`);
}
```

#### Detener o continuar ante errores:
Decidir si el proceso debe detenerse al encontrar un error o continuar con los siguientes scripts. Esto puede configurarse según el nivel de criticidad de la migración.

### Inclusión de logs:

Los puntos clave para agregar logs en la lógica para ejecutar los script de migración podrían ser:

1. Inicio del proceso de migración:
Registrar un mensaje indicando que se están iniciando las migraciones, junto con detalles de configuración como el space y environment objetivos o el número total de archivos de scripts de migración a ejecutar.

2. Antes de ejecutar cada script:
Registrar el nombre del archivo y el comando que se va a ejecutar.

3. Resultados de la ejecución:
Registrar si un script fue exitoso o falló, junto con detalles del error en caso de fallar.

## Recursos para consultar

* [Documentación de contentful-migration](https://github.com/contentful/contentful-migration/blob/main/README.md#reference-documentation)

* [Documentación de migration de Contentful CLI](https://github.com/contentful/contentful-cli/blob/main/docs/space/migration/README.md)