#1. DEFINICIÓN DE LA APP

Crea una app en local en un solo archivo index.html que se conecte con dos apis de Open-Meteo para que, al pedirle al usuario un lugar y una actividad, las apis devuelvan las buenas horas para hacer alguna actividad.

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
* **Huso Horario** (necesario para seleccionar el día local y mostrar las horas): `results[0].timezone` (ej. `"Europe/Madrid"`). Si falta o no es válido, mostrar un error de zona horaria, sin utilizar silenciosamente la del navegador.

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
La consulta debe respetar la URL prioritaria de `buenas_horas_segun_actividad.md`. Solo las coordenadas varían con los datos obtenidos en Geocoding:

| Parámetro | Tipo | Dinámico | Descripción / Valor |
| :--- | :--- | :--- | :--- |
| `latitude` | `Float` | **SÍ** | Coordenada de latitud obtenida en la API 1 (ej. `37.38283`). |
| `longitude` | `Float` | **SÍ** | Coordenada de longitud obtenida en la API 1 (ej. `-5.97317`). |
| `hourly` | `String` | No | Lista exacta: `temperature_2m,precipitation_probability,precipitation,relative_humidity_2m,apparent_temperature,wind_speed_10m,wind_direction_10m,snow_depth`. |

No añadir `forecast_days`, `timezone` ni otros parámetros. La respuesta utiliza por defecto GMT/UTC y abarca siete días. La aplicación debe interpretar cada `hourly.time[i]` como UTC (añadiendo `Z` al convertirlo a un instante), convertirlo a la zona IANA de Geocoding y conservar solo los instantes estrictamente posteriores al momento actual cuya fecha local coincida con la de hoy en ese municipio. La hora actual ya iniciada queda excluida. El filtrado se realiza en la aplicación, no en la URL.

Si la respuesta no cubre alguna hora del día local, no inventar datos ni mostrar horas de otro día para completarlo.

Referencia de los valores predeterminados: [documentación de Open-Meteo](https://open-meteo.com/en/docs).

### Ejemplo de Petición HTTP
```text
GET https://api.open-meteo.com/v1/forecast?latitude=37.38283&longitude=-5.97317&hourly=temperature_2m,precipitation_probability,precipitation,relative_humidity_2m,apparent_temperature,wind_speed_10m,wind_direction_10m,snow_depth
```

### Estructura de la Respuesta JSON y Correlación de Datos
La API devuelve un objeto con `hourly.time` y ocho listas meteorológicas paralelas con el mismo índice temporal. Ejemplo ilustrativo de la estructura:

```json
{
  "latitude": 37.38,
  "longitude": -5.97,
  "generationtime_ms": 0.12,
  "utc_offset_seconds": 0,
  "timezone": "GMT",
  "timezone_abbreviation": "GMT",
  "elevation": 18.0,
  "hourly_units": {
    "time": "iso8601",
    "temperature_2m": "°C",
    "precipitation_probability": "%",
    "precipitation": "mm",
    "relative_humidity_2m": "%",
    "apparent_temperature": "°C",
    "wind_speed_10m": "km/h",
    "wind_direction_10m": "°",
    "snow_depth": "m"
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
    ],
    "precipitation": [0, 0, 0.1],
    "relative_humidity_2m": [60, 65, 70],
    "apparent_temperature": [18, 17, 16],
    "wind_speed_10m": [10, 12, 15],
    "wind_direction_10m": [180, 190, 200],
    "snow_depth": [0, 0, 0]
  }
}
```

#### Regla de Mapeo e Interferencia de Datos:
Las listas dentro del objeto `hourly` se corresponden de manera síncrona según su posición de índice $i$:
* `hourly.time[i]`: Marca temporal en UTC con esta consulta (ej. `"2026-10-21T14:00"`, que debe interpretarse como `"2026-10-21T14:00Z"`).
* `hourly.temperature_2m[i]`: Temperatura estimada en grados Celsius ($^\circ\text{C}$) a la hora $i$.
* `hourly.precipitation_probability[i]`: Probabilidad de precipitación expresada en porcentaje ($\%$, de $0$ a $100$) a la hora $i$.
* Las otras seis variables conservan ese mismo índice; sus unidades y criterios se describen en `buenas_horas_segun_actividad.md`.

---

## 3. Secuencia de Integración requerida

```
[Parámetro entrada: Búsqueda]
              │
              ▼
    1. GET /v1/search?name={ciudad}
              │
              ├───► [Si results está ausente o vacío] ───► Pedir un municipio válido
              │
              ▼ [Obtener latitude y longitude]
    2. GET /v1/forecast?latitude={lat}&longitude={lon}...
              │
              ▼
[Listas paralelas: time y las ocho variables meteorológicas]
              │
              ▼
[Convertir UTC a hora local y seleccionar las horas futuras de hoy]
```

# 3. COMO DEBE FUNCIONAR LA APP

Primero le pediremos al usuario un municipio específico y una actividad específica entre las que hay en el documento "buenas_horas_segun_actividad.md".
Segundo, mandaremos el nombre de ese municipio a la api de geocoding de open-meto, para saber su latitud y longitud.
Tercero, mandaremos la latitud y longitud a la api de previsión meteorológica de open-meteo para tener una lista con el clima del dia actual por horas futuras, obviaremos las horas que ya han pasado.
Cuarto, se evalúan todas las horas futuras disponibles de hoy según la actividad seleccionada y se destacan las que cumplen estos requerimientos, sin ocultar las demás:

### Consideraremos que una hora es "buena hora" cuando:
Leer archivo "buenas_horas_segun_actividad.md".

# 4. QUE HACER EN CASO DE ERRORES

Si Geocoding responde correctamente pero `results` está ausente o vacío, mostrar el aviso:
"El municipio introducido no es válido, por favor introduzca otro."
Posteriormente, permita al usuario escribir un municipio válido.

Los errores HTTP, de conexión, de tiempo de espera o de respuesta inválida deben tratarse por separado, con un mensaje del problema del servicio y la posibilidad de reintentar. No deben presentarse como un municipio inexistente.

# 5. FRONTEND

Debe ser una aplicación minimalista que inicialmente contenga un buscador de municipios y un filtro de actividad según `buenas_horas_segun_actividad.md`. Tras la búsqueda, aparecerá una lista de todas las horas futuras disponibles de hoy, con un titular que muestre municipio, región y país. Las buenas horas tendrán fondo blanco y borde azul brillante; las demás, fondo gris y borde discontinuo. Si la cobertura es incompleta, se mostrarán las horas disponibles y un aviso, sin inventar datos para las horas ausentes. El usuario verá los datos meteorológicos, pero no podrá configurar los umbrales técnicos de evaluación.
Colores: azul cielo y blanco nube.
Sin emoticonos.
Estilo app de climatología moderna.
