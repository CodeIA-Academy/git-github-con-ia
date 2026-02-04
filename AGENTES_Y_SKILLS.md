# Agentes, Subagentes, Skills, Contexto y Tools - Luckyled Ecommerce

**Fuente:** https://github.com/yosnap/luckyled-ecommerce.git

---

## 1. AGENTES DISPONIBLES

### 1.1 SEO Agent (Agente Principal)

**Tipo:** Microservicio independiente en Python
**Ubicación:** `/seo-agent/`
**Framework:** FastAPI + Uvicorn
**Puerto:** 8001
**Propósito:** Generación automática de metadatos SEO para e-commerce

#### Características Principales
- Generación de metadatos SEO (meta title, meta description, keywords, OG tags)
- Procesamiento batch asincrónico
- Soporte para múltiples modelos IA via OpenRouter
- Integración con PostgreSQL para persistencia
- Validación de longitudes y formato

#### Modelos Soportados
```
- anthropic/claude-3.5-haiku (recomendado para bulk)
- anthropic/claude-3-sonnet (calidad balanceada)
- openai/gpt-4o-mini (económico)
- google/gemini-2.0-flash (alternativa Google)
```

#### Configuración Default
```python
default_model: "anthropic/claude-3.5-haiku"
batch_size: 50
parallel_requests: 5
max_retries: 3
temperature: 0.7
meta_title_length: 50-60 caracteres
meta_desc_length: 150-160 caracteres
keywords: 3-5 palabras clave
```

---

## 2. SUBAGENTES / GENERADORES ESPECIALIZADOS (SKILLS)

El SEO Agent implementa **3 generadores especializados** que actúan como skills:

### 2.1 ProductSEOGenerator

**Archivo:** `/seo-agent/generators/product_seo.py`
**Entidades:** Productos LED
**Responsabilidad:** Generar metadatos SEO para productos

#### Campos Generados
```
- metaTitle (50-60 caracteres)
- metaDesc (150-160 caracteres)
- metaKeywords (3-5 palabras)
- ogTitle
- ogDescription
- ogImage
- slug (generado automáticamente)
```

#### Contexto Consumido
```
- Nombre del producto
- Descripción (limpiada de HTML)
- Categoría(s) asociadas
- Precio
- SKU
- URL de imagen del producto
- Atributos adicionales
```

#### Ejemplo de Entrada
```json
{
  "id": 123,
  "name": "LED RGB 10W Profesional",
  "description": "<p>LED de alta calidad RGB...</p>",
  "category_names": ["Iluminación", "LED Profesional"],
  "sku": "LED-RGB-10W-001",
  "normal_price": 45.99,
  "image_url": "https://cdn.luckyled.es/products/led-rgb-10w.jpg"
}
```

#### Ejemplo de Salida
```json
{
  "metaTitle": "LED RGB 10W Profesional - Iluminación de Calidad | LuckyLED",
  "metaDesc": "LED RGB 10W profesional para eventos. Tecnología avanzada, colores vibrantes. Compra en LuckyLED con envío rápido.",
  "metaKeywords": "LED RGB, iluminación profesional, LED 10W",
  "ogTitle": "LED RGB 10W Profesional",
  "ogDescription": "Equipo LED RGB de 10W para eventos y espacios profesionales",
  "ogImage": "https://cdn.luckyled.es/products/led-rgb-10w.jpg"
}
```

---

### 2.2 CategorySEOGenerator

**Archivo:** `/seo-agent/generators/category_seo.py`
**Entidades:** Categorías de productos
**Responsabilidad:** Generar metadatos SEO para categorías jerárquicas

#### Campos Generados
Mismo formato que ProductSEOGenerator

#### Contexto Consumido
```
- Nombre de la categoría
- Descripción de la categoría
- Categoría padre (si existe)
- Contador de productos en la categoría
```

#### Ejemplo de Entrada
```json
{
  "id": 5,
  "name": "Proyectores LED",
  "description": "Proyectores LED profesionales para eventos",
  "parent_name": "Iluminación",
  "product_count": 24,
  "image": "https://cdn.luckyled.es/categories/proyectores.jpg"
}
```

---

### 2.3 TagSEOGenerator

**Archivo:** `/seo-agent/generators/tag_seo.py`
**Entidades:** Tags/etiquetas de productos
**Responsabilidad:** Generar metadatos SEO para etiquetas

#### Campos Generados
Mismo formato que ProductSEOGenerator

#### Contexto Consumido
```
- Nombre del tag
- Descripción del tag
- Color asociado al tag
- Contador de productos con el tag
```

---

## 3. ENDPOINTS / HERRAMIENTAS (TOOLS)

### 3.1 API REST del SEO Agent

**Base URL:** `http://localhost:8001`

#### Endpoints de Información

```http
GET /api/models
  Descripción: Obtiene lista dinámicamente de modelos disponibles en OpenRouter
  Parámetros:
    - search (opcional): Filtrar por nombre
    - provider (opcional): Filtrar por proveedor (anthropic, openai, google)
  Respuesta: Array<ModelInfo>
    - id: string (identificador del modelo)
    - name: string (nombre legible)
    - pricing: { input: number, output: number } (costo por token)
    - context_length: number (tokens máximos)
    - variants: string[] (variantes disponibles)
```

```http
GET /api/models/variants
  Descripción: Retorna las variantes de modelos disponibles
  Respuesta:
    - free: "Versión gratuita de modelo"
    - extended: "Contexto extendido"
    - thinking: "Razonamiento extendido"
    - online: "Búsqueda web en tiempo real"
    - nitro: "Inferencia de alta velocidad"
```

```http
GET /api/models/recommended
  Descripción: Retorna modelos optimizados para tareas SEO
  Respuesta: Array de modelos recomendados por caso de uso
```

```http
GET /api/stats
  Descripción: Obtiene estadísticas globales de SEO
  Respuesta: SEOStats
    - products: { total, without_seo, with_seo }
    - categories: { total, without_seo, with_seo }
    - tags: { total, without_seo, with_seo }
```

```http
GET /api/health
  Descripción: Health check del servicio SEO Agent
  Respuesta: { status: "healthy", database: "connected" }
```

---

#### Endpoints de Generación

```http
POST /api/generate/single
  Descripción: Genera SEO para una entidad individual (preview, sin guardar)
  Body:
    {
      "entity_type": "product" | "category" | "tag",
      "entity_id": number,
      "model_id": string,
      "temperature": number (0-1)
    }
  Respuesta: GenerationResult
    {
      "success": boolean,
      "data": {
        "metaTitle": string,
        "metaDesc": string,
        "metaKeywords": string,
        "ogTitle": string,
        "ogDescription": string,
        "ogImage": string
      },
      "tokens_used": { input: number, output: number },
      "cost": number
    }
  Status Code: 200 | 400 | 500
```

```http
POST /api/generate/batch
  Descripción: Inicia procesamiento batch asincrónico
  Body:
    {
      "entity_type": "product" | "category" | "tag",
      "entity_ids": number[] (opcional, si no: procesa todos sin SEO),
      "batch_size": number (default: 50),
      "settings": {
        "model_id": string,
        "temperature": number,
        "parallel_requests": number (default: 5)
      }
    }
  Respuesta: { job_id: string, status: "queued" }
  Status Code: 202 (Accepted)
```

```http
GET /api/generate/batch/{job_id}
  Descripción: Obtiene progreso de un job batch
  Respuesta: BatchProgress
    {
      "job_id": string,
      "status": "pending" | "processing" | "completed" | "failed",
      "processed": number,
      "successful": number,
      "failed": number,
      "total": number,
      "current_entity": {
        "id": number,
        "type": string,
        "name": string
      },
      "estimated_completion": "ISO datetime"
    }
```

```http
DELETE /api/generate/batch/{job_id}
  Descripción: Cancela un job batch en ejecución
  Respuesta: { job_id: string, status: "cancelled" }
```

```http
POST /api/preview
  Descripción: Genera SEO sin guardar en la base de datos
  Body: (same as /api/generate/single)
  Respuesta: (same as /api/generate/single)
```

---

### 3.2 Router tRPC en Next.js

**Ubicación:** `/src/server/api/routers/seo.ts`

```typescript
export const seoRouter = createTRPCRouter({
  // Lectura
  getByPageKey(pageKey: string)
    // Obtiene SEO metadata de una página específica
    // pageKey: 'home' | 'productos' | 'ofertas' | etc.

  listStaticPages()
    // Lista todas las páginas SEO estáticas configuradas

  // Escritura (Admin)
  upsert(seoSettings: SeoPageSettings)
    // Crear o actualizar configuración SEO de página

  update(pageKey: string, data: Partial<SeoPageSettings>)
    // Actualizar parcialmente

  delete(pageKey: string)
    // Eliminar configuración SEO
})
```

#### Páginas SEO Preconfiguradas
```
- home
- productos
- ofertas
- contacto
- nosotros
- ayuda
- carrito
- checkout
- login
- registro
- politica-privacidad
- terminos
- cookies
- envios
- devoluciones
```

---

### 3.3 Proxy API en Next.js

**Ubicación:** `/src/app/api/seo-agent/[...path]/route.ts`

Actúa como intermediario entre el frontend y el SEO Agent Python:

```
GET  /api/seo-agent/api/models                    → GET http://localhost:8001/api/models
GET  /api/seo-agent/api/stats                     → GET http://localhost:8001/api/stats
POST /api/seo-agent/api/generate/single           → POST http://localhost:8001/api/generate/single
POST /api/seo-agent/api/generate/batch            → POST http://localhost:8001/api/generate/batch
GET  /api/seo-agent/api/generate/batch/{job_id}  → GET http://localhost:8001/api/generate/batch/{job_id}
```

---

## 4. CONTEXTO DISPONIBLE

### 4.1 Configuración Global (config.py)

```python
@dataclass
class Config:
    # Base de Datos
    database_url: str
      # PostgreSQL connection string
      # Ejemplo: postgresql://user:pass@localhost:5432/luckyled

    # OpenRouter API
    openrouter_api_key: str
      # API key para acceder a OpenRouter

    openrouter_site_url: str = "https://luckyled.es"
      # URL del sitio para OpenRouter headers

    # Modelo y Temperatura
    default_model: str = "anthropic/claude-3.5-haiku"
      # Modelo LLM a usar por default

    default_temperature: float = 0.7
      # Temperatura de generación (0-1)

    # Procesamiento Batch
    batch_size: int = 50
      # Entidades procesadas por lote

    parallel_requests: int = 5
      # Requests concurrentes al LLM

    max_retries: int = 3
      # Reintentos en caso de error

    # Restricciones SEO
    meta_title_min: int = 50
    meta_title_max: int = 60
      # Rango de caracteres para meta title

    meta_desc_min: int = 150
    meta_desc_max: int = 160
      # Rango de caracteres para meta description

    keywords_min: int = 3
    keywords_max: int = 5
      # Rango de palabras clave

    delay_between_batches: float = 1.0
      # Segundos entre lotes
```

---

### 4.2 Sistema de Prompts Contextuales

**Ubicación:** `/seo-agent/prompts/`

#### Estructura de Prompts

Cada generador utiliza un prompt especializado:

```
/seo-agent/prompts/
├── product.txt     # Prompt para productos
├── category.txt    # Prompt para categorías
└── tag.txt         # Prompt para tags
```

#### Características del Prompt de Producto

```
- Restricciones estrictas de caracteres
- Ejemplos correctos incluidos
- Formato esperado: JSON
- Regla de terminación: termina con "| LuckyLED"
- Idioma: Español de España
- Validación de palabras clave relevantes
```

#### Ejemplo de Estructura de Prompt

```
Eres un experto en SEO para e-commerce de iluminación LED.

Información del producto:
- Nombre: {name}
- Descripción: {description}
- Categoría: {category}
- Precio: {price}
- SKU: {sku}

Genera metadatos SEO con ESTRICTAMENTE:
- Meta Title: 50-60 caracteres, incluir marca "LuckyLED"
- Meta Description: 150-160 caracteres
- Keywords: 3-5 palabras clave (separadas por comas)

Retorna JSON válido:
{
  "metaTitle": "...",
  "metaDesc": "...",
  "metaKeywords": "...",
  "ogTitle": "...",
  "ogDescription": "...",
  "ogImage": "..."
}
```

---

### 4.3 Modelos de Base de Datos

**Archivo:** `/seo-agent/models/database.py`

#### Modelo: Product

```python
@dataclass
class Product:
    id: int
    name: str
    slug: str
    description: str (HTML limpiable)
    short_desc: str
    sku: str
    normal_price: float
    category_names: List[str]
    attributes: Dict[str, any]
    image_url: str

    # SEO Fields
    meta_title: Optional[str]
    meta_desc: Optional[str]
    meta_keywords: Optional[str]
    og_title: Optional[str]
    og_description: Optional[str]
    og_image: Optional[str]
```

#### Modelo: Category

```python
@dataclass
class Category:
    id: int
    name: str
    slug: str
    description: str
    parent_name: Optional[str]
    product_count: int
    image: Optional[str]

    # SEO Fields
    meta_title: Optional[str]
    meta_desc: Optional[str]
    og_title: Optional[str]
    og_description: Optional[str]
    og_image: Optional[str]
```

#### Modelo: Tag

```python
@dataclass
class Tag:
    id: int
    name: str
    slug: str
    description: str
    color: str
    product_count: int

    # SEO Fields
    meta_title: Optional[str]
    meta_description: Optional[str]
    og_image: Optional[str]
```

---

### 4.4 Funciones de Base de Datos Disponibles

**Archivo:** `/seo-agent/models/database.py`

```python
# Lectura Individual
get_product_by_id(product_id: int) -> Product
get_category_by_id(category_id: int) -> Category
get_tag_by_id(tag_id: int) -> Tag

# Lectura Batch (sin SEO)
get_products_without_seo(limit: int = 100) -> List[Product]
get_categories_without_seo(limit: int = 100) -> List[Category]
get_tags_without_seo() -> List[Tag]

# Actualización
update_product_seo(product_id: int, seo_data: Dict) -> bool
update_category_seo(category_id: int, seo_data: Dict) -> bool
update_tag_seo(tag_id: int, seo_data: Dict) -> bool

# Estadísticas
get_seo_stats() -> SEOStats
  Retorna:
    {
      "products": {
        "total": int,
        "without_seo": int,
        "with_seo": int
      },
      "categories": {...},
      "tags": {...}
    }

# Validación
validate_seo_content(content: str) -> ValidationResult
```

---

### 4.5 Cliente OpenRouter

**Archivo:** `/seo-agent/openrouter_client.py`

#### Características

```python
class OpenRouterClient:
    def __init__(api_key: str, site_url: str)

    # Cache dinámico de modelos
    async get_available_models() -> List[ModelInfo]

    # Inferencia con manejo de errores
    async generate(
        model_id: str,
        messages: List[Dict],
        temperature: float = 0.7,
        max_tokens: int = 2000
    ) -> GenerationResponse

    # Detección de variantes
    def get_model_variants(model_id: str) -> List[str]

    # Cálculo de costos
    def calculate_cost(
        model_id: str,
        input_tokens: int,
        output_tokens: int
    ) -> float
```

#### Variantes de Modelos

```
:free      - Versión gratuita del modelo
:thinking  - Razonamiento extendido (más caro, mejor calidad)
:online    - Búsqueda web en tiempo real
:nitro     - Inferencia de alta velocidad

Ejemplo: "anthropic/claude-3.5-haiku:nitro"
```

---

## 5. ARQUITECTURA DE INTEGRACIÓN

### 5.1 Flujo Completo de Generación SEO

```
┌─────────────────────┐
│   Frontend Admin    │
│  /admin/seo-gen     │
└──────────┬──────────┘
           │
           │ Selecciona:
           │ - Tipo (product/category/tag)
           │ - Modelo IA
           │ - Temperature
           │ - Batch size
           │
           ▼
┌─────────────────────────────┐
│   Next.js Server (tRPC)     │
│  /src/server/api/routers/   │
└──────────┬──────────────────┘
           │
           │ Llama proxy
           │
           ▼
┌─────────────────────────────┐
│  Next.js API Route          │
│  /api/seo-agent/[...path]   │
│  (Redirecciona a)           │
└──────────┬──────────────────┘
           │
           │ HTTP POST/GET
           │
           ▼
┌──────────────────────────────┐
│   SEO Agent (FastAPI)        │
│   http://localhost:8001      │
│                              │
│  1. Carga entidades del DB   │
│  2. Limpia HTML              │
│  3. Construye prompt         │
│  4. Llama OpenRouter API     │
│  5. Parsea respuesta JSON    │
│  6. Valida longitudes        │
│  7. Guarda en DB (batch)     │
└──────────┬───────────────────┘
           │
           │ Para batch:
           │ retorna job_id
           │
           ▼
┌──────────────────────────────┐
│   Frontend                   │
│   - Preview (single)         │
│   - Progress bar (batch)     │
│   - Resultados con éxito     │
└──────────────────────────────┘
```

---

### 5.2 Middleware de Autenticación

**Archivo:** `/src/server/trpc.ts`

```typescript
publicProcedure
  ├─ Sin autenticación
  └─ Ejemplo: listStaticPages()

protectedProcedure
  ├─ Usuario autenticado
  └─ Ejemplo: obtener datos personales

adminProcedure
  ├─ Admin o Comercial
  └─ Ejemplo: actualizar SEO

fullAdminProcedure
  ├─ Admin completo (excluye COMERCIAL)
  └─ Ejemplo: borrar entidades

superAdminProcedure
  ├─ Super admin únicamente
  └─ Ejemplo: cambiar permisos
```

---

## 6. VARIABLES DE ENTORNO REQUERIDAS

### 6.1 SEO Agent (.env)

```bash
# Base de Datos
DATABASE_URL=postgresql://user:password@localhost:5432/luckyled

# OpenRouter API
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx

# Configuración
SEO_DEFAULT_MODEL=anthropic/claude-3.5-haiku
SEO_DEFAULT_TEMPERATURE=0.7
SEO_BATCH_SIZE=50
SEO_PARALLEL_REQUESTS=5
SEO_MAX_RETRIES=3

# Host y puerto
SEO_HOST=0.0.0.0
SEO_PORT=8001
```

### 6.2 Next.js (.env.local)

```bash
# Base de Datos
DATABASE_URL=postgresql://user:password@localhost:5432/luckyled

# Autenticación
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=xxxxxxxxxxxxxxxx

# Redis
UPSTASH_REDIS_REST_URL=https://xxxxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=xxxxxxxxxxxxx

# CDN
BUNNY_STORAGE_API_KEY=xxxxxxxxxxxxxxxx
BUNNY_STORAGE_ZONE=luckyled

# Email
RESEND_API_KEY=re_xxxxxxxxxxxxxxxx

# APIs Externas
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx
```

---

## 7. CAPACIDADES Y LIMITACIONES

### 7.1 ✅ Capacidades

- Generación de metadatos SEO para 3 tipos de entidades
- Procesamiento batch asincrónico con tracking en tiempo real
- Soporte para múltiples modelos IA (4+ opciones)
- Almacenamiento automático en PostgreSQL
- Preview sin guardar cambios
- Estadísticas en tiempo real
- Validación automática de longitud de caracteres
- Manejo de retries exponencial
- Caché dinámico de modelos disponibles
- CORS habilitado para comunicación Next.js ↔ Python

### 7.2 ❌ Limitaciones

- Solo genera contenido en español
- Validación SEO limitada (solo longitudes de texto)
- No genera slugs automáticamente en el generador
- No hay soporte para A/B testing de SEO
- Caché en memoria (no persistente entre reinicios)
- No hay integración con herramientas de SEO externas (SemRush, Ahrefs, etc.)

---

## 8. STACK TECNOLÓGICO

### Frontend
- **Next.js 15** + React 19
- **TypeScript** para type safety
- **Tailwind CSS** + shadcn/ui
- **Zustand** para state management
- **TanStack Query** para data fetching
- **tRPC** para API type-safe

### Backend
- **Node.js / Next.js** (API routes, tRPC)
- **Prisma ORM** para database abstraction
- **PostgreSQL** como database principal
- **Lucia Auth** para autenticación JWT
- **FastAPI** (Python microservicio SEO)
- **SQLAlchemy** (ORM Python)
- **Pydantic** para validación

### Servicios Externos
- **OpenRouter** - Acceso a múltiples LLMs
- **Bunny.net** - CDN para imágenes
- **Upstash Redis** - Cache distribuido
- **Resend** - Servicio de email

### DevOps
- **Docker** - Containerización
- **PostgreSQL** - Database (producción)
- **GitHub Actions** - CI/CD

---

## 9. PUERTOS Y SERVICIOS

```
Frontend (Next.js)      : http://localhost:3000
SEO Agent API (FastAPI) : http://localhost:8001
PostgreSQL              : localhost:5432
Redis (Upstash)         : https://xxxxx.upstash.io (cloud)
```

---

## 10. REFERENCIAS DOCUMENTALES

- `/seo-agent/README.md` - Guía completa del SEO Agent
- `/docs/` - Documentación del proyecto (24+ archivos)
- `/CLAUDE.md` - Instrucciones y configuración
- `/CHANGELOG.md` - Historial de cambios y versiones
- `/prisma/schema.prisma` - Esquema completo de base de datos

---

**Última actualización:** Febrero 2025
**Versión del proyecto:** v0.3.5+
