# Kampaoh Influencer Dashboard — Especificación Técnica

---

## Índice

1. [Arquitectura del sistema](#1-arquitectura-del-sistema)
2. [Procesamiento del Excel](#2-procesamiento-del-excel)
3. [Diseño del dashboard](#3-diseño-del-dashboard)
4. [Wireframe / Pseudo-UI](#4-wireframe--pseudo-ui)
5. [Lógica del botón "Briefing"](#5-lógica-del-botón-briefing)
6. [Lógica del botón "Mail Camping"](#6-lógica-del-botón-mail-camping)
7. [Automatización de emails](#7-automatización-de-emails)
8. [Inserción de `{{link_notion}}` en el email](#8-inserción-de-link_notion-en-el-email)
9. [Ejemplos reales de emails generados](#9-ejemplos-reales-de-emails-generados)
10. [Gestión futura de Notion](#10-gestión-futura-de-notion)
11. [Integraciones con Make, Zapier y APIs](#11-integraciones-con-make-zapier-y-apis)
12. [Sugerencias tecnológicas](#12-sugerencias-tecnológicas)
13. [Reglas de validación y errores](#13-reglas-de-validación-y-errores)

---

## 1. Arquitectura del sistema

### Visión general

```
┌─────────────────────────────────────────────────────────────┐
│                    FUENTE DE DATOS                          │
│              Excel (.xlsx) con 3 hojas activas              │
│        "BBDD 2026" · "Colaboradores 2026" · "Campings"      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  CAPA DE PROCESAMIENTO                      │
│   SheetJS (frontend) o Node.js + xlsx (backend opcional)    │
│   · Lectura y parseo del Excel                              │
│   · Filtrado por status "Completada"                        │
│   · Unificación de hojas de influencers                     │
│   · Normalización de destinos                               │
│   · Match con hoja "Campings"                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    ESTADO DE LA APP                         │
│            Zustand / Context API (React)                    │
│   · Lista de influencers procesados                         │
│   · Estado de cada colaboración                             │
│   · Links de Notion por influencer                          │
│   · Filtros activos                                         │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   CAPA DE PRESENTACIÓN                      │
│          React + TypeScript + Tailwind CSS + Shadcn/UI      │
│   · Dashboard principal con filtros                         │
│   · Tabla/cards de influencers                              │
│   · Panel de detalle                                        │
│   · Generador de emails                                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  CAPA DE ACCIÓN                             │
│   · Generación de emails (mailto / Gmail API / Brevo)       │
│   · Inserción manual de links Notion                        │
│   · Exportación de datos                                    │
│   · (Futuro) Integración Make / Zapier / Notion API         │
└─────────────────────────────────────────────────────────────┘
```

### Stack tecnológico

| Capa | Tecnología | Justificación |
|---|---|---|
| Framework UI | React + TypeScript | Escalable, tipado fuerte, ecosistema maduro |
| Estilos | Tailwind CSS | Utilidades atómicas, fácil de mantener |
| Componentes | Shadcn/UI | Accesibles, sin dependencia de librería cerrada |
| Tablas | TanStack Table v8 | Filtrado, ordenación y paginación eficiente |
| Iconos | Lucide React | Línea limpia, coherente con estética Notion |
| Formularios | React Hook Form | Validación sin overhead |
| Estado global | Zustand | Ligero, sin boilerplate de Redux |
| Procesamiento Excel | SheetJS (xlsx) | Lee `.xlsx` directamente en el navegador |
| Emails (v1) | `mailto:` links | Sin backend, cero configuración |
| Emails (v2) | Gmail API / Brevo | Para borradores automáticos y trazabilidad |

---

## 2. Procesamiento del Excel

### Mapa de columnas

#### Hojas "BBDD 2026" y "Colaboradores 2026"

| Columna | Campo | Variable | Notas |
|---|---|---|---|
| A | Username Instagram | `{{username}}` | Obligatorio |
| B | Nombre real | `{{nombre}}` | Opcional |
| C | — | — | Ignorar |
| D | — | — | Ignorar |
| E | Status | — | Filtrar solo `"Completada"` (exacto) |
| F | — | — | Ignorar |
| G | Fecha de visita | `{{fecha}}` | Formatear como `DD/MM/YYYY` |
| H | Destino | `{{destino}}` | Usar para match con campings |
| I | Email influencer | `{{email}}` | Opcional, pero crítico para envío |
| J | Tipología | `{{tipologia}}` | Ej: "Bell XL", "Bungalow", etc. |

#### Hoja "Campings"

| Columna | Campo | Notas |
|---|---|---|
| A | Nombre del camping | Usar para match con `{{destino}}` |
| D | Email del camping | Puede estar vacío |

### Algoritmo de procesamiento

```typescript
import * as XLSX from 'xlsx';

interface Influencer {
  username: string;
  nombre: string;
  fecha: string;
  destino: string;
  email: string;
  tipologia: string;
  emailCamping: string | null;
  matchCampingConfianza: 'exacto' | 'parcial' | 'dudoso' | 'no_encontrado';
  linkNotion: string;
  estado: EstadoColaboracion;
}

type EstadoColaboracion =
  | 'lista_para_briefing'
  | 'falta_link_notion'
  | 'briefing_generado'
  | 'camping_con_email'
  | 'camping_sin_email'
  | 'pendiente_revision'
  | 'email_enviado'
  | 'error_datos';

function procesarExcel(file: File): Influencer[] {
  const workbook = XLSX.read(await file.arrayBuffer());

  // 1. Leer ambas hojas de influencers
  const hojaBBDD = XLSX.utils.sheet_to_json(workbook.Sheets['BBDD 2026'], { header: 1 });
  const hojaColab = XLSX.utils.sheet_to_json(workbook.Sheets['Colaboradores 2026'], { header: 1 });
  const hojaCampings = XLSX.utils.sheet_to_json(workbook.Sheets['Campings'], { header: 1 });

  // 2. Unificar registros
  const todasLasFilas = [...hojaBBDD, ...hojaColab];

  // 3. Parsear campings
  const campings = parsearCampings(hojaCampings);

  // 4. Filtrar y mapear influencers
  return todasLasFilas
    .filter(fila => fila[4] === 'Completada')  // Columna E (índice 4)
    .filter(fila => fila[0])                    // Username no vacío
    .map(fila => {
      const destino = normalizarDestino(fila[7]);
      const matchCamping = buscarCamping(destino, campings);

      return {
        username: String(fila[0] ?? '').trim(),
        nombre: String(fila[1] ?? '').trim(),
        fecha: formatearFecha(fila[6]),
        destino: String(fila[7] ?? '').trim(),
        email: String(fila[8] ?? '').trim(),
        tipologia: String(fila[9] ?? '').trim(),
        emailCamping: matchCamping?.email ?? null,
        matchCampingConfianza: matchCamping?.confianza ?? 'no_encontrado',
        linkNotion: '',
        estado: determinarEstado(fila, matchCamping),
      };
    });
}
```

### Normalización de destinos y match con campings

```typescript
function normalizarDestino(valor: unknown): string {
  return String(valor ?? '')
    .toLowerCase()
    .normalize('NFD')
    .replace(/[̀-ͯ]/g, '') // quitar tildes
    .replace(/\s+/g, ' ')
    .trim();
}

function buscarCamping(destino: string, campings: Camping[]) {
  // 1. Coincidencia exacta
  const exacto = campings.find(c => normalizarDestino(c.nombre) === destino);
  if (exacto) return { ...exacto, confianza: 'exacto' };

  // 2. Coincidencia parcial (destino contenido en nombre del camping o viceversa)
  const parcial = campings.find(c => {
    const nombreNorm = normalizarDestino(c.nombre);
    return nombreNorm.includes(destino) || destino.includes(nombreNorm);
  });
  if (parcial) return { ...parcial, confianza: 'parcial' };

  // 3. Sin coincidencia — marcar para revisión manual
  return null;
}
```

---

## 3. Diseño del dashboard

### Sistema de diseño

#### Tokens de color

```css
--color-bg:          #FAFAFA;   /* fondo general */
--color-surface:     #FFFFFF;   /* cards y paneles */
--color-border:      #E5E7EB;   /* bordes suaves */
--color-text-main:   #111827;   /* títulos */
--color-text-muted:  #6B7280;   /* metadatos */
--color-accent:      #2563EB;   /* azul acción primaria */
--color-success:     #16A34A;   /* email OK, completado */
--color-warning:     #D97706;   /* datos dudosos */
--color-danger:      #DC2626;   /* error, sin email */
--color-neutral:     #9CA3AF;   /* pendiente */
```

#### Tipografía

```css
font-family: 'Inter', system-ui, sans-serif;
--text-xs:   11px / 1.4;
--text-sm:   13px / 1.5;
--text-base: 15px / 1.6;
--text-lg:   18px / 1.4;
--text-xl:   22px / 1.3;
```

#### Espaciado y radios

```css
--radius-sm:  8px;
--radius-md:  12px;
--radius-lg:  16px;   /* cards */
--radius-xl:  20px;   /* panel detalle */
--shadow-sm:  0 1px 3px rgba(0,0,0,0.06);
--shadow-md:  0 4px 16px rgba(0,0,0,0.08);
```

### Componentes

#### 1. Header superior

```
┌─────────────────────────────────────────────────────────────┐
│  ⛺ Kampaoh · Influencer Hub                    [Cargar Excel]│
│                                                              │
│  42 completadas    8 sin briefing    5 campings sin email    │
└─────────────────────────────────────────────────────────────┘
```

Contadores con iconos Lucide:
- `CheckCircle2` — colaboraciones completadas
- `Clock` — briefings pendientes (falta link Notion)
- `AlertTriangle` — campings sin email

#### 2. Barra de filtros

```
┌──────────────┬──────────────┬──────────────┬────────────────┐
│ 🔍 Buscar…   │ Destino ▾    │ Fecha ▾      │ Estado email ▾ │
└──────────────┴──────────────┴──────────────┴────────────────┘
                                              [Ordenar: Fecha ▾]
```

Filtros disponibles:
- Búsqueda libre: username o nombre real
- Selector de destino (todos los destinos únicos del Excel)
- Rango de fechas o mes concreto
- Estado de email: todos / con email / sin email / enviado
- Ordenación: fecha visita asc/desc, username A-Z

#### 3. Tabla de influencers

Columnas visibles por defecto:

| Username | Nombre | Destino | Fecha | Tipología | Email influencer | Email camping | Acciones |
|---|---|---|---|---|---|---|---|
| @laura_viaja | Laura M. | Cádiz | 14/06/26 | Bell XL | ✓ laura@… | ✓ cadiz@… | [Briefing] [Mail Camping] [···] |
| @surf_marta | Marta G. | Tarifa | 20/06/26 | Bungalow | ✗ Sin email | ⚠ Sin email camping | [Briefing] [···] |

Indicadores visuales de estado por fila (badge de color):
- `Lista para briefing` — azul
- `Falta link Notion` — naranja
- `Email enviado` — verde
- `Camping sin email` — rojo suave
- `Pendiente revisión` — gris

#### 4. Panel de detalle (drawer lateral)

Se abre al clicar en una fila o en `[···]`.

```
┌────────────────────────────────────────┐
│  @laura_viaja                    [✕]   │
│  Laura Martínez · Bell XL · Cádiz      │
│  Visita: 14/06/2026                    │
│                                        │
│  Email influencer: laura@gmail.com     │
│  Email camping:    cadiz@kampaoh.com   │
│                                        │
│  Link Notion:  [___________________]  │
│                [Guardar link]          │
│                                        │
│  ── Preview email influencer ──────── │
│  Asunto: Colaboración Kampaoh Cádiz ⛺ │
│  ...cuerpo del email...                │
│                                        │
│  ── Preview email camping ──────────── │
│  Asunto: Colaboración Kampaoh Cádiz ⛺ │
│  ...cuerpo del email...                │
│                                        │
│  [Enviar briefing]  [Enviar a camping] │
└────────────────────────────────────────┘
```

#### 5. Footer / barra de estado

```
┌─────────────────────────────────────────────────────────────┐
│  Sistema activo · Última carga: 29/04/2026 10:32           │
│  84 registros procesados · 42 completados · 42 ignorados   │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Wireframe / Pseudo-UI

```
╔══════════════════════════════════════════════════════════════╗
║  ⛺ Kampaoh · Influencer Hub                  [Cargar Excel] ║
║                                                              ║
║   ┌──────────┐   ┌─────────────────┐   ┌──────────────┐    ║
║   │    42    │   │        8        │   │      5       │    ║
║   │Completad.│   │Sin briefing     │   │Sin email     │    ║
║   └──────────┘   └─────────────────┘   └──────────────┘    ║
╠══════════════════════════════════════════════════════════════╣
║  🔍 Buscar username o nombre...                              ║
║  [Destino ▾]  [Fecha ▾]  [Estado email ▾]  [Ordenar ▾]      ║
╠══════════════════════════════════════════════════════════════╣
║  USERNAME         NOMBRE       DESTINO  FECHA   ESTADO      ║
║  ─────────────────────────────────────────────────────────  ║
║  @laura_viaja     Laura M.     Cádiz    14/06   🔵 Lista    ║
║  ✉ laura@…        ⛺ cadiz@…              [Briefing][Mail ⛺] ║
║                                                              ║
║  @surf_marta      Marta G.     Tarifa   20/06   🟠 Falta    ║
║  ✗ Sin email      ⚠ Sin email                  [Briefing]   ║
║                                                              ║
║  @nomad_pablo     Pablo R.     Zahara   05/07   🔴 Sin email ║
║  ✉ pablo@…        ✗ Sin email camping          [Briefing]   ║
║                                                              ║
╠══════════════════════════════════════════════════════════════╣
║  Sistema activo · 29/04/2026 10:32 · 84 registros · 42 OK  ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 5. Lógica del botón "Briefing"

### Flujo completo

```
[Usuario clica "Briefing"]
         │
         ▼
¿Existe {{link_notion}}?
         │
    NO ──┤── SÍ
         │         │
         ▼         ▼
  Mostrar aviso  ¿Existe {{email}} del influencer?
  "Añade el link         │
   de Notion antes       │
   de enviar"       NO ──┤── SÍ
                         │         │
                         ▼         ▼
                   Mostrar aviso  Generar email
                   "Este influencer  con plantilla
                    no tiene email"
                                   │
                                   ▼
                             Abrir mailto:
                             o crear borrador
                             en Gmail API
                                   │
                                   ▼
                            Marcar estado:
                           "Briefing generado"
```

### Plantilla del email al influencer

**Asunto:**
```
Colaboración Kampaoh {{destino}} ⛺
```

**Cuerpo:**
```html
Buenos días, {{nombre}}:

Soy Andrea, del departamento de marketing de Kampaoh. ¡Se va acercando la 
fecha de vuestra llegada a {{destino}}! Como sabéis, tenéis una reserva para 
el {{fecha}} en Kampaoh en nuestra {{tipologia}}.

Comparto contigo en <a href="{{link_notion}}">este enlace</a> los aspectos 
a tener en cuenta acerca de nuestra colaboración. Solo tienes que hacer clic 
sobre los triángulos para desplegar la información. Si después de revisarlo 
tienes alguna duda, me dices sin problema.

Estoy a vuestra disposición para lo que necesitéis. Espero que disfrutéis 
mucho en Kampaoh {{destino}} ⛺❤

Un abrazo,
Andrea
Kampaoh · Departamento de Marketing
```

### Implementación del generador

```typescript
function generarEmailBriefing(influencer: Influencer): EmailGenerado {
  if (!influencer.linkNotion) {
    throw new Error('FALTA_LINK_NOTION');
  }
  if (!influencer.email) {
    throw new Error('FALTA_EMAIL_INFLUENCER');
  }

  const nombre = influencer.nombre || influencer.username;

  const asunto = `Colaboración Kampaoh ${influencer.destino} ⛺`;

  const cuerpoHTML = `
    <p>Buenos días, ${nombre}:</p>
    <p>Soy Andrea, del departamento de marketing de Kampaoh. ¡Se va acercando la 
    fecha de vuestra llegada a ${influencer.destino}! Como sabéis, tenéis una reserva 
    para el ${influencer.fecha} en Kampaoh en nuestra ${influencer.tipologia}.</p>
    <p>Comparto contigo en <a href="${influencer.linkNotion}">este enlace</a> los 
    aspectos a tener en cuenta acerca de nuestra colaboración. Solo tienes que hacer 
    clic sobre los triángulos para desplegar la información. Si después de revisarlo 
    tienes alguna duda, me dices sin problema.</p>
    <p>Estoy a vuestra disposición para lo que necesitéis. Espero que disfrutéis 
    mucho en Kampaoh ${influencer.destino} ⛺❤</p>
    <p>Un abrazo,<br>Andrea<br>Kampaoh · Departamento de Marketing</p>
  `.trim();

  return { destinatario: influencer.email, asunto, cuerpoHTML };
}
```

---

## 6. Lógica del botón "Mail Camping"

### Flujo completo

```
[Usuario clica "Mail Camping"]
         │
         ▼
Buscar camping por {{destino}}
(normalizar y comparar con hoja "Campings" col. A)
         │
         ├── No encontrado ──▶ Aviso: "Camping no identificado"
         │
         ├── Encontrado, sin email (col. D vacía)
         │        └──▶ Estado: "No se puede enviar: camping sin email"
         │
         └── Encontrado, con email ──▶ Generar email al camping
                                              │
                                              ▼
                                        Abrir mailto:
                                        o borrador Gmail
                                              │
                                              ▼
                                       Marcar estado:
                                      "Email camping enviado"
```

### Plantilla del email al camping

**Asunto:**
```
Colaboración Kampaoh {{destino}} ⛺
```

**Cuerpo:**
```
Buenos días,

¿Qué tal? Os informo de que próximamente tenemos la entrada de esta influencer. 
Podéis consultar todos los datos de la reserva en el localizador.

Estoy a vuestra disposición para lo que necesitéis.

Un saludo,
Andrea
Kampaoh · Departamento de Marketing
```

### Implementación

```typescript
function generarEmailCamping(influencer: Influencer, campings: Camping[]): EmailGenerado {
  const matchCamping = buscarCamping(
    normalizarDestino(influencer.destino),
    campings
  );

  if (!matchCamping) {
    throw new Error('CAMPING_NO_IDENTIFICADO');
  }
  if (!matchCamping.email) {
    throw new Error('CAMPING_SIN_EMAIL');
  }

  const asunto = `Colaboración Kampaoh ${influencer.destino} ⛺`;

  const cuerpo = `
Buenos días,

¿Qué tal? Os informo de que próximamente tenemos la entrada de esta influencer. 
Podéis consultar todos los datos de la reserva en el localizador.

Estoy a vuestra disposición para lo que necesitéis.

Un saludo,
Andrea
Kampaoh · Departamento de Marketing
  `.trim();

  return { destinatario: matchCamping.email, asunto, cuerpo };
}
```

---

## 7. Automatización de emails

### Nivel 1 — `mailto:` (sin backend, implementación inmediata)

```typescript
function abrirMailto(email: EmailGenerado): void {
  const params = new URLSearchParams({
    subject: email.asunto,
    body: email.cuerpo,
  });
  window.open(`mailto:${email.destinatario}?${params.toString()}`);
}
```

Ventajas: cero configuración, funciona desde el primer día.
Limitación: no registra envíos, no permite HTML en el cuerpo.

### Nivel 2 — Gmail API (borradores automáticos)

```typescript
// Requiere OAuth 2.0 con scope gmail.compose
async function crearBorradorGmail(email: EmailGenerado, token: string) {
  const mensaje = crearMensajeMIME(email);
  await fetch('https://gmail.googleapis.com/gmail/v1/users/me/drafts', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      message: { raw: btoa(mensaje) },
    }),
  });
}
```

Ventajas: el borrador aparece en Gmail de Andrea, HTML completo, trazabilidad.

### Nivel 3 — Brevo / Mailgun / SendGrid (envío masivo futuro)

Para volumenes altos. Requiere backend o Make/Zapier como intermediario.

---

## 8. Inserción de `{{link_notion}}` en el email

### Regla fundamental

> El texto visible es siempre `"este enlace"`. La URL nunca aparece como texto plano.

### Correcto

```html
Comparto contigo en <a href="https://notion.so/briefing-abc123">este enlace</a> los aspectos...
```

### Incorrecto — nunca hacer esto

```
<!-- INCORRECTO: URL visible -->
Comparto contigo este enlace: https://notion.so/briefing-abc123

<!-- INCORRECTO: URL duplicada -->
este enlace (https://notion.so/briefing-abc123) los aspectos...
```

### Validación antes de envío

```typescript
function validarLinkNotion(link: string): ValidationResult {
  if (!link || link.trim() === '') {
    return { valido: false, error: 'El link de Notion es obligatorio para enviar el briefing.' };
  }
  try {
    new URL(link); // valida que sea una URL bien formada
    return { valido: true };
  } catch {
    return { valido: false, error: 'El link de Notion no parece una URL válida.' };
  }
}
```

### Gestión del campo en la UI

- Campo de texto en el panel de detalle de cada influencer
- Indicador visual: `⚠ Falta link Notion` (naranja) / `✓ Link añadido` (verde)
- El botón "Enviar briefing" permanece deshabilitado hasta que el link esté validado
- El link se persiste en el estado local (Zustand) y, en futuras versiones, en base de datos

---

## 9. Ejemplos reales de emails generados

### Email al influencer — ejemplo completo

**Destinatario:** `laura@gmail.com`  
**Asunto:** `Colaboración Kampaoh Cádiz ⛺`

```html
Buenos días, Laura:

Soy Andrea, del departamento de marketing de Kampaoh. ¡Se va acercando la 
fecha de vuestra llegada a Cádiz! Como sabéis, tenéis una reserva para el 
14/06/2026 en Kampaoh en nuestra Bell XL.

Comparto contigo en <a href="https://notion.so/briefing-cadiz-laura">este enlace</a> 
los aspectos a tener en cuenta acerca de nuestra colaboración. Solo tienes que 
hacer clic sobre los triángulos para desplegar la información. Si después de 
revisarlo tienes alguna duda, me dices sin problema.

Estoy a vuestra disposición para lo que necesitéis. Espero que disfrutéis 
mucho en Kampaoh Cádiz ⛺❤

Un abrazo,
Andrea
Kampaoh · Departamento de Marketing
```

### Email al camping — ejemplo completo

**Destinatario:** `cadiz@kampaoh.com`  
**Asunto:** `Colaboración Kampaoh Cádiz ⛺`

```
Buenos días,

¿Qué tal? Os informo de que próximamente tenemos la entrada de esta influencer. 
Podéis consultar todos los datos de la reserva en el localizador.

Estoy a vuestra disposición para lo que necesitéis.

Un saludo,
Andrea
Kampaoh · Departamento de Marketing
```

---

## 10. Gestión futura de Notion

### Estado actual (v1)

- Campo manual para pegar el link de Notion en el panel de detalle
- Validación de URL antes de permitir envío
- Persistencia local en Zustand (se pierde al recargar — aceptable en v1)

### Estado intermedio (v2) — persistencia real

Opciones sin backend propio:

| Opción | Cómo | Coste |
|---|---|---|
| LocalStorage | Guardar links por username en el navegador | Gratis, pero solo en ese dispositivo |
| Google Sheets | Hoja extra con username + link Notion | Gratis, requiere OAuth |
| Airtable | Base de datos simple en la nube | Plan gratuito disponible |
| Supabase | Postgres gestionado con API REST | Plan gratuito disponible |

### Estado objetivo (v3) — integración real con Notion API

```
Flujo automatizado:
1. Andrea sube el Excel actualizado
2. El sistema detecta nuevos influencers con status "Completada"
3. Make/Zapier llama a Notion API → duplica plantilla de briefing
4. La nueva página de Notion se vincula automáticamente al influencer
5. El link queda disponible sin intervención manual
6. Andrea solo revisa y envía
```

**Notion API — operaciones necesarias:**
- `POST /v1/pages` — crear página desde plantilla (duplicar bloque)
- `GET /v1/databases/{id}/query` — listar briefings existentes
- `PATCH /v1/pages/{id}` — actualizar propiedades (fecha, destino, etc.)

**Plantilla de Notion recomendada:**
- Base de datos con campos: Influencer, Destino, Fecha, Tipología, URL pública
- Cada registro = una página de briefing
- Página publicada con `public_url` para compartir sin que el influencer necesite cuenta Notion

---

## 11. Integraciones con Make, Zapier y APIs

### Flujo Make/Zapier recomendado

```
┌─────────────────────────────────────────────────────────────┐
│ TRIGGER: Google Sheets actualizado (nueva fila "Completada") │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PASO 1: Filtrar                                              │
│ · status === "Completada"                                    │
│ · username no vacío                                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PASO 2: Buscar camping en hoja "Campings"                    │
│ · Match por nombre de destino                               │
│ · Extraer email del camping                                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PASO 3: Crear página en Notion                               │
│ · Duplicar plantilla de briefing                            │
│ · Rellenar campos: influencer, destino, fecha, tipología    │
│ · Obtener URL pública de la página                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PASO 4: Guardar link Notion en Google Sheets / Airtable      │
│ · Asociado al username del influencer                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PASO 5: Crear borrador de email en Gmail                     │
│ · Email al influencer con link Notion incrustado             │
│ · Email al camping (si tiene email)                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ PASO 6: Notificar a Andrea                                   │
│ · Email / Slack / notificación en el dashboard              │
│ · "Tienes X borradores listos para revisar"                 │
└─────────────────────────────────────────────────────────────┘
```

### APIs relevantes

| Servicio | Uso | Autenticación |
|---|---|---|
| Google Sheets API v4 | Fuente de datos alternativa al Excel | OAuth 2.0 |
| Notion API | Crear y gestionar briefings | Integration token |
| Gmail API v1 | Crear borradores automáticos | OAuth 2.0 |
| Brevo API | Envío masivo con trazabilidad | API key |
| Make (Integromat) | Orquestación de todos los pasos | Web hooks |

---

## 12. Sugerencias tecnológicas

### Para empezar hoy (sin backend)

```bash
# Crear proyecto con Vite + React + TypeScript
npm create vite@latest kampaoh-influencer-hub -- --template react-ts
cd kampaoh-influencer-hub

# Instalar dependencias principales
npm install \
  tailwindcss @tailwindcss/vite \
  lucide-react \
  xlsx \
  zustand \
  @tanstack/react-table \
  react-hook-form \
  @hookform/resolvers \
  zod

# Shadcn UI (componentes accesibles)
npx shadcn@latest init
npx shadcn@latest add button card input select badge drawer table
```

### Estructura de carpetas recomendada

```
src/
├── components/
│   ├── dashboard/
│   │   ├── Header.tsx
│   │   ├── FilterBar.tsx
│   │   ├── InfluencerTable.tsx
│   │   ├── InfluencerCard.tsx
│   │   ├── DetailPanel.tsx
│   │   └── Footer.tsx
│   └── ui/               ← componentes Shadcn
├── lib/
│   ├── excel.ts          ← procesamiento SheetJS
│   ├── emailTemplates.ts ← generación de emails
│   ├── campingMatcher.ts ← lógica de matching
│   └── validators.ts     ← validaciones
├── store/
│   └── influencerStore.ts ← Zustand
├── types/
│   └── index.ts          ← interfaces TypeScript
└── App.tsx
```

### Decisiones de arquitectura clave

| Decisión | Opción elegida | Alternativa descartada | Por qué |
|---|---|---|---|
| Procesamiento Excel | Frontend (SheetJS) | Backend Node.js | No requiere infraestructura, Andrea puede usar desde cualquier navegador |
| Estado global | Zustand | Redux Toolkit | Menos boilerplate, suficiente para esta escala |
| Tablas | TanStack Table | Ag-Grid | Open source, integración natural con React |
| Emails v1 | mailto: | Gmail API | Funcionamiento inmediato sin OAuth |
| Persistencia v1 | localStorage | Base de datos | Sin backend en v1, migrar en v2 |

---

## 13. Reglas de validación y errores

### Validaciones al procesar el Excel

| Condición | Acción |
|---|---|
| Status !== "Completada" | Ignorar fila silenciosamente |
| Username (col. A) vacío | Ignorar fila, registrar en log |
| Fila completamente vacía | Ignorar sin registrar |
| Destino no encontrado en campings | Marcar como `pendiente_revision` |
| Match de camping dudoso (parcial) | Marcar como `pendiente_revision` con indicador visual |
| Fecha con formato inválido | Mostrar fila con aviso, no bloquear |

### Validaciones antes de enviar email al influencer

| Condición | Comportamiento |
|---|---|
| `linkNotion` vacío | Botón deshabilitado + aviso "Añade el link de Notion" |
| `linkNotion` no es URL válida | Aviso inline en el campo + botón deshabilitado |
| `email` influencer vacío | Botón deshabilitado + estado "Sin email de contacto" |
| `nombre` vacío | Usar `username` como fallback en el saludo |

### Validaciones antes de enviar email al camping

| Condición | Comportamiento |
|---|---|
| Camping no encontrado | Estado "Camping no identificado" — botón deshabilitado |
| Camping encontrado, sin email | Estado "No se puede enviar: camping sin email" |
| Match dudoso | Aviso "Verifica el camping antes de enviar" + botón habilitado con confirmación |

### Mensajes de error en la UI

```typescript
const MENSAJES_ERROR = {
  FALTA_LINK_NOTION: 'Añade el link de Notion antes de enviar el briefing.',
  FALTA_EMAIL_INFLUENCER: 'Este influencer no tiene email registrado.',
  CAMPING_NO_IDENTIFICADO: 'No se encontró un camping para este destino. Verifica manualmente.',
  CAMPING_SIN_EMAIL: 'El camping no tiene email registrado en el sistema.',
  LINK_NOTION_INVALIDO: 'El link de Notion no es una URL válida.',
  EXCEL_HOJA_NO_ENCONTRADA: 'No se encontró la hoja "{nombre}" en el archivo Excel.',
} as const;
```

### Estados del sistema por colaboración

| Estado | Color | Descripción |
|---|---|---|
| `lista_para_briefing` | Azul | Tiene email, link Notion y camping con email |
| `falta_link_notion` | Naranja | Falta el link de Notion para poder enviar |
| `briefing_generado` | Verde claro | El email de briefing ha sido generado/enviado |
| `camping_con_email` | Verde | El camping tiene email registrado |
| `camping_sin_email` | Rojo suave | El camping no tiene email — solo briefing posible |
| `pendiente_revision` | Gris | Match de camping dudoso o datos incompletos |
| `email_enviado` | Verde oscuro | Ambos emails confirmados como enviados |
| `error_datos` | Rojo | Datos insuficientes para procesar la colaboración |

---

*Documento generado para uso interno de Kampaoh · Departamento de Marketing*  
*Versión 1.0 · Abril 2026*
