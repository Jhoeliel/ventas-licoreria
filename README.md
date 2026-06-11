# D'Cafiro — App de Gestión de Ventas

Aplicación web de gestión de ventas para la licorería **D'Cafiro** (Chimbote, Perú). Permite registrar ventas, gestionar inventario, consultar reportes de rentabilidad y mantener el catálogo de productos, todo desde el navegador sin instalación alguna.

---

## Tabla de contenidos

1. [Arquitectura general](#arquitectura-general)
2. [Stack tecnológico](#stack-tecnológico)
3. [Estructura del proyecto](#estructura-del-proyecto)
4. [Configuración de Google Cloud](#configuración-de-google-cloud)
5. [Estructura del Google Spreadsheet](#estructura-del-google-spreadsheet)
6. [Sistema de roles y usuarios](#sistema-de-roles-y-usuarios)
7. [Funcionalidades por módulo](#funcionalidades-por-módulo)
8. [Catálogo de productos](#catálogo-de-productos)
9. [Cálculo de stock proyectado](#cálculo-de-stock-proyectado)
10. [Lógica de paquetes y vínculos](#lógica-de-paquetes-y-vínculos)
11. [Zona horaria](#zona-horaria)
12. [Cómo replicar el proyecto](#cómo-replicar-el-proyecto)
13. [Variables y constantes clave](#variables-y-constantes-clave)
14. [Limitaciones conocidas](#limitaciones-conocidas)

---

## Arquitectura general

```
Navegador (GitHub Pages)
    │
    ├── Google Identity Services (OAuth 2.0)
    │       └── Autenticación con cuenta Google
    │
    └── Google Sheets API v4 (REST directa)
            └── Spreadsheet como base de datos
```

La app es un **único archivo HTML** (`index.html`) sin dependencias externas ni framework. Se hostea en GitHub Pages y se comunica directamente con la API REST de Google Sheets usando el token OAuth del usuario logueado. No hay backend propio ni servidor intermedio.

**Ventajas de esta arquitectura:**
- Costo cero de hosting y base de datos
- El Spreadsheet es visible y editable directamente por el negocio
- Despliegue inmediato: hacer push a GitHub = nueva versión en producción
- Sin configuración de servidores

**Limitaciones:**
- Depende de que el usuario tenga conexión a internet
- La API de Google Sheets tiene límites de cuota (100 req/100s por usuario)
- El token OAuth expira cada hora (se renueva silenciosamente)

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Frontend | HTML5 + CSS3 + JavaScript ES6 (vanilla, sin frameworks) |
| Autenticación | Google Identity Services (GSI) — OAuth 2.0 implicit flow |
| Base de datos | Google Sheets API v4 (REST) |
| Hosting | GitHub Pages |
| Despliegue | Push manual a rama `main` |

---

## Estructura del proyecto

```
ventas-licoreria/
├── index.html          # Aplicación completa (HTML + CSS + JS en un solo archivo)
├── productos.csv       # Catálogo de productos exportado (referencia/importación)
└── README.md           # Este archivo
```

Todo el código vive en `index.html`, organizado en secciones:

```
index.html
├── <head>          Variables globales, constantes OAuth
├── <style>         CSS completo con variables de diseño (tema oscuro)
├── <body>
│   ├── #login-screen       Pantalla de inicio de sesión
│   └── #app                Aplicación principal
│       ├── header          Logo, email, badge de rol, botón salir
│       ├── nav             Pestañas de navegación (visibilidad según rol)
│       └── panels          Contenido de cada pestaña
└── <script>
    ├── OAuth / sesión
    ├── Sheets API helpers (sheetsGet, sheetsAppend, sheetsUpdate)
    ├── Panel: Nueva Venta
    ├── Panel: Mis Ventas Hoy
    ├── Panel: Reporte por Fechas
    ├── Panel: Stock
    ├── Panel: Productos (mantenimiento)
    └── Panel: Config
```

---

## Configuración de Google Cloud

### 1. Crear proyecto en Google Cloud Console

1. Ve a [console.cloud.google.com](https://console.cloud.google.com)
2. Crea un nuevo proyecto (ej. `dcafiro-ventas`)
3. Habilita la **Google Sheets API**:
   - APIs y Servicios → Biblioteca → buscar "Google Sheets API" → Habilitar

### 2. Crear credenciales OAuth 2.0

1. APIs y Servicios → **Credenciales** → Crear credenciales → **ID de cliente OAuth**
2. Tipo de aplicación: **Aplicación web**
3. Nombre: el que quieras (ej. `D'Cafiro App`)
4. **Orígenes JavaScript autorizados:**
   ```
   https://TU_USUARIO.github.io
   http://localhost        ← para desarrollo local
   ```
5. **URI de redireccionamiento:** no es necesario para implicit flow
6. Copiar el **Client ID** generado

### 3. Configurar pantalla de consentimiento OAuth

1. APIs y Servicios → **Pantalla de consentimiento de OAuth**
2. Tipo de usuario: **Externo**
3. Completar nombre de app, email de soporte
4. **Scopes a agregar:**
   - `https://www.googleapis.com/auth/spreadsheets`
   - `https://www.googleapis.com/auth/userinfo.email`
   - `openid`
5. **Usuarios de prueba:** agregar los emails que usarán la app (máximo 100 en modo Testing)
6. Estado: puede quedarse en **Testing** si no se necesitan más de 100 usuarios

> **Nota:** En modo Testing, solo los emails agregados como "usuarios de prueba" pueden iniciar sesión. Esto es suficiente para un negocio pequeño y evita pasar por la revisión de Google.

### 4. Actualizar el Client ID en el código

En `index.html`, buscar y reemplazar:

```javascript
var CLIENT_ID = 'TU_CLIENT_ID.apps.googleusercontent.com';
```

---

## Estructura del Google Spreadsheet

El Spreadsheet actúa como base de datos. Debe tener exactamente estas hojas:

### Hoja: `Productos`

| Col | Campo | Tipo | Descripción |
|-----|-------|------|-------------|
| A | Código | Texto | Identificador único (ej. `RCB246`) |
| B | Nombre | Texto | Nombre completo del producto |
| C | Unidad | Texto | Unidad de medida (`NIU`, `PQT`, `SIX PACK`, `CJ`, `CJT`, etc.) |
| D | Precio | Número | Precio de venta en soles |
| E | Costo | Número | Costo de adquisición en soles |
| F | Tipo | Texto | `UNIDAD` o `PAQUETE` |
| G | Factor | Entero | Unidades por paquete (solo aplica si Tipo=PAQUETE) |
| H | Vínculo | Texto | Código del producto relacionado (ver sección de vínculos) |
| I | Estado | Texto | `ACTIVO` o `INACTIVO` |

La fila 1 debe contener los encabezados. Los productos `INACTIVO` no aparecen en Nueva Venta ni en Stock.

### Hoja: `Ventas`

| Col | Campo | Descripción |
|-----|-------|-------------|
| A | Timestamp | Fecha y hora UTC del registro (`new Date().toISOString()`) |
| B | Fecha | Fecha local de la venta (`YYYY-MM-DD`) |
| C | Hora | Hora local de la venta (`HH:MM:SS`) |
| D | Usuario | Email del operador que registró |
| E | Código | Código del producto vendido |
| F | Nombre | Nombre del producto vendido |
| G | Unidad | Unidad del producto |
| H | Cantidad | Cantidad vendida |
| I | Importe | Monto real cobrado (después de descuento) |
| J | Ref.Calculado | Subtotal sin descuento (cantidad × precio unitario) |
| K | Estado | `VENTA` o `ANULACION` |
| L | Ref.Anulacion | Timestamp de la venta original (si es anulación) |
| M | Motivo | Motivo de anulación (si aplica) |
| N | Descuento | Monto descontado en soles (0 si no hubo descuento) |

### Hoja: `Stock`

| Col | Campo | Descripción |
|-----|-------|-------------|
| A | Timestamp | Fecha y hora UTC del movimiento |
| B | Fecha | Fecha local |
| C | Hora | Hora local |
| D | Usuario | Email del administrador |
| E | Código | Código del producto |
| F | Nombre | Nombre del producto |
| G | Unidad | Unidad |
| H | Cantidad | Cantidad del movimiento |
| I | Tipo | `ENTRADA`, `SALIDA` o `AJUSTE` |
| J | Motivo | Descripción del movimiento |

### Hoja: `Usuarios`

| Col | Campo | Descripción |
|-----|-------|-------------|
| A | Email | Email de la cuenta Google del usuario |
| B | Rol | `ADMIN` o `OPERADOR` |

La fila 1 debe tener encabezados. Si un email no aparece en esta hoja, se le asigna `OPERADOR` por defecto.

### Hoja: `Config`

| A (clave) | B (valor) | Descripción |
|-----------|-----------|-------------|
| `hoja_productos` | `Productos` | Nombre de la hoja de productos |
| `hoja_ventas` | `Ventas` | Nombre de la hoja de ventas |
| `hoja_stock` | `Stock` | Nombre de la hoja de stock |

Esta hoja permite cambiar los nombres de las demás hojas sin tocar el código.

---

## Sistema de roles y usuarios

La app tiene dos roles:

| Rol | Pestañas accesibles |
|-----|-------------------|
| `ADMIN` | Nueva Venta, Mis Ventas Hoy, Reporte por Fechas, Stock, Productos, Config |
| `OPERADOR` | Nueva Venta, Mis Ventas Hoy |

**Flujo de autenticación y rol:**

```
1. Usuario hace clic en "Iniciar sesión con Google"
2. Google Identity Services devuelve un access_token + id_token
3. El email se extrae del id_token (JWT decode, sin llamada extra a userinfo)
4. El email y token se guardan en sessionStorage y localStorage
5. La app lee la hoja Usuarios para determinar el rol
6. Se muestran/ocultan las pestañas según el rol
7. El token se renueva silenciosamente antes de expirar (cada ~55 min)
```

**Persistencia de sesión:**
- El token y email se guardan en `sessionStorage` (por pestaña) y el email en `localStorage` (persistente entre sesiones)
- Al recargar la página, si hay token y email válidos, la app entra directamente sin mostrar la pantalla de login
- Si el token está expirado pero hay email guardado, se lanza una renovación silenciosa antes de entrar

---

## Funcionalidades por módulo

### Nueva Venta

- Lista completa de productos **ACTIVOS** con búsqueda en tiempo real por nombre o código
- Al seleccionar un producto se muestra el formulario con:
  - **Cantidad** — editable
  - **Precio unitario S/** — precargado del catálogo, editable manualmente
  - **Descuento S/** — monto fijo en soles (opcional)
  - **Total a cobrar S/** — calculado automáticamente (cantidad × precio − descuento), solo lectura
- **Selector de fecha** — por defecto el día actual, editable para registrar ventas retroactivas. Muestra aviso visual si la fecha es distinta a hoy
- El importe registrado en el Sheet es el **total real cobrado** (con descuento aplicado)

### Mis Ventas Hoy

- Filtrada por el email del usuario logueado y la fecha actual
- Resumen: total de transacciones activas, monto total vendido (para cuadre de caja), total de anulaciones
- Lista en orden inverso (más recientes primero)
- Las ventas anuladas aparecen tachadas con el motivo
- Opción de anular cualquier venta activa con motivo obligatorio

### Reporte por Fechas *(solo ADMIN)*

- Filtro por rango de fechas desde/hasta
- Tarjetas de resumen: Importe Total, Costo Total, Margen Total
- Tabla detallada por producto: Cantidad, Importe, Costo, Margen S/, Margen %
- El costo se obtiene del catálogo actual (limitación: si el costo cambió desde que se hizo la venta, el margen histórico puede ser inexacto)

### Stock *(solo ADMIN)*

- Tabla de todos los productos inventariables con columnas: Stock, Valor en Inventario, Proyección de Venta, Margen S/, Margen %
- 3 tarjetas de resumen global: Valor en Inventario, Proyección de Venta, Margen Total
- Registro de movimientos: **ENTRADA** (compras), **SALIDA** (retiros), **AJUSTE** (conteo físico que reemplaza el stock base)
- El stock proyectado se calcula dinámicamente (ver sección siguiente)

### Productos *(solo ADMIN)*

- CRUD completo del catálogo
- Código autogenerado: iniciales de las primeras 3 palabras del nombre + número correlativo (ej. `RON CARTAVIO BLACK` → `RCB246`)
- Activar/desactivar productos (nunca se eliminan para preservar historial)
- Vinculación bidireccional entre presentación unidad y presentación paquete: al guardar, la app actualiza automáticamente el campo Vínculo en ambos productos

### Config *(solo ADMIN)*

- Configuración del **Spreadsheet ID** (se guarda en `localStorage`, no en el Sheet)
- Nombres de las hojas (se leen/guardan en la hoja `Config`)
- Botón para recargar el catálogo de productos

---

## Catálogo de productos

El archivo `productos.csv` contiene el catálogo inicial con 405 productos usando `;` como separador y `,` como decimal (compatible con configuración regional es-PE).

**Para importar a Google Sheets:**
1. Abrir Google Sheets → Archivo → Importar
2. Subir el CSV
3. Separador: punto y coma (`;`)
4. No convertir números automáticamente (para evitar problemas con decimales)

---

## Cálculo de stock proyectado

El stock no se guarda como un número fijo — se **calcula dinámicamente** leyendo el historial de movimientos y ventas. Esto evita inconsistencias si se corrigen registros directamente en el Sheet.

**Algoritmo por producto:**

```
1. Leer todos los movimientos de stock del producto
2. Encontrar el último movimiento tipo AJUSTE → ese es el stock "base"
3. Sumar todas las ENTRADAS posteriores al último AJUSTE
4. Restar todas las SALIDAS posteriores al último AJUSTE
5. Restar todas las ventas activas (no anuladas) posteriores al último AJUSTE
6. Sumar/restar movimientos de paquetes vinculados (convertidos a unidades)
7. Restar ventas de paquetes vinculados (convertidos a unidades)
```

**Solo se calcula stock para productos inventariables:**
- Tipo `UNIDAD`
- Tipo `PAQUETE` sin vínculo (paquetes independientes)

Los paquetes con vínculo no tienen stock propio — sus movimientos se convierten a unidades del producto vinculado usando el factor de conversión.

---

## Lógica de paquetes y vínculos

Muchos productos tienen dos presentaciones: unidad individual y paquete/caja.

**Ejemplo:** Corona 330ml
- `CCE029` — CERVEZA CORONA EXTRA X 330 ML (UNIDAD, precio unitario)
- `CCE030` — CERVEZA CORONA EXTRA SIX PACK X 330 ML (PAQUETE, factor=6, vínculo=CCE029)

**Reglas:**
- El stock se lleva en **unidades** (`CCE029`)
- Si se registra una ENTRADA de 10 six-packs (`CCE030`), el sistema suma `10 × 6 = 60` unidades al stock de `CCE029`
- Si se vende 1 six-pack (`CCE030`), el sistema descuenta `1 × 6 = 6` unidades del stock de `CCE029`
- Los paquetes sin vínculo (ej. `CHC061` — Cerveza Helada Caja) se inventarían como entidad independiente

**Vínculo entre unidad y paquete:**
- La columna H de Productos guarda el código del producto relacionado
- Una UNIDAD puede apuntar al PAQUETE más grande disponible
- Un PAQUETE apunta a la UNIDAD base
- Al editar el vínculo desde la pestaña Productos, la app actualiza ambos lados automáticamente

---

## Zona horaria

La app está diseñada para Perú (**UTC-5**). Todas las fechas y horas visibles usan la hora local del dispositivo:

```javascript
function fechaLocal(d) {
  // Usa getFullYear/getMonth/getDate (hora local, no UTC)
  return y + '-' + m + '-' + day;
}

function horaLocal(d) {
  // Usa getHours/getMinutes/getSeconds (hora local, no UTC)
  return h + ':' + min + ':' + s;
}
```

El campo **Timestamp** (col A de Ventas y Stock) sigue usando UTC (`toISOString()`) porque se usa para ordenar cronológicamente y como referencia técnica. Los campos **Fecha** y **Hora** siempre reflejan la hora local.

---

## Cómo replicar el proyecto

### Paso 1: Configurar Google Cloud

Seguir la sección [Configuración de Google Cloud](#configuración-de-google-cloud) y obtener tu `CLIENT_ID`.

### Paso 2: Crear el Spreadsheet

1. Crear un nuevo Google Spreadsheet
2. Crear las hojas: `Productos`, `Ventas`, `Stock`, `Usuarios`, `Config`
3. Agregar encabezados en fila 1 de cada hoja (según la sección [Estructura del Google Spreadsheet](#estructura-del-google-spreadsheet))
4. En la hoja `Config`, agregar las filas:
   ```
   hoja_productos | Productos
   hoja_ventas    | Ventas
   hoja_stock     | Stock
   ```
5. En la hoja `Usuarios`, agregar tu email con rol `ADMIN`
6. Copiar el **Spreadsheet ID** desde la URL:
   ```
   https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit
   ```

### Paso 3: Adaptar el código

En `index.html` buscar y reemplazar:

```javascript
var CLIENT_ID = 'REEMPLAZAR_CON_TU_CLIENT_ID.apps.googleusercontent.com';
```

### Paso 4: Importar el catálogo

Importar `productos.csv` a la hoja `Productos` del Spreadsheet (ver sección [Catálogo de productos](#catálogo-de-productos)).

O crear los productos manualmente desde la pestaña **Productos** una vez que la app esté funcionando.

### Paso 5: Publicar en GitHub Pages

1. Crear un repositorio en GitHub
2. Subir `index.html` a la rama `main`
3. Ir a Settings → Pages → Source: `main` / `root`
4. La app estará disponible en `https://TU_USUARIO.github.io/TU_REPO/`
5. Agregar esa URL como origen autorizado en Google Cloud Console (ver Paso 1)

### Paso 6: Primera configuración

1. Abrir la app en el navegador
2. Iniciar sesión con la cuenta Google que pusiste como ADMIN en la hoja Usuarios
3. Ir a la pestaña **⚙ Config**
4. Ingresar el Spreadsheet ID
5. Clic en **Guardar configuración**
6. La app cargará el catálogo y estará lista

---

## Variables y constantes clave

```javascript
// OAuth
var CLIENT_ID = 'xxx.apps.googleusercontent.com'; // Reemplazar con el tuyo
var SCOPES = 'https://www.googleapis.com/auth/spreadsheets ' +
             'https://www.googleapis.com/auth/userinfo.email openid';

// localStorage keys
'dcafiro_sheet_id'    // Spreadsheet ID ingresado en Config
'dcafiro_user_email'  // Email del usuario (persistente)

// sessionStorage keys
'user_email'          // Email del usuario (por sesión de pestaña)
'access_token'        // Token OAuth actual
'token_expira'        // Timestamp de expiración del token (ms)
```

---

## Limitaciones conocidas

| Limitación | Descripción |
|-----------|-------------|
| Costo histórico | El costo en el Reporte por Fechas usa el costo actual del catálogo, no el costo al momento de la venta |
| Sin offline | La app requiere conexión a internet para todas las operaciones |
| Cuota de API | Google Sheets API tiene límite de 100 peticiones por 100 segundos por usuario |
| Modo Testing | Máximo 100 usuarios de prueba en Google Cloud. Suficiente para negocios pequeños sin pasar por revisión de Google |
| Token horario | El token OAuth dura 1 hora. La renovación silenciosa requiere que el navegador tenga cookies de Google habilitadas |
| Registros históricos | Los registros anteriores a la corrección de zona horaria (antes de junio 2026) pueden tener fecha UTC incorrecta. No se corrigen retroactivamente |
