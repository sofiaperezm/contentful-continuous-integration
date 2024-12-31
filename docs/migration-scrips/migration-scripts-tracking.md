# Tracking de scripts de migración

Este documento detalla dos enfoques para automatizar la ejecución de scripts de migración en Contentful cada vez que se suba un cambio al modelo de contenido.

## Enfoque 1: Uso de versionTracking

### Requerimientos

* **Content Type versionTracking:** Un tipo de contenido en Contentful que almacena las versiones de migración ejecutadas.

* **Scripts de migración:** Archivos predefinidos que representan cambios al modelo de contenido.

* **CLI de Contentful:** Herramienta de línea de comandos para ejecutar migraciones.

### Flujo del Proceso

1. Verificar scripts de migración aplicados

Consultar las entradas del content type `versionTracking` para obtener la última versión ejecutada con éxito:

```js
console.log("Figure out latest ran migration of the contentful space");
const { items: versions } = await environment.getEntries({
    content_type: "versionTracking",
});

let storedVersionEntry = versions[0];

const currentVersionString = storedVersionEntry.fields.version[defaultLocale];
```

2. Determinar scripts de migración pendientes

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

3. Ejecutar scripts de migración y actualizar versionTracking

Para cada migración en la lista de migraciones pendientes por ejecutar (`migrationsToRun` del snippet anterior), se deberá correr el script correspondiente en el ambiente de Contentful utilizando el comando adecuado y las opciones requeridas.

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

## Enfoque 2: Uso de Contentful Merge App

### Requerimientos

* **Contentful Merge App:** Herramienta que genera un archivo de diferencias entre ambientes.

* **CLI de Contentful:** Para ejecutar los scripts generados.

### Flujo del Proceso

1. Generar el script de migración

Utilizar la CLI para exportar las diferencias entre el ambiente fuente y el objetivo.

```bash
contentful merge export --yes
    --output-file ${folder}/${filename}
    --te ${context.targetEnvironmentId}
    --se ${context.sourceEnvironmentId}
    --management-token ${context.accessToken}
```

2. Aplicar el script generado

Ejecutar el script exportado en el ambiente objetivo.

```bash
contentful space migration --yes
    --management-token ${context.accessToken}
    --environment-id ${context.targetEnvironmentId}
    ${folder}/${file}
```

## Análisis comparativo

Ambos enfoques tienen sus ventajas y desventajas, a continuación se muestra una análisis comparativo teniendo en cuenta aspectos como escalabilidad y complejidad operativa.

| Aspecto                     | versionTracking                       | Merge APP                        |
| ---                         |                ----                   |                               ---|
| Requerimientos              | Configuración manual de migraciones   | Instalación de Merge App         |
| Automatización              | Semi-automatizada                     | Totalmente automatizada          |
| Escalabilidad               | Alta, pero requiere más mantenimiento | Alta, con menor esfuerzo         |
| Complejidad Operativa       | Alta                                  | Baja                             |
| Control y Auditar Cambios   | Completo                              | Limitado a diferencias generadas |

La elección de un enfoque sobre otro dependerá si el proyecto requiere un control granular de migraciones y se cuenta con recursos para manejar la complejidad, el enfoque de `versionTracking` es adecuado. Sin embargo, para simplificar y acelerar la sincronización entre ambientes, se recomienda adoptar Contentful Merge App.