# Fase 1: Microcks — Mocking Dinámico de APIs en Kubernetes

## 1.1 ¿Qué es Microcks?

Microcks es una plataforma **open-source** para el mocking y testing de APIs. Su propuesta de valor central es que lee un **contrato de API** (OpenAPI, AsyncAPI, Postman Collection, etc.) y genera automáticamente un servidor de mocks completamente funcional, sin escribir código adicional.

### Conceptos clave:

| Concepto | Descripción |
|---|---|
| **Dynamic Mocking** | El mock genera respuestas variables usando scripts o templates (no solo valores estáticos) |
| **Service Map** | Vista de Microcks que muestra todos los servicios importados y sus versiones |
| **Test Runner** | Motor integrado en Microcks para ejecutar pruebas de conformidad contra APIs reales |

### Referencias esenciales

- 📖 [Documentación oficial de Microcks](https://microcks.io/documentation/)
- 📖 [Mocking con OpenAPI](https://microcks.io/documentation/using/openapi/)
- 📖 [Helm Chart de Microcks](https://microcks.io/documentation/installing/kubernetes/)
- 📖 [Ejemplos de contratos para mocking](https://github.com/microcks/microcks/tree/main/samples)

---

## 1.2 Instalación de Microcks

### Paso 1: Agregar el repositorio Helm de Microcks

```bash
# Agregar el repositorio de Helm de Microcks
helm repo add microcks https://microcks.io/helm
helm repo update

# Verificar que el chart está disponible
helm search repo microcks
```

### Paso 2: Instalar Microcks en el clúster

```bash
# Instalar Microcks usando los valores del archivo de configuración
helm install microcks microcks/microcks \
  --namespace microcks \
  --values 01-microcks/microcks-values.yaml

# Verificar que los pods estén corriendo
kubectl get pods -n microcks --watch
```

**Espera hasta que TODOS los pods estén `Running`**

```bash
# Verificar los servicios desplegados
kubectl get services -n microcks
```

Deberías ver servicios como:
```
NAME                 TYPE        CLUSTER-IP     PORT(S)
microcks             ClusterIP   10.96.x.x      8080/TCP
microcks-grpc        ClusterIP   10.96.x.x      9090/TCP
microcks-mongodb     ClusterIP   10.96.x.x      27017/TCP
```

### Paso 3: Acceder a la UI de Microcks

```bash
# En una terminal separada (déjala corriendo durante toda la PoC)
kubectl port-forward svc/microcks 9090:8080 -n microcks
```

> Deja esta terminal abierta. Puedes abrir una nueva terminal para los siguientes pasos.

Abre tu navegador en: **http://localhost:9090**

---

## 1.3 Importar un Contrato OpenAPI

Microcks genera mocks a partir de contratos que incluyan **`examples`** en las respuestas.

> 📁 El archivo ya existe en el repo: `01-microcks/petstore-with-examples.yaml`

### Paso 4: Importar la Petstore API con ejemplos

En Microcks podemos importar un archivo local:

1. En la UI de Microcks, haz clic en **+ Quick Import** (esquina superior derecha)
2. Selecciona el archivo `01-microcks/petstore-with-examples.yaml` desde tu sistema de archivos
3. Haz clic en **Upload**

### Paso 5: Explorar el Mock generado

Una vez importado verás el servicio **Petstore 1.0.0** en **APIs | Services**:

1. Ve al menú lateral → **APIs | Services** → clic en **Petstore**
2. En la sección **OPERATIONS** verás `GET /pet/findByStatus` (3 samples) y `GET /pet/{petId}` (2 samples)
   (cada sample corresponde a un ejemplo del contrato)
3. Expande una operación haciendo clic en el `›` — en la sección **MOCKS** verás la URL:
   ```
   http://localhost:9090/rest/Petstore/1.0.0/pet/findByStatus?status=available
   ```

> 💡 El patrón de URLs de Microcks es: `http://localhost:9090/rest/{title}/{version}/{path}`

### Paso 6: Consumir el mock desde la terminal

```bash
# Mascotas disponibles → retorna el ejemplo "available"
curl -s "http://localhost:9090/rest/Petstore/1.0.0/pet/findByStatus?status=available" 

# Mascotas en adopción → retorna el ejemplo "pending"
curl -s "http://localhost:9090/rest/Petstore/1.0.0/pet/findByStatus?status=pending" 

# Mascota por ID (retorna "pet_1" o "pet_2" según el ID)
curl -s "http://localhost:9090/rest/Petstore/1.0.0/pet/1" 
```

---

## 1.4 Mocking Dinámico con Dispatcher Scripts

Microcks permite que los mocks sean **dinámicos**: la respuesta varía según los parámetros de entrada.

### ¿Cómo funciona?

Microcks usa **Dispatcher Scripts** (Groovy) para seleccionar qué ejemplo retornar basado en la solicitud entrante:

```groovy
// Ejemplo: retorna una respuesta diferente según el ID solicitado
if (mockRequest.getParameter("id") == "1") {
  return "available_pet"
} else {
  return "not_found"
}
```

### Referencias

- 📖 [Dispatcher Scripts — Documentación](https://microcks.io/documentation/using/advanced/dispatching/)

---

## 1.5 Actividades TODO

### TODO 1: Análisis del Mocking Dinámico

**Objetivo:** Entender cómo Microcks gestiona contratos antes de configurarlo con Backstage.

Responde en el archivo `01-microcks/analisis-mocking.md`:

1. ¿Qué **operaciones** expone la API Petstore importada? Lista cada una con su método HTTP y path.
2. ¿Cuál es la **URL base** generada por Microcks para hacer mocking de esta API?
3. Ejecuta al menos **2 llamadas curl** al mock usando los comandos del Paso 6. Pega las URLs y respuestas obtenidas.
4. ¿Qué ventaja ofrece Microcks frente a un mock estático (ej: un archivo JSON servido con `http-server`)?

---

### TODO 2: Importar tu propio contrato OpenAPI

**Objetivo:** Crear y publicar un contrato OpenAPI básico en Microcks.

Crea un archivo `01-microcks/mi-api.yaml` con un contrato OpenAPI 3.0 que describa una API sencilla (al menos 2 endpoints).

Luego impórtalo en Microcks y verifica que los mocks funcionan.

---

*Continúa con → [Fase 2: Backstage](../02-backstage/README.md)*
