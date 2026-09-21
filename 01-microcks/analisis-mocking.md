# =====================================================================
# Análisis del Mocking Dinámico con Microcks
# =====================================================================

## Servicio analizado: Petstore 1.0.0 (`petstore-with-examples.yaml`)

### 1. Operaciones disponibles

| Método | Path | Descripción | Dispatcher | Samples |
|---|---|---|---|---|
| `GET` | `/pet/findByStatus?status={status}` | Busca mascotas por estado (`available`, `pending`, `sold`) | `URI_PARAMS` (regla: `status`) | 3 (`available`, `pending`, `sold`) |
| `GET` | `/pet/{petId}` | Busca una mascota por su ID | `SCRIPT` (Groovy) | 2 (`pet_1`, `pet_2`) |

- **`GET /pet/findByStatus`**: Microcks toma el valor del query param `status` y devuelve el ejemplo que se llama igual. Si el valor no tiene ejemplo (por ejemplo `?status=foo`), responde `400 The response ?status=foo does not exist!`.
- **`GET /pet/{petId}`**: un script Groovy lee el `petId` de la URL con `mockRequest.getURIParameters().get("petId")`. Si es `1` devuelve `pet_1`, si es `2` devuelve `pet_2` y con cualquier otro ID devuelve `pet_1` por defecto.

### 2. URL base del mock generado por Microcks

```
http://localhost:9090/rest/Petstore/1.0.0
```

Sigue el patrón `http://<host>/rest/{title}/{version}/{path}`: `Petstore` sale de `info.title` y `1.0.0` de `info.version`. El host es `localhost:9090` por el `port-forward` hacia el servicio `microcks` (puerto 8080) en el namespace `microcks`.

### 3. Llamadas curl al mock

#### Llamada 1: mascotas en adopción (`status=pending`)
```bash
curl -s "http://localhost:9090/rest/Petstore/1.0.0/pet/findByStatus?status=pending"
```

**Respuesta obtenida:**
```json
[
  {
    "id": 3,
    "name": "Rex",
    "status": "pending",
    "category": { "id": 1, "name": "Dogs" }
  }
]
```

#### Llamada 2: mascota por ID (`petId=2`)
```bash
curl -s "http://localhost:9090/rest/Petstore/1.0.0/pet/2"
```

**Respuesta obtenida:**
```json
{
  "id": 2,
  "name": "Misifus",
  "status": "available",
  "category": { "id": 2, "name": "Cats" },
  "photoUrls": ["https://example.com/misifus.jpg"],
  "tags": [{ "id": 2, "name": "indoor" }]
}
```

#### Llamadas adicionales (muestran el comportamiento dinámico)

| Llamada | Ejemplo devuelto | Respuesta |
|---|---|---|
| `/pet/findByStatus?status=sold` | `sold` | `[{"id":4,"name":"Luna","status":"sold","category":{"id":2,"name":"Cats"}}]` |
| `/pet/1` | `pet_1` | `{"id":1,"name":"Firulais","status":"available", ...}` |
| `/pet/3` | `pet_1` (valor por defecto del script) | `{"id":1,"name":"Firulais","status":"available", ...}` |

Con la misma operación y distintos parámetros se obtienen respuestas distintas: `pending` devuelve a Rex, `sold` a Luna, `/pet/2` a Misifus y `/pet/1` a Firulais.

### 4. Ventaja del mocking dinámico vs. mock estático

Un mock estático, como un archivo JSON servido con `http-server`, devuelve siempre la misma respuesta sin importar la petición. Microcks, en cambio, elige la respuesta según los datos de entrada: el dispatcher `URI_PARAMS` mira el query param `status` y el dispatcher `SCRIPT` evalúa el `petId` con lógica Groovy. Así se pueden simular varios escenarios (distintos estados, IDs existentes o no, errores `400`) con un solo endpoint, sin escribir código de servidor.

Además, el contrato OpenAPI es la única fuente de verdad: los mocks se generan a partir de sus `examples`. Si el contrato cambia, basta con reimportarlo para que el mock quede actualizado. Con un mock estático habría que editar a mano los archivos JSON, y es fácil que se desalineen de la especificación real.

Esto permite desarrollar en paralelo. El equipo de frontend o los consumidores de la API pueden empezar a integrar desde el día uno con respuestas realistas y variadas, sin esperar a que el backend esté implementado. Microcks también valida el contrato: por ejemplo, rechaza con `400` una petición sin el parámetro obligatorio `status`, así que detecta errores de integración temprano. Además, el mismo contrato sirve después para ejecutar pruebas de conformidad con su Test Runner contra la implementación real.
