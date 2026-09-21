# Fase 2: Backstage — Portal del Desarrollador

## 2.1 ¿Qué es Backstage?

### Conceptos clave que debes dominar:

| Concepto | Descripción |
|---|---|
| **Plugin** | Extensión que agrega funcionalidad a Backstage (ej: integración con Microcks) |
| **TechDocs** | Sistema de documentación-as-code integrado en Backstage |
| **Entity** | Cualquier elemento registrado en el catálogo (Component, API, System, User, etc.) |

### Tipos de entidades del catálogo

| Kind | Propósito |
|---|---|
| `Component` | Un servicio, librería o sitio web |
| `API` | Una interfaz de programación expuesta por un Component |
| `System` | Agrupación lógica de Components relacionados |
| `Resource` | Infraestructura que un Component consume (DB, cola, etc.) |

### Referencias esenciales

- 📖 [Documentación oficial de Backstage](https://backstage.io/docs)
- 📖 [Software Catalog — Conceptos](https://backstage.io/docs/features/software-catalog/)
- 📖 [Descriptor Format (catalog-info.yaml)](https://backstage.io/docs/features/software-catalog/descriptor-format)
- 📖 [Plugin Microcks para Backstage](https://github.com/microcks/microcks-backstage-provider)

---

## 2.2 Crear la aplicación Backstage

### Paso 1: Crear la app con el CLI

```bash
# Crear una nueva app de Backstage (en el directorio del repo)
npx @backstage/create-app@latest --skip-install

# Cuando pregunte el nombre, escribe: api-platform-portal
```

> Esto crea una carpeta `api-platform-portal/` con la estructura completa de Backstage.

### Paso 2: Instalar dependencias

```bash
# Entrar al directorio de la app
cd api-platform-portal

# Instalar dependencias
yarn install
```

### Paso 3: Ejecutar Backstage localmente

```bash
# Iniciar en modo desarrollo (desde la carpeta api-platform-portal)
yarn start
```

Backstage estará disponible en: **http://localhost:3000**

> La primera vez puede tardar 1-2 minutos en compilar.

---

## 2.3 Configurar el plugin de Microcks

El proveedor de Microcks en Backstage sincroniza automáticamente las APIs y sus mocks en el catálogo.

### Paso 4: Instalar y registrar el provider en el Backend

```bash
# Desde la carpeta api-platform-portal, instalar en packages/backend
yarn --cwd packages/backend add @microcks/microcks-backstage-provider
# Agregar dependencias requeridas
yarn --cwd packages/backend add @backstage/backend-plugin-api @backstage/plugin-catalog-node

```

Registra el módulo en `api-platform-portal/packages/backend/src/index.ts` antes de hacer `backend.start()`:

```typescript
import { createBackendModule, coreServices } from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { MicrocksApiEntityProvider } from '@microcks/microcks-backstage-provider';

// Microcks catalog entity provider
const catalogModuleMicrocks = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'microcks-provider',
  register(reg) {
    reg.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
        config: coreServices.rootConfig,
        logger: coreServices.logger,
        scheduler: coreServices.scheduler,
      },
      async init({ catalog, config, logger, scheduler }) {
        catalog.addEntityProvider(
          MicrocksApiEntityProvider.fromConfig(config, {
            logger,
            scheduler,
          }),
        );
      },
    });
  },
});
backend.add(catalogModuleMicrocks);
```

### Paso 5: Configurar la conexión a Microcks

Revisa y completa el archivo `02-backstage/app-config.yaml`. Este archivo configura en la sección `catalog.providers.microcksApiEntity` la conexión a Microcks (que corre en el clúster Kind). 

Al incluir este **Entity Provider**, Backstage consulta periódicamente a Microcks mediante su API, descubriendo e importando **automáticamente** las definiciones de APIs (OpenAPI, AsyncAPI, etc.) y sus mocks directamente en el catálogo de Backstage, sin necesidad de registrarlas manualmente una a una:

```bash
# Copiar la configuración local al directorio de la app
cp 02-backstage/app-config.yaml api-platform-portal/app-config.yaml
```

Luego reinicia Backstage:

```bash
# Detener el servidor con Ctrl+C y volver a ejecutar
yarn start
```

---

## 2.4 Demo: Ciclo Contrato → Mock → Catálogo Automático

Al tener configurado el **Microcks Entity Provider**, ya no es necesario registrar manualmente archivos `catalog-info.yaml`. Las APIs subidas a Microcks son descubiertas e importadas automáticamente en Backstage.

### Paso 6: Verificar el descubrimiento automático en Backstage

1. Ingresa a **http://localhost:3000/catalog** en tu navegador.
2. Filtra por **Kind: API**.
3. Verifica que tu API aparece registrada automáticamente con sus metadatos, tags y enlace a los mocks.
4. Entra al detalle de la API y revisa la pestaña **Definition** para ver la documentación interactiva OpenAPI.

### Paso 7: Actualización en vivo del contrato (Ciclo Contrato → Catálogo)

1. Edita tu archivo `01-microcks/mi-api.yaml` para agregar o modificar un endpoint.
2. En la UI de Microcks (**http://localhost:9090**), ve a **Services** → tu API → **Edit** → **Force Import**.
3. Comprueba que el nuevo mock responde:
   ```bash
   curl -s http://localhost:9090/rest/OrdersAPI/1.0.0/orders/1/status 
   ```
4. En Backstage (**http://localhost:3000/catalog**), ve a tu API y observa cómo la pestaña **Definition** se actualiza automáticamente con el nuevo contrato (puedes forzar la actualización con el menú `⋮` → **Schedule Entity Refresh**).

---

## 2.5 Actividades TODO

### TODO 3: Analizar el Catálogo de APIs

**Objetivo:** Reflexionar sobre el valor del catálogo centralizado para la gestión de APIs.

Navega por la UI de Backstage y responde en el archivo `02-backstage/analisis-catalogo.md`:

1. ¿Qué información muestra Backstage sobre la API sincronizada desde Microcks?
2. ¿Cómo mejora el descubrimiento automático de APIs en un equipo de 50+ ingenieros en comparación con el registro manual?
3. ¿Qué diferencia existe entre un `Component` y una `API` en el modelo de datos de Backstage?

---

## 2.6 Limpieza del entorno

Una vez completado el laboratorio:

```bash
# 1. Detener Backstage (Ctrl+C en la terminal donde corre yarn start)

# 2. Eliminar Microcks del clúster
helm uninstall microcks -n microcks
kubectl delete namespace microcks

# 3. Eliminar el clúster Kind
kind delete cluster --name api-platform

# 4. Verificar que todo se eliminó
kind get clusters
# No debe aparecer 'api-platform'
```
