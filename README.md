# Publica Hoy

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

Interfaz HTML estática desplegada en GitHub Pages. No tiene backend propio — toda la lógica vive en Make.

**Flujo del usuario:**
1. Llena el formulario: producto, descripción, formato, foto opcional
2. El dashboard sube la foto a Cloudinary (si aplica) antes de disparar los escenarios
3. Dispara Escenario 1 → recibe copy + hashtags → los muestra
4. Dispara Escenario 2 → queda en polling contra Escenario 3
5. Cuando imagen_url está disponible en Airtable → la muestra

---

### Escenarios Make

| # | Función |
|---|---------|
| 1 | Generación de copy + hashtags |
| 2 | Generación de imagen |
| 3 | Consulta de imagen por session_id |

> ⚠️ Los IDs de escenarios y webhooks se gestionan en variables de entorno del dashboard. Ver sección de configuración.

**Escenario 1 — Contenido**
- Trigger: webhook con `{ producto, descripcion, formato, session_id }`
- Módulo Claude (Haiku): genera copy + hashtags según formato elegido
- Webhook Response: devuelve `{ copy, hashtags }` al dashboard en tiempo real

**Escenario 2 — Imagen**
- Trigger: webhook con `{ copy, hashtags, formato, session_id, foto_url? }`
- Módulo Claude (Haiku): extrae 4-6 palabras clave del copy para overlay de texto
- Router:
  - **Ruta A** (sin foto): `GenerateImage` con gpt-image-1, genera imagen de marketing desde cero
  - **Ruta B** (con foto): descarga foto → `editImage` con gpt-image-2 e `input_fidelity: medium`, usa la foto como referencia visual
- Sube resultado a Cloudinary
- Guarda en Airtable: `session_id`, `copy`, `hashtags`, `formato`, `imagen_url`, `estado`
- Webhook Response: `{ status: ok }`

**Escenario 3 — Consulta de imagen**
- Trigger: webhook con `{ session_id }`
- Consulta Airtable por `session_id`
- Webhook Response: devuelve `{ imagen_url }` si existe, o `{ imagen_url: null }` si aún no hay registro

---

### Configuración del dashboard

El dashboard lee sus endpoints desde constantes al inicio del archivo `index.html`. Al instalar una nueva instancia para un cliente, reemplazar estos valores:

```javascript
const CONFIG = {
  WEBHOOK_CONTENIDO: "https://hook.make.com/REEMPLAZAR",
  WEBHOOK_IMAGEN:    "https://hook.make.com/REEMPLAZAR",
  WEBHOOK_CONSULTA:  "https://hook.make.com/REEMPLAZAR",
  CLOUDINARY_CLOUD:  "REEMPLAZAR",
  CLOUDINARY_PRESET: "REEMPLAZAR"
};
```

---

### Almacenamiento

**Airtable**
- Una base por cliente
- Campos de la tabla principal: `session_id`, `copy`, `hashtags`, `formato`, `imagen_url`, `estado`

**Cloudinary**
- Un preset unsigned por cliente
- Carpeta imágenes generadas: `publica-hoy/`
- Carpeta fotos subidas por usuario: `publica-hoy-uploads/`

---

## Personalización por cliente

> ⚠️ Esta sección es crítica antes de pasar a PROD con un cliente específico.

Los prompts base están configurados para un perfil genérico de PyME mexicana. Al incorporar un cliente real, ajustar:

### Escenario 1 — Prompt de copy

Adaptar según:
- **Tono de voz** del negocio (formal, cercano, divertido, aspiracional)
- **Categoría de producto** (alimentos, servicios, moda, automotriz, etc.)
- **Plataforma destino** (Instagram, Facebook, TikTok)
- **Restricciones de marca** (palabras que usar o evitar, claims específicos)

Ejemplo para una cafetería:
```
// Antes (genérico)
"Genera copy de marketing para redes sociales para una PyME mexicana..."

// Después (cliente específico)
"Eres el community manager de Café Origen, una cafetería de especialidad en CDMX.
Tono: cálido, culto, apasionado por el café. Evitar lenguaje corporativo.
Genera copy para Instagram destacando origen del grano y proceso artesanal..."
```

### Escenario 2 — Prompt de imagen

Adaptar según:
- **Paleta de colores** de la marca
- **Estilo visual** (minimalista, vibrante, rústico, premium)
- **Tipo de producto** habitual del cliente

### Escenario 2 — Palabras clave para overlay

El prompt de extracción de palabras también puede ajustarse si el cliente prefiere frases completas, un CTA fijo, o idioma específico.

---

## Checklist de instalación por cliente

- [ ] Duplicar los 3 escenarios Make con nomenclatura `[cliente]-publica-hoy-[función]-DEV`
- [ ] Crear tabla en Airtable
- [ ] Crear carpeta y preset en Cloudinary
- [ ] Ajustar prompts según brief del cliente
- [ ] Actualizar `CONFIG` en el dashboard con los webhooks nuevos
- [ ] Probar con al menos 5 productos representativos del catálogo
- [ ] Probar con foto y sin foto
- [ ] Probar con fotos de baja calidad (celular, mal iluminadas)
- [ ] Duplicar escenarios a `-PROD` y actualizar dashboard
- [ ] Entregar accesos al cliente

---

## Notas técnicas

**`input_fidelity` en GPT Image Edit**

Controla qué tan fiel es el modelo a la imagen original:
- `high` — máxima fidelidad; puede rechazar ciertas categorías (vehículos, personas) por políticas de OpenAI
- `medium` — balance entre fidelidad y libertad creativa; **configuración recomendada para uso general**
- `low` — usa la imagen solo como inspiración; resultados más genéricos

Se usa `medium` porque los usuarios pueden subir cualquier tipo de producto y `high` genera errores de contenido en categorías comunes como vehículos o bebidas.

**session_id**

Generado por el dashboard al inicio de cada sesión. Conecta los tres escenarios y el registro en Airtable. Formato recomendado: `session_{timestamp}_{random}`.

---

## Stack

| Componente | Tecnología |
|------------|-----------|
| Frontend | HTML/CSS/JS estático — GitHub Pages |
| Automatización | Make.com |
| IA texto | Anthropic Claude Haiku |
| IA imagen | OpenAI GPT Image 1 / GPT Image 2 |
| Almacenamiento imágenes | Cloudinary |
| Base de datos | Airtable |
