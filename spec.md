#1. DEFINICIÓN DE LA APP

Crea una app en local en un solo archivo index.html que se conecte con dos apis de Open-Meteo para que, al pedirle al usuario un lugar, las apis devuelvan las buenas horas para hacer alguna actividad.

#2. COMO FUNCIONAN LAS APIS

Este apartado documenta técnicamente el funcionamiento, las URLs, los parámetros requeridos y la estructura de respuesta de las dos APIs de Open-Meteo que deben integrarse. Ambas APIs son públicas, gratuitas para uso no comercial, no requieren autenticación mediante API Key y responden exclusivamente en formato JSON mediante peticiones HTTP `GET`.

---

## 1. API de Geocoding (Conversión de Ciudad a Coordenadas)

* **Servicio**: Open-Meteo Geocoding API
* **Base URL**: `https://geocoding-api.open-meteo.com/v1/search`
* **Método HTTP**: `GET`
* **Propósito**: Transformar una cadena de texto (nombre de una ciudad) en coordenadas geográficas numéricas ($\text{latitud}$ y $\text{longitud}$) indispensables para consultar la previsión meteorológica.

### Parámetros de Consulta (Query Parameters)
Para construir la URL dinámica, se deben sustituir o ajustar los siguientes parámetros tras el carácter `?`:

| Parámetro | Tipo | Dinámico | Descripción / Valor recomendado |
| :--- | :--- | :--- | :--- |
| `name` | `String` | **SÍ** | El nombre de la ciudad a buscar (debe aplicarse codificación de URL / `encodeURIComponent`). Ejemplo: `Sevilla` |
| `count` | `Integer` | No | Número de resultados deseados. Usar `1` para obtener únicamente la coincidencia más relevante. |
| `language` | `String` | No | Idioma de los resultados. Usar `es` para español. |

### Ejemplo de Petición HTTP
```text
GET https://geocoding-api.open-meteo.com/v1/search?name=Sevilla&count=1&language=es
```

### Estructura de la Respuesta JSON y Extracción de Datos
La API devuelve un objeto principal. Si se encuentra la ubicación, contendrá un array denominado `results`:

```json
{
  "results": [
    {
      "id": 2510911,
      "name": "Sevilla",
      "latitude": 37.38283,
      "longitude": -5.97317,
      "elevation": 18.0,
      "feature_code": "PPLA",
      "country_code": "ES",
      "timezone": "Europe/Madrid",
      "country": "España"
    }
  ],
  "generationtime_ms": 0.45
}
```

#### Variables a extraer en código:
* **Latitud**: `results[0].latitude` (ej. `37.38283`)
* **Longitud**: `results[0].longitude` (ej. `-5.97317`)
* **Huso Horario** (opcional pero recomendado): `results[0].timezone` (ej. `"Europe/Madrid"`)

#### Casos Límite y Manejo de Errores:
* Si la búsqueda no devuelve coincidencias, la clave `results` no existirá en el JSON o será un array vacío (`results: []`).
* El sistema debe validar la presencia de `results[0]` antes de intentar acceder a `latitude` o `longitude`.

---

## 2. API de Previsión Meteorológica (Forecast API)

* **Servicio**: Open-Meteo Weather Forecast API
* **Base URL**: `https://api.open-meteo.com/v1/forecast`
* **Método HTTP**: `GET`
* **Propósito**: Obtener la previsión meteorológica hora a hora para una posición geográfica determinada durante la jornada actual.

### Parámetros de Consulta (Query Parameters)
Los parámetros de esta URL deben configurarse dinámicamente injecting los datos obtenidos en la llamada previa de Geocoding:

| Parámetro | Tipo | Dinámico | Descripción / Valor |
| :--- | :--- | :--- | :--- |
| `latitude` | `Float` | **SÍ** | Coordenada de latitud obtenida en la API 1 (ej. `37.38283`). |
| `longitude` | `Float` | **SÍ** | Coordenada de longitud obtenida en la API 1 (ej. `-5.97317`). |
| `hourly` | `String` | No | Lista de variables requeridas separadas por coma. Debe incluir obligatoriamente: `temperature_2m,precipitation_probability`. |
| `forecast_days` | `Integer` | No | Número de días a consultar. Usar `1` para limitar la previsión al día actual. |
| `timezone` | `String` | **SÍ** | Zona horaria para alinear las horas locales. Usar `auto` o el parámetro `timezone` devuelto por la API de Geocoding (ej. `Europe/Madrid`). |

### Ejemplo de Petición HTTP
```text
GET https://api.open-meteo.com/v1/forecast?latitude=37.38283&longitude=-5.97317&hourly=temperature_2m,precipitation_probability&forecast_days=1&timezone=Europe/Madrid
```

### Estructura de la Respuesta JSON y Correlación de Datos
La API devuelve un objeto que contiene la propiedad `hourly`, la cual encapsula tres listas paralelas con el mismo índice temporal:

```json
{
  "latitude": 37.38,
  "longitude": -5.97,
  "generationtime_ms": 0.12,
  "utc_offset_seconds": 7200,
  "timezone": "Europe/Madrid",
  "timezone_abbreviation": "CEST",
  "elevation": 18.0,
  "hourly_units": {
    "time": "iso8601",
    "temperature_2m": "°C",
    "precipitation_probability": "%"
  },
  "hourly": {
    "time": [
      "2026-10-21T00:00",
      "2026-10-21T01:00",
      "2026-10-21T02:00"
    ],
    "temperature_2m": [
      18.4,
      17.9,
      17.2
    ],
    "precipitation_probability": [
      0,
      5,
      15
    ]
  }
}
```

#### Regla de Mapeo e Interferencia de Datos:
Las tres listas dentro del objeto `hourly` se corresponden de manera síncrona según su posición de índice $i$:
* `hourly.time[i]`: Marca temporal en formato ISO 8601 (ej. `"2026-10-21T14:00"`).
* `hourly.temperature_2m[i]`: Temperatura estimada en grados Celsius ($^\circ\text{C}$) a la hora $i$.
* `hourly.precipitation_probability[i]`: Probabilidad de precipitación expresada en porcentaje ($\%$, de $0$ a $100$) a la hora $i$.

---

## 3. Secuencia de Integración requerida

```
[Parámetro entrada: Búsqueda]
              │
              ▼
    1. GET /v1/search?name={ciudad}
              │
              ├───► [Si result está vacío] ───► Notificar que la ciudad no existe
              │
              ▼ [Obtener latitude y longitude]
    2. GET /v1/forecast?latitude={lat}&longitude={lon}...
              │
              ▼
[Listas paralelas: time, temperature_2m, precipitation_probability]
```

#3. COMO DEBE FUNCIONAR LA APP

Primero le pediremos al usuario un municipio específico.
Segundo, mandaremos el nombre de ese municipio a la api de geocoding de open-meto, para saber su latitud y longitud.
Tercero, mandaremos la latitud y longitud a la api de previsión meteorológica de open-meteo para tener una lista con el clima del dia actual por horas futuras, obviaremos las horas que ya han pasado.
Cuarto, entre esa lista se elige las horas con mejores climas con estos requerimientos:

### Consideraremos que una hora es "buena hora" cuando:
1. La temperatura esté entre 18ºC y 27ºC, ambos incluidos.
2. Haya un probabilidad de lluvia por debajo del 10%, estando este excluido.
3. La hora no coincida con las siguientes: Entre las 23:00 (excluido) y las 6:00 (incluido); entre las 14:00 y las 15:30 (ambos excluidos).

# 4. QUE HACER EN CASO DE ERRORES

Es posible que el usuario añada una ciudad inexistente, en ese caso la api nos devolverá un error. Si esto sucede, debemos hacerselo saber al usuario para que introduzca un municipio válido. Para ello debe aparecer un mensaje de aviso que ponga:
"El municipio introducido no es válido, por favor introduzca otro."
Posteriormente, permita al usuario escribir un municipio válido.

# 5. FRONTEND

Debe ser una aplicación minimalista que, inicialmente contenga un buscador de municipios. Cuando encuentre el municipio y tengamos las "buenas horas", aparezca una lista con un titular del municipio y país del mismo, con estas buenas horas.
Colores: azul cielo y blanco nube.
Sin emoticonos.
Estilo app de climatología moderna.
