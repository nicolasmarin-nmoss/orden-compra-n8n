# Generador de Orden de Compra - n8n

Workflow de n8n que genera órdenes de compra automáticamente a partir de:
- **PDF** (cotizaciones, presupuestos)
- **Excel / CSV** (tablas de productos, listas de precios)
- **Imágenes** (JPG, PNG — fotos de cotizaciones, facturas escaneadas)
- **Texto libre** (cualquier descripción del pedido)

---

## Requisitos

- n8n instalado (v1.x o superior)
- API Key de Anthropic (Claude)
- Node.js (si ejecutas n8n localmente)

---

## Instalación

### 1. Importar el workflow

1. Abre tu instancia de n8n
2. Ve a **Workflows → Import from File**
3. Selecciona el archivo `workflow.json`

### 2. Configurar las credenciales de Anthropic

1. En n8n, ve a **Credentials → Add Credential**
2. Busca **"Header Auth"**
3. Configura:
   - **Name:** `Anthropic API Key`
   - **Name (header):** `x-api-key`
   - **Value:** `sk-ant-xxxxxxxxxxxxx` (tu API key de Anthropic)
4. Guarda la credencial con el nombre `Anthropic API Key`
5. En el workflow, los nodos `Claude Vision - Leer Imagen` y `Claude - Generar Orden de Compra` ya están configurados para usar esta credencial

### 3. Activar el workflow

1. Abre el workflow importado
2. Haz clic en el toggle **"Active"** (arriba a la derecha)
3. El webhook quedará disponible en:
   ```
   https://TU-INSTANCIA/webhook/orden-compra
   ```

---

## Uso

### Opción A: Formulario Web

1. Abre `test_form.html` en tu navegador
2. Configura la URL del webhook (ej: `http://localhost:5678/webhook/orden-compra`)
3. Sube un archivo o ingresa texto
4. Haz clic en **"Generar Orden de Compra"**
5. Descarga el resultado en HTML o JSON

### Opción B: API REST directa

**Subir un archivo:**
```bash
curl -X POST https://TU-INSTANCIA/webhook/orden-compra \
  -F "file=@cotizacion.pdf"
```

**Enviar texto:**
```bash
curl -X POST https://TU-INSTANCIA/webhook/orden-compra \
  -H "Content-Type: application/json" \
  -d '{
    "texto": "Proveedor: ABC SpA, RUT 12.345.678-9\nProducto: 10 sillas ergonómicas a $120.000 c/u\nEntrega: 5 días hábiles"
  }'
```

### Respuesta

```json
{
  "success": true,
  "ordenDeCompra": {
    "numero": "OC-20260319-001",
    "fecha": "2026-03-19",
    "proveedor": {
      "nombre": "ABC SpA",
      "rut": "12.345.678-9",
      ...
    },
    "items": [
      {
        "linea": 1,
        "descripcion": "Silla ergonómica",
        "cantidad": 10,
        "unidad": "UN",
        "precioUnitario": 120000,
        "total": 1200000
      }
    ],
    "subtotal": 1200000,
    "iva": 228000,
    "total": 1428000,
    "moneda": "CLP",
    ...
  },
  "htmlOrden": "<html>...orden formateada...</html>"
}
```

---

## Diagrama del Workflow

```
[Webhook POST /orden-compra]
         │
         ▼
[Detectar Tipo de Entrada]
   (pdf / excel / image / text)
         │
         ▼
[Switch: Rutear por Tipo]
   │         │         │         │
  PDF      Excel    Imagen    Texto
   │         │         │         │
[Extraer] [Extraer] [Base64] [Directo]
   │         │         │
   │         │    [Claude Vision]
   │         │         │
[Norm.]  [Norm.]   [Norm.]   [Norm.]
   │         │         │         │
   └─────────┴─────────┴─────────┘
                   │
                   ▼
    [Claude - Generar Orden de Compra]
                   │
                   ▼
       [Formatear (JSON + HTML)]
                   │
                   ▼
        [Responder al Cliente]
```

---

## Personalización

### Cambiar el modelo de Claude

En los nodos `Claude Vision - Leer Imagen` y `Claude - Generar Orden de Compra`, cambia:
```
"claude-sonnet-4-6"  →  "claude-opus-4-6"  (más preciso, más lento)
                     →  "claude-haiku-4-5-20251001"  (más rápido, menos preciso)
```

### Cambiar el IVA

En el nodo `Claude - Generar Orden de Compra`, modifica el system prompt:
```
"porcentajeIva": 19   →  0 para sin IVA
```

### Agregar envío por email

Añade un nodo de **Gmail** o **SMTP** después de `Formatear Orden de Compra` y adjunta el HTML como cuerpo del email.

### Guardar en Google Sheets / base de datos

Añade un nodo de **Google Sheets**, **Airtable**, o **Postgres** después de formatear para almacenar las órdenes generadas.

---

## Troubleshooting

| Problema | Solución |
|----------|----------|
| `401 Unauthorized` en Claude | Verifica que la API key esté correcta en las credenciales |
| PDF sin texto extraído | El PDF puede ser una imagen escaneada; usa la ruta de imagen |
| Error al parsear Excel | Asegúrate de que el archivo sea .xlsx o .csv válido |
| Webhook no responde | Verifica que el workflow esté **Active** en n8n |
| CORS en el formulario | Agrega `Access-Control-Allow-Origin: *` en las respuestas del webhook (ya configurado) |
