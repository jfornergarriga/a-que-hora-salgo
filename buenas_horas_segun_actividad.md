# CRITERIOS Y REGLAS DE EVALUACIÓN: BUENAS HORAS SEGÚN ACTIVIDAD

## Introducción y Propósito

Este documento establece la especificación técnica de las reglas de negocio y el comportamiento de la interfaz para determinar y mostrar qué horas del día se clasifican como "buenas" según la actividad deportiva seleccionada.

El sistema evalúa la respuesta de la API de Open-Meteo aplicando umbrales técnicos ocultos al usuario. Además, este archivo resuelve las reglas de visualización de datos, manejo de errores y estados vacíos.

## 1. Configuración de la API y Búsqueda

### Consulta de Previsión Meteorológica
La aplicación realizará la petición utilizando estrictamente la siguiente URL, con los parámetros `latitude`, `longitude` y `hourly`. Solo las coordenadas variarán según Geocoding; la lista de ocho variables meteorológicas de `hourly` se mantendrá sin modificaciones:

`https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&hourly=temperature_2m,precipitation_probability,precipitation,relative_humidity_2m,apparent_temperature,wind_speed_10m,wind_direction_10m,snow_depth`

### Resolución de Ubicación (Geocoding)
* Dado que el parámetro `count=1` elige automáticamente la primera coincidencia, el frontend **debe mostrar claramente el Municipio, Región y País** obtenidos de la API para que el usuario verifique la exactitud del lugar.
* **Reinicio de estado:** Si el usuario cambia el municipio o la actividad en el filtro, los resultados anteriores se ocultarán inmediatamente, exigiendo una nueva búsqueda.

### Variables Disponibles por Hora
| Variable | Descripción | Unidad |
| :--- | :--- | :--- |
| `temperature_2m` | Temperatura real del aire a 2 m | $^\circ\text{C}$ |
| `apparent_temperature` | Sensación térmica | $^\circ\text{C}$ |
| `precipitation_probability` | Probabilidad de precipitación | $\%$ |
| `precipitation` | Precipitación total (Acumulada en la hora anterior) | $\text{mm}$ |
| `relative_humidity_2m` | Humedad relativa del aire | $\%$ |
| `wind_speed_10m` | Velocidad del viento a 10 m | $\text{km/h}$ |
| `wind_direction_10m` | Dirección del viento a 10 m | Grados ($^\circ$) |
| `snow_depth` | Espesor / cobertura de nieve acumulada | $\text{m}$ |

## 2. Clasificación de Actividades (Filtros del Usuario)

El usuario no podrá configurar los umbrales técnicos de evaluación. Sí verá los datos meteorológicos en las tarjetas, según las reglas de visualización. Para elegir actividad interactuará con un filtro que le permitirá elegir una **Clasificación General** o una **Actividad Específica** dentro de la siguiente jerarquía cerrada:

1. **Deportes de invierno**
   * Esquí y Snowboard
   * Patinaje sobre hielo y Hockey
   * Bobsleigh, Skeleton y Luge
2. **Deportes de verano y clima cálido**
   * Deportes acuáticos (Natación, waterpolo, vela y esquí acuático)
   * Deportes de resistencia al aire libre (Ciclismo de carretera, atletismo y triatlón)
3. **Deportes de clima ventoso o marítimo**
   * Kitesurf, Windsurf y Surf
   * Golf
4. **Deportes de interior (Indoor)**
   * Baloncesto, Bádminton, Tenis de mesa y Escalada en rocódromo

## 3. Matriz de Umbrales Técnicos (Lógica Interna)

El sistema evaluará **cualquier hora futura del día de hoy**, sin restricciones de franjas horarias (desaparecen las pausas de mediodía o límites nocturnos predefinidos).

| Actividad Específica | Sensación Térmica (`apparent_temperature`) | Prob. Precipitación (`precipitation_probability`) | Precipitación total (`precipitation`) | Viento (`wind_speed_10m`) | Espesor Nieve (`snow_depth`) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Esquí / Snowboard** | $-10 ^\circ\text{C} \le T_{\text{ap}} \le 5 ^\circ\text{C}$ | $\le 30\%$ | *Indiferente* | $\le 35\text{ km/h}$ | $\ge 0.15\text{ m}$ |
| **Patinaje / Hockey exterior** | $-10 ^\circ\text{C} \le T_{\text{ap}} \le 3 ^\circ\text{C}$ | $\le 10\%$ | $0.0\text{ mm}$ | $\le 20\text{ km/h}$ | *Indiferente* |
| **Bobsleigh / Luge** | $-15 ^\circ\text{C} \le T_{\text{ap}} \le 2 ^\circ\text{C}$ | $\le 20\%$ | *Indiferente* | $\le 25\text{ km/h}$ | *Indiferente* |
| **Acuáticos (Natación, etc.)**| $22 ^\circ\text{C} \le T_{\text{ap}} \le 35 ^\circ\text{C}$ | $\le 10\%$ | $0.0\text{ mm}$ | $\le 15\text{ km/h}$ | $= 0.00\text{ m}$ |
| **Resistencia (Ciclismo, etc.)**| $10 ^\circ\text{C} \le T_{\text{ap}} \le 25 ^\circ\text{C}$ | $\le 20\%$ | $\le 0.1\text{ mm}$ | $\le 25\text{ km/h}$ | $= 0.00\text{ m}$ |
| **Kitesurf / Surf / Windsurf**| $15 ^\circ\text{C} \le T_{\text{ap}} \le 30 ^\circ\text{C}$ | $\le 25\%$ | $\le 0.5\text{ mm}$ | $\ge 20\text{ km/h}$ | $= 0.00\text{ m}$ |
| **Golf** | $12 ^\circ\text{C} \le T_{\text{ap}} \le 28 ^\circ\text{C}$ | $\le 10\%$ | $0.0\text{ mm}$ | $\le 15\text{ km/h}$ | $= 0.00\text{ m}$ |
| **Deportes de Interior** | *Indiferente* | *Indiferente* | *Indiferente* | *Indiferente* | *Indiferente* |

> **Aclaración sobre Esquí / Snowboard:** La cantidad total y el tipo de precipitación se ignoran (son indiferentes). Se mantienen todos los demás criterios de su fila: sensación térmica entre -10 y 5 °C, probabilidad de precipitación menor o igual al 30 %, viento menor o igual a 35 km/h y espesor de nieve mayor o igual a 0,15 m. Todos estos límites son inclusivos.

## 4. Reglas de Visualización e Interfaz

La presentación de los resultados debe seguir estrictamente estas directrices de diseño y comportamiento:

### Tarjetas Horarias
1. **Mostrar TODAS las horas futuras disponibles:** Se mostrarán todas las horas futuras de hoy presentes en la respuesta, según la fecha y hora local del municipio definidas en `spec.md`, sin filtrar las que no sean buenas. Si la cobertura es incompleta, se mostrarán las horas disponibles y un aviso explícito; no se inventarán datos ni se utilizarán horas de otro día para completar la lista.
2. **Distinción Visual:**
   * **Horas Buenas:** Fondo blanco y borde azul brillante.
   * **Resto de horas:** Fondo gris y borde discontinuo.
3. **Datos Visibles en la Tarjeta:**
   * **Principal:** Temperatura real (`temperature_2m`).
   * **Secundario:** Sensación térmica (`apparent_temperature`), renderizada con menor importancia y debajo de la temperatura real.
   * **Específicos:** Se mostrarán únicamente los datos relevantes utilizados para evaluar esa actividad en concreto (ej. velocidad del viento para Surf, nieve para Esquí). Esta estructura debe ser idéntica para todas las tarjetas de la misma búsqueda.
4. **Datos Incompletos:** Si la API no devuelve algún valor necesario para la tarjeta, se mostrará el texto «Sin datos» en su lugar. La tarjeta no se ocultará, pero se añadirá un aviso indicando que "no puede evaluarse completamente".

### Mensajes y Estados Vacíos
El sistema debe diferenciar claramente entre tres escenarios mediante mensajes específicos:
1. **Ninguna hora cumple los requisitos:** Mostrar la lista de tarjetas (en gris/discontinuo) indicando que "Ninguna hora cumple las condiciones óptimas".
2. **Fin del día:** Si la búsqueda se realiza muy tarde y ya no quedan horas futuras hoy, mostrar el mensaje "No quedan horas disponibles en el día de hoy".
3. **Falta de cobertura / Error API:** Mostrar un aviso explícito si la API falla o no hay datos de cobertura para esa ubicación.
