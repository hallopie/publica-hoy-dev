# Publica Hoy — DEV

Herramienta de generación de contenido para redes sociales orientada a PyMEs mexicanas. El usuario describe su producto, elige un formato y opcionalmente sube una foto — el sistema genera copy, hashtags e imagen de marketing lista para publicar.

Desarrollado por **Somos Enlace IA / Planta Baja Producciones**.

---

## Arquitectura general

```
Dashboard (GitHub Pages)
        ↓
  Escenario 1 — Contenido (Make)
        → Claude genera copy + hashtags
        → Responde al dashboard
        ↓
  Escenario 2 — Imagen (Make)
   ├── Ruta A (sin foto): GPT Image genera imagen de marketing
   └── Ruta B (con foto): GPT Image Edit usa foto como referencia visual
        → Sube imagen a Cloudinary
        → Guarda registro en Airtable
        ↓
  Escenario 3 — Consulta (Make)
        → Dashboard polling por session_id
        → Devuelve imagen_url cuando está lista
```

---

## Componentes

### Dashboard

**URL DEV:** `https://hallopie.github.io/publica-hoy-dev/`

Interfaz HTML estática desplegada en GitHub Pages. No tiene backend propio — toda la lógica vive en Make.

**Flujo del usuario:**
1. Llena el formulario: producto, descripción, formato, foto opcional
2. El dashboard sube la foto a Cloudinary (si aplica) antes de disparar los escenarios
3. Dispara Escenario 1 → recibe copy + hashtags → los muestra
4. Dispara Escenario 2 → queda en polling contra Escenario 3
5. Cuando imagen_url está disponible en Airtable → la muestra

---

### Escenarios Make

| # | Nombre | ID | Webhook |
|---|--------|----|---------|
| 1 | `enlace-ia-pymes-contenido-DEV` | 5259408 | `derwww6xbvtnwj1fx1a5je9tdbekdkyw` |
| 2 | `enlace-ia-pymes-imagen-DEV` | 5270499 | `2qh3lj2igxhq7ojzbps5k8mgvqnodstb` |
| 3 | `enlace-ia-pymes-consulta-imagen-DEV` | 5270906 | `21lr8y44per9hmgy6ssd4kuwhqf7vl0s` |

**Escenario 1 — Contenido**
- Trigger: webhook con `{ producto, descripcion, formato, session_id }`
- Módulo Claude (Haiku): genera copy + hashtags según formato elegido
- Webhook Response: devuelve `{ copy, hashtags }` al dashboard en tiempo real

**Escenario 2 — Imagen**
- Trigger: webhook con `{ copy, hashtags, formato, session_id, foto_url? }`
- Módulo Claude (Haiku): extrae 4-6 palabras clave del copy para overlay de texto
- Router:
  - **Ruta A** (sin foto): `openai-gpt-3:GenerateImage` con `gpt-image-1`, genera imagen de marketing desde cero
  - **Ruta B** (con foto): HTTP descarga foto desde Cloudinary → `openai-gpt-3:editImage` con `gpt-image-2-2026-04-21` e `input_fidelity: medium`, usa la foto como referencia visual
- Sube resultado a Cloudinary (`/publica-hoy/promo_{session_id}`)
- Guarda en Airtable: `session_id`, `copy`, `hashtags`, `formato`, `imagen_url`, `estado: listo`
- Webhook Response: `{ status: ok }`

**Escenario 3 — Consulta de imagen**
- Trigger: webhook con `{ session_id }`
- Consulta Airtable por `session_id`
- Webhook Response: devuelve `{ imagen_url }` si existe, o `{ imagen_url: null }` si aún no hay registro

---

### Almacenamiento

**Airtable**
- Base: `appb6lwlVX9DkUxpo`
- Tabla: `tblvHDPvkaVMYAXsl`
- Campos: `session_id`, `copy`, `hashtags`, `formato`, `imagen_url`, `estado`

**Cloudinary**
- Cloud: `dk7drtfdb`
- Preset unsigned: `publica-hoy-unsigned`
- Carpeta imágenes generadas: `publica-hoy/`
- Carpeta fotos subidas por usuario: `publica-hoy-uploads/` *(convención recomendada)*

---

## Personalización por cliente

> ⚠️ Esta sección es crítica antes de pasar a PROD con un cliente específico.

Los prompts actuales están configurados para un perfil genérico de PyME mexicana. Al incorporar un cliente real, deben ajustarse:

### Escenario 1 — Prompt de copy (módulo Claude)

Ubicación en el blueprint: módulo `anthropic-claude:simpleTextPrompt`, campo `textPrompt`.

Adaptar según:
- **Tono de voz** del negocio (formal, cercano, divertido, aspiracional)
- **Categoría de producto** (alimentos, servicios, moda, automotriz, etc.)
- **Plataforma destino** (Instagram, Facebook, TikTok — cada una tiene convenciones distintas)
- **Restricciones de marca** (palabras que usar o evitar, claims específicos)

Ejemplo de ajuste para una cafetería:
```
// Antes (genérico)
"Genera copy de marketing para redes sociales para una PyME mexicana..."

// Después (cliente específico)
"Eres el community manager de Café Origen, una cafetería de especialidad en CDMX.
Tono: cálido, culto, apasionado por el café. Evitar lenguaje corporativo.
Genera copy para Instagram destacando origen del grano y proceso artesanal..."
```

### Escenario 2 — Prompt de imagen (módulo editImage / GenerateImage)

Ubicación: módulo `openai-gpt-3:editImage` (Ruta B) y `openai-gpt-3:GenerateImage` (Ruta A).

Adaptar según:
- **Paleta de colores** de la marca
- **Estilo visual** (minimalista, vibrante, rústico, premium, etc.)
- **Elementos de marca** a incluir o evitar en el fondo
- **Tipo de producto** habitual (si siempre son alimentos, ajustar el prompt base)

### Escenario 2 — Extracción de palabras clave (módulo Claude, módulo 7)

El prompt que genera el texto overlay también puede ajustarse si el cliente prefiere:
- Frases completas en lugar de palabras sueltas
- Un CTA específico siempre presente ("¡Pídelo ya!", "Solo hoy")
- Idioma mixto (español/inglés) según su audiencia

---

## Checklist antes de PROD

- [ ] Prompts de copy e imagen ajustados al cliente
- [ ] Probado con al menos 5 productos representativos del catálogo del cliente
- [ ] Probado con foto y sin foto
- [ ] Probado con fotos de baja calidad (celular, mal iluminadas)
- [ ] Escenarios duplicados con sufijo `-PROD` y webhooks nuevos
- [ ] Dashboard apuntando a webhooks PROD
- [ ] Variables de ambiente / IDs de Airtable actualizados para cuenta del cliente si aplica
- [ ] Cloudinary configurado con preset del cliente si requiere carpeta propia

---

## Notas técnicas

**`input_fidelity` en GPT Image Edit**
El parámetro `input_fidelity` controla qué tan fiel es el modelo a la imagen original:
- `high` — respeta la imagen al máximo; puede generar errores de contenido con vehículos u objetos complejos
- `medium` — balance entre fidelidad y libertad creativa; configuración actual recomendada para uso general
- `low` — usa la imagen solo como inspiración; genera resultados más genéricos

**Por qué no se usa `input_fidelity: high` en producción**
El modelo con fidelidad alta rechaza ciertas categorías de imágenes (vehículos, algunos alimentos, personas) por políticas de contenido de OpenAI. Dado que los usuarios de Publica Hoy pueden subir cualquier tipo de producto, `medium` es el punto de equilibrio más estable.

**session_id**
Generado por el dashboard al inicio de cada sesión. Es el hilo que conecta los tres escenarios y el registro en Airtable. Formato recomendado: `session_{timestamp}_{random}`.

---

## Stack

| Componente | Tecnología |
|------------|-----------|
| Frontend | HTML/CSS/JS estático — GitHub Pages |
| Automatización | Make.com |
| IA texto | Anthropic Claude (Haiku 4.5) |
| IA imagen | OpenAI GPT Image 1 / GPT Image 2 |
| Almacenamiento imágenes | Cloudinary |
| Base de datos | Airtable |
