# FacturaHub + n8n — automatiza tu facturación sin código

Workflows listos para **automatizar tu facturación con [n8n](https://n8n.io)** y la API de [FacturaHub](https://facturahub.com): cuando se paga un pedido en tu tienda (WooCommerce, Shopify) o llega un evento de cualquier ERP, n8n crea el cliente, genera la factura y la envía por email. **Cero código.**

> **n8n facturación** · **WooCommerce factura automática n8n** · **Shopify factura n8n** · **automatizar facturas España**

---

## ¿Qué es esto?

Este repo contiene **plantillas de workflows de n8n** (`workflows/*.json`) que puedes importar tal cual en tu instancia de n8n (cloud o self-hosted). Cada workflow conecta un evento de tu negocio con la API de FacturaHub para que **no vuelvas a crear facturas a mano**.

n8n es una herramienta visual de automatización: arrastras nodos, los conectas y se ejecutan solos cuando ocurre algo (un pedido pagado, un webhook, etc.). No necesitas programar.

### Sé honesto contigo: qué hay y qué no hay (todavía)

- **NO existe un nodo nativo de n8n para FacturaHub.** Usamos el nodo genérico **HTTP Request** apuntando a la API REST. Funciona igual de bien, solo es un poco más manual la primera vez.
- **NO hay webhooks salientes desde FacturaHub** (aún). El disparador siempre viene de tu tienda/ERP, no de FacturaHub.
- Lo que **sí** funciona hoy: crear clientes, crear facturas y enviarlas por email vía API, de forma 100% automatizable.

---

## El flujo

Los tres workflows siguen el mismo patrón de tres pasos sobre la API:

```
Trigger (pedido pagado / webhook)
        │
        ▼
HTTP Request  →  POST https://api.facturahub.com/api/clients     (crea/registra el cliente, devuelve _id)
        │
        ▼
HTTP Request  →  POST https://api.facturahub.com/api/invoices    (crea la factura con items[], taxRate 21%, retentionRate, clientId)
        │
        ▼
HTTP Request  →  PATCH https://api.facturahub.com/api/invoices/{id}/send   (envía la factura por email al cliente)
```

> **Host de la API: `https://api.facturahub.com`** (no uses `facturahub.com` para llamadas API — eso es la web).

---

## Workflows incluidos

| Workflow | Disparador | Para quién |
|---|---|---|
| [`workflows/woocommerce-to-invoice.json`](workflows/woocommerce-to-invoice.json) | **WooCommerce Trigger** (pedido en estado `processing`) | Tiendas WooCommerce / WordPress |
| [`workflows/shopify-to-invoice.json`](workflows/shopify-to-invoice.json) | **Shopify Trigger** (`orders/paid`) | Tiendas Shopify |
| [`workflows/webhook-to-invoice.json`](workflows/webhook-to-invoice.json) | **Webhook** (POST entrante) | Cualquier ERP, CRM o app que pueda mandar un POST |

Los tres:

1. Comprueban que el pedido está pagado (nodo IF, salvo el webhook genérico que asume pago confirmado).
2. Mapean los datos del cliente (facturación) a `POST /api/clients`.
3. Mapean las líneas del pedido (`line_items`) a los `items[]` de la factura con **IVA 21%** por defecto.
4. Crean la factura como `draft` y luego la envían con `PATCH /api/invoices/{id}/send`.

---

## Cómo importar un workflow en n8n

1. Abre tu instancia de n8n.
2. Arriba a la derecha: menú **⋮ → Import from File** (o **Import from URL** pegando el enlace raw del JSON).
3. Selecciona el `.json` que quieras de la carpeta `workflows/`.
4. Verás los nodos ya conectados. Antes de activarlo, configura las credenciales (siguiente sección).
5. Pulsa **Active** cuando todo esté listo.

---

## Autenticación (Header Auth con `x-api-key`)

La API de FacturaHub acepta dos formas de autenticación. La recomendada para n8n es la **API Key** por cabecera.

### Opción A — API Key (recomendada)

1. En FacturaHub: **Ajustes → Integraciones** y copia tu API Key.
2. En n8n: **Credentials → New → Header Auth**.
3. Rellena:
   - **Name**: `x-api-key`
   - **Value**: tu API Key
4. Guárdala (p. ej. como `FacturaHub API Key (x-api-key)`).
5. En cada nodo **HTTP Request** de los workflows, la autenticación ya está puesta como
   *Generic Credential Type → Header Auth*; solo selecciona la credencial que acabas de crear (en las plantillas aparece el placeholder `REEMPLAZA_CREDENCIAL_FACTURAHUB`).

### Opción B — JWT (Bearer)

Si prefieres token de sesión, haz primero `POST https://api.facturahub.com/auth/login` con `{ "email", "password" }`, coge el token de la respuesta y úsalo como cabecera `Authorization: Bearer <jwt>`. Los JWT caducan, así que para automatizaciones permanentes la API Key es mejor.

---

## Campos que usa cada llamada (reales, según el OpenAPI)

**`POST /api/clients`** — body JSON:
```json
{ "name": "...", "taxId": "...", "email": "...", "address": "...", "country": "ES" }
```
Devuelve `{ "_id": "...", ... }` → ese `_id` es el `clientId` de la factura.

**`POST /api/invoices`** — body JSON:
```json
{
  "clientId": "<_id del cliente>",
  "items": [
    { "description": "Producto", "quantity": 1, "unitPrice": 10.00, "taxRate": 21 }
  ],
  "retentionRate": 0,
  "currency": "EUR",
  "status": "draft"
}
```
Devuelve `{ "_id": "...", "number": "...", ... }`.

**`PATCH /api/invoices/{id}/send`** — sin body; envía la factura por email. Usa el `_id` devuelto por el paso anterior.

---

## Personalizar

- **IVA**: cambia `taxRate: 21` por el tipo que corresponda (10, 4, 0...).
- **Retención IRPF**: ajusta `retentionRate` (p. ej. `15` para autónomos con retención del 15%).
- **No enviar automáticamente**: borra el último nodo (`PATCH .../send`) y la factura quedará en borrador en FacturaHub.
- **Otra moneda**: cambia `currency`.

---

## Enlaces

- **API (código y referencia)**: https://github.com/Santy1422/facturahub-api
- **Swagger / OpenAPI**: https://facturahub.com/api-docs/
- **Guía n8n (paso a paso)**: https://facturahub.com/es/spain/n8n
- **FacturaHub**: https://facturahub.com

---

## Licencia

MIT. Úsalo, modifícalo y compártelo libremente.
