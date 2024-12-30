# Migration scripts dentro de procesos de CI/CD

La presente documentación consiste en describir una serie de pasos requeridos y recomendaciones para implementar lógica que permita correr los migration scripts de Contentful dentro de procesos de Continuous Integration.

## 1. Inicialización del entorno:

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

## 2. Obtención de archivos de migración:

Una vez configurado el entorno, el siguiente paso es identificar y obtener los archivos que contienen las migraciones. En este paso se recopilan todos los scripts o archivos que contienen las especificaciones de la(s) migracion(es).

### 2.1 Identificar el directorio de migraciones.

El proceso de migración implica la ejecución de varios scripts de migración. Estos archivos generalmente están almacenados en una carpeta específica (por ejemplo, `/migrations`) dentro del codebase. El proceso consiste en localizar estos archivos y organizarlos de manera que se ejecuten en el orden correcto.

Los archivos de migración deberían tener una estructura clara y estar nombrados de manera que facilite su orden de ejecución. Un enfoque común es usar una marca temporal, como por ejemplo un timestamp (`2024-12-30-01-create-content-type.js`).

```js
console.log("Read all the available migrations from the file system");
const availableMigrations = (await readdirAsync(MIGRATIONS_DIR))
    .filter((file) => /^\d+?\.js$/.test(file))
    .map((file) => getVersionOfFile(file));
```

### 2.2 Verificar scripts de migraciones.

Consultar las entries  del content type `versionTracking` en Contentful para obtener el último valor de la migración que se ejecutó con éxito. Este valor será utilizado para comparar con los archivos de migración disponibles.

```js
console.log("Figure out latest ran migration of the contentful space");
const { items: versions } = await environment.getEntries({
    content_type: "versionTracking",
});

let storedVersionEntry = versions[0];

const currentVersionString = storedVersionEntry.fields.version[defaultLocale];
```

Una vez que se cuente con la versión de migración más reciente de Contentful, se deberá comparar esta versión con los archivos de migración disponibles en el directorio de migraciones.

```js
console.log("Evaluate which migrations to run");
const currentMigrationIndex = availableMigrations.indexOf(currentVersionString);

if (currentMigrationIndex === -1) {
    throw new Error(
        `Version ${currentVersionString} is not matching with any known migration`
    );
}
const migrationsToRun = availableMigrations.slice(currentMigrationIndex + 1);
```

## 3. Ejecución de los scripts y actualización del versionTracking

Después de que se ejecuta una migración, es crucial actualizar el `versionTracking` en Contentful para registrar que la migración ha sido ejecutada correctamente. Esto ayudará a mantener un historial de migraciones y asegurará que las siguientes ejecuciones solo procesen las migraciones pendientes.

### 3.1 Ejecutar scripts de migraciones.

Para cada migración en la lista de migraciones pendientes por ejecutar (`migrationsToRun` del snippet anterior), se deberá correr el script correspondiente en el ambiente de Contentful utilizando el comando adecuado.

```bash
# Comando usando migration del CLI de Contentful
contentful space migration --yes --management-token <TOKEN> --environment-id <ENVIRONMENT> ./migrations/<FILE>
```

```js
console.log("Run migrations and update version entry");

// Definir las opciones de migración que se usarán para ejecutar cada migración.
const migrationOptions = {
    spaceId: SPACE_ID,
    environmentId: ENVIRONMENT_ID,
    accessToken: CMA_ACCESS_TOKEN,
    yes: true,
};

// Iniciar el proceso de ejecución de migraciones, iterando sobre cada migración pendiente en la lista `migrationsToRun`
while ((migrationToRun = migrationsToRun.shift())) {
    const filePath = path.join(
        __dirname,
        "..",
        "migrations",
        getFileOfVersion(migrationToRun)
    );
    
    console.log(`Running ${filePath}`);

    // Ejecutar la migración con las opciones proporcionadas
    await runMigration(
        Object.assign(migrationOptions, {
            filePath,
        })
    );
    console.log(`${migrationToRun} succeeded`);

    // Actualizar el entry de `versionTracking` en Contentful para reflejar que esta migración ha sido ejecutada.
    storedVersionEntry.fields.version[defaultLocale] = migrationToRun;
    storedVersionEntry = await storedVersionEntry.update();
    storedVersionEntry = await storedVersionEntry.publish();

    console.log(`Updated version entry to ${migrationToRun}`);
}
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

#### Inicio del proceso de migración:
Registrar un mensaje indicando que se están iniciando las migraciones, junto con detalles de configuración como el space y environment objetivos o el número total de archivos de scripts de migración a ejecutar.

#### Antes de ejecutar cada script:
Registrar el nombre del archivo y el comando que se va a ejecutar.

#### Resultados de la ejecución:
Registrar si un script fue exitoso o falló, junto con detalles del error en caso de fallar.

## Recursos adicionales

* [Documentación de contentful-migration](https://github.com/contentful/contentful-migration/blob/main/README.md#reference-documentation)

* [Documentación de migration de Contentful CLI](https://github.com/contentful/contentful-cli/blob/main/docs/space/migration/README.md)