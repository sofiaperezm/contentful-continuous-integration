# Configuración en procesos de CD/CI

La presenta documentación define una serie de pasos y tareas necesarios para implementar lógica de configuración de pipelines para herramientas de CI/CD, tales como Jenkins o circleci; este tipo de lógica generalmente se encuentra en archivos como `.circleci/config.yml` o `bitbucket-pipelines.yml`. Dentro de estos pasos se hace énfasis en la ejecución de los scripts de migración de Contentful.

## 1. Preparar el Entorno (Prepare the Environment)

### 1.1 Configurar el entorno de ejecución:

* Asegurarse que el entorno tenga la configuración adecuada para la aplicación (por ejemplo, instalar Python, Node.js, etc.).
* Definir las variables de entorno necesarias, como `NODE_ENV`, `DATABASE_URL`, o API tokens específicas (`MANAGEMENT_API_KEY`).
* Configurar herramientas adicionales necesarias para el proceso, como Docker, si es parte del pipeline.

### 1.2 Instalar dependencias necesarias:

* Emplear gestores de dependencias como `npm install` o `pip install` para asegurarse de que todas las librerías requeridas estén disponibles.
* Asegurarse de que las versiones de las dependencias sean compatibles con el proyecto actual.
* Garantizar acceso a servicios externos o APIs, lo que incluye verificar que las credenciales necesarias para servicios externos (como Contentful, bases de datos, etc.) estén configuradas y accesibles en el entorno.

## 2 Ejecutar Scripts de Migración

### 2.1 Correr migraciones pendientes

Para esto, se debe indicar el archivo que contiene la lógica para ejecutar los scripts de migración (por ejemplo `scripts/migrate.js`) y la configuración necesaria que podría incluir las credenciales requeridas por Contentful. 

```yml
- run:
    name: Running migration scripts
    command: |
        scripts/migrate.js $SPACE_ID master $MANAGEMENT_API_KEY
```

### 2.2 Actualizar el sistema de seguimiento de versiones

Una vez completada una migración, se deberá actualizar un sistema de control de versiones (puede incluir una tabla en una base de datos, un archivo global de estado o el mecanismo de `versionTracking`) para evitar que se vuelva a ejecutar accidentalmente.

## 3. Validar cambios post-migración

Comprobar que todas las modificaciones en el content model (como nuevos content types o fields modificados) están presentes en el environment de destino. Por ejemplo se podría agregar una tarea para correr una bateria de tests para evaluar cambios al content model.

```yml
- run:
    name: run content model tests
    command: |
        . venv/bin/activate
        pytest --environment-id="$CURRENT_ENVIRONMENT_ID"
```

Siguiendo con el ejemplo, se correría el siguiente test que evalúa que el content type de `Post` ahora cuenta con 5 fields (en vez de 4).

```py
def test_content_type_post(self, contentful_client):
    """Test content model of a post"""
    post_content_type = contentful_client.content_type("post")
    assert len(post_content_type.fields) == 5
```

## 4. Testing

Se requiere correr pruebas unitarias, de integración o end-to-end para verificar la funcionalidad específica de cada módulo o parte de la aplicación que haya sido afectada por las migraciones.

## 5. Almacenar artefactos

* Almacenar los logs generados durante la ejecución de migraciones y pruebas.
* Crear reportes resumidos que detallen el estado de cada paso del pipeline.

```yml
- store_artifacts:
    path: test-reports
    destination: test-reports
```

## 6. Despliegue

* Asegurarse de que las migraciones se implementen primero en entornos de `qa` o `staging` antes de producción.

* Confirmar que el despliegue cumpla con criterios de éxito predefinidos, como pruebas de regresión aprobadas.

```yml
- deploy:
    name: Deploy to Hosting Provider
    command: |
        if [ "$CURRENT_BRANCH" == "main" ] || [ "$CURRENT_BRANCH" == "staging" ] || [ "$CURRENT_BRANCH" == "qa" ]; then
            echo "Deploying branch $CURRENT_BRANCH to the hosting provider..."
            deploy_command --branch $CURRENT_BRANCH
        else
            echo "Branch $CURRENT_BRANCH is not configured for deployment."
        fi
```

## Recursos para consultar

* [Integrating migrations in your continuous delivery pipeline](https://www.contentful.com/developers/docs/concepts/deployment-pipeline/)

* [Create and deploy content type changes](https://www.contentful.com/developers/docs/tutorials/general/create-and-deploy-content-type-changes/)

* [Content model changes at scale – TELUS and CMS as Code](https://www.contentful.com/blog/content-model-changes-scale-telus-cms-as-code/)

* [Continuous Integration Tutorial](https://www.contentful.com/developers/docs/tutorials/general/continuous-integration-with-circleci/)