# ITA Piel · Fitzpatrick

Web app (PWA) para registrar, por participante, el **ITA°** de la piel (desde una foto de celular) y el **fototipo Fitzpatrick** (por cuestionario), como variables de entrada para modelos de ML sobre señales PPG.

Todo funciona dentro del celular: sin servidor, sin cuenta y sin internet después de la primera carga. Los datos se guardan en el navegador y salen en un CSV.

## Publicarla (la cámara necesita HTTPS)

Sube la carpeta completa (`index.html`, `manifest.webmanifest`, `sw.js`, `icon.svg`) a cualquier hosting estático con HTTPS:

- **Netlify Drop**: entra a app.netlify.com/drop y arrastra la carpeta. Te da una URL `https://…`.
- **GitHub Pages**: crea un repositorio, sube los archivos y activa Pages.

Abre la URL en el celular y, en el menú del navegador, elige **“Agregar a pantalla de inicio”**. La página no contiene datos: los registros nunca salen del celular hasta que exportas el CSV.

Sin HTTPS (por ejemplo abriendo el archivo directamente) solo funciona el botón **“Usar app de cámara”**.

## Protocolo de captura

1. **Sin flash.** Iluminación del laboratorio constante, sin luz solar directa ni sombras.
2. **Sitio:** cara dorsal del antebrazo distal, centrado entre radio y cúbito, 3 traveses de dedo proximal a la articulación de la muñeca (el mismo punto del sensor PPG). Mide **antes** de colocar el sensor, porque la presión deja marca.
3. Celular paralelo a la piel y a distancia fija; usa un separador (por ejemplo un tubo de 15 cm).
4. **Tarjeta gris 18 %** en el mismo plano que la piel, dentro del círculo amarillo.
5. Dentro del círculo verde no debe haber vello denso, venas visibles, lunares ni tatuajes. Toca la imagen para mover el círculo.
6. Al inicio de cada sesión pulsa **“Fijar cámara”** con la piel y la tarjeta en cuadro. La app guarda esos valores y los reaplica a todos los participantes.
7. Por defecto se toman 3 fotos por participante; se guardan el promedio y la desviación estándar.

## Cálculo

1. Los píxeles dentro del círculo se pasan de sRGB a RGB lineal.
2. Se calcula una media robusta: se descarta el 10 % más oscuro y el 10 % más brillante (vello, poros y brillos).
3. **Corrección con tarjeta gris:** cada canal se escala para que la tarjeta valga 0,18 (corrección diagonal von Kries). Esto corrige el balance de blancos y la exposición.
4. RGB lineal → XYZ → CIE L\*a\*b\* (D65).
5. `ITA° = arctan((L* − 50) / b*) × 180 / π`.
6. Categorías (Chardon 1991; Del Bino 2006):

   | Categoría | ITA° |
   |---|---|
   | Muy clara | > 55 |
   | Clara | 41–55 |
   | Intermedia | 28–41 |
   | Bronceada | 10–28 |
   | Marrón | −30–10 |
   | Oscura | < −30 |

**Cuestionario Fitzpatrick:** es la versión puntuada de 10 preguntas (0–40 puntos), en tres partes: rasgos heredados (1–4), reacción al sol (5–8) y hábitos de exposición (9–10).

| Puntaje | Tipo |
|---|---|
| 0–6 | I |
| 7–13 | II |
| 14–20 | III |
| 21–27 | IV |
| 28–34 | V |
| ≥ 35 | VI |

El CSV guarda cada respuesta (`fitz_q1`…`fitz_q10`), así que después puedes recalcular con otra versión, por ejemplo solo las preguntas de quemadura y bronceado (q5, q6).

## Columnas del CSV

| Columna | Descripción |
|---|---|
| participant_id | ID escrito por el operador (el mismo de la grabación PPG) |
| timestamp | Fecha y hora de registro (ISO 8601 con zona horaria) |
| photo_time_first | Hora de la primera foto |
| operator, device_label, body_site | Datos de configuración |
| n_photos | Fotos aceptadas |
| L_mean, a_mean, b_mean | CIE L\*a\*b\* promedio (corregido con tarjeta gris si se usó) |
| ITA_mean, ITA_sd | ITA° promedio y desviación estándar entre fotos |
| ITA_per_photo | ITA de cada foto, separados por `;` |
| ITA_category / ITA_category_es | Categoría ITA (código en inglés / texto en español) |
| ITA_uncorrected_mean | ITA sin corrección de tarjeta gris (para control) |
| gray_card_used | `true` si todas las fotos tuvieron una tarjeta válida |
| wb_gains_rgb | Ganancias R/G/B aplicadas, por foto |
| capture_source | `camara_web` (cámara en vivo) o `app_camara` (app nativa) |
| cam_locked, cam_* | Estado y valores de exposición, ISO y temperatura de color, cuando el navegador los informa |
| image_size | Resolución de la imagen analizada |
| quality_flags | Alertas de calidad, separadas por `;` (ver abajo) |
| fitz_q1 … fitz_q10 | Respuesta de cada pregunta (0–4) |
| fitz_score, fitz_type, fitz_type_num | Puntaje 0–40, tipo en romano (I–VI) y en número (1–6) |
| notes | Notas del operador |
| user_agent, app_version | Trazabilidad |

**Alertas de calidad:**

| Alerta | Significado |
|---|---|
| `piel_saturada` | Más del 0,5 % de los píxeles de piel están saturados |
| `piel_subexpuesta` | Más del 5 % de los píxeles de piel son casi negros |
| `piel_no_uniforme` | La desviación estándar de L\* en la piel es mayor que 5 |
| `tarjeta_*` | Problemas con la tarjeta gris (saturada, oscura o no uniforme) |
| `correccion_extrema` | Alguna ganancia de color es mayor que 5 o menor que 0,2 |
| `sin_tarjeta_gris` | Foto tomada sin tarjeta gris |
| `b_bajo_ita_inestable` | b\* ≤ 1, el ITA no es fiable |
| `variabilidad_alta` | La desviación estándar del ITA entre fotos es mayor que 3° |
| `roi_pequena` | El círculo de piel tiene muy pocos píxeles |

## Limitaciones (para la sección de métodos)

- El ITA de celular es **relativo al dispositivo**. Usa un solo celular para todo el estudio; la concordancia entre celulares distintos es baja (ICC ≈ 0,40, arXiv 2512.21988).
- La imagen del navegador ya viene procesada por la cámara (no es RAW). Fijar la cámara y usar la tarjeta gris reducen esa variación, pero no la eliminan.
- Valida en una submuestra (unos 20 participantes que cubran todo el rango de tonos) contra un colorímetro o espectrofotómetro y reporta la correlación.
- El ITA mide **color**; el Fitzpatrick mide **respuesta al sol**. No son intercambiables: usa los dos como variables separadas.

## Referencias

- Chardon A, Cretois I, Hourseau C. Skin colour typology and suntanning pathways. *Int J Cosmet Sci.* 1991;13:191–208.
- Del Bino S, et al. Relationship between skin response to ultraviolet exposure and skin color type. *Pigment Cell Res.* 2006;19:606–614.
- Fitzpatrick TB. The validity and practicality of sun-reactive skin types I through VI. *Arch Dermatol.* 1988;124:869–871.
- Eilers S, et al. Accuracy of self-report in assessing Fitzpatrick skin phototypes I through VI. *JAMA Dermatol.* 2013;149:1289–1294.
- Smartphone-based ITA (SITA): arXiv 2411.13832.
- Color-clinical decoupling in smartphone dermatology: arXiv 2512.21988.
