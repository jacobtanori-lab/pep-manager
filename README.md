# PEP Manager

SPA single-file para gestionar un **repositorio personal de Personas Políticamente Expuestas (PEP) nacionales**, basado en el **Anexo 1 del documento SHCP "Lista de PEP Nacionales 2020"**.

- **Frontend:** un único `index.html` con HTML + [Tailwind CSS](https://cdn.tailwindcss.com) (CDN) + JavaScript vanilla.
- **Auth:** [MSAL.js v3](https://github.com/AzureAD/microsoft-authentication-library-for-js) por CDN, multi-tenant + cuenta MSA personal.
- **Backend:** Microsoft Graph **Workbook API** leyendo/escribiendo un archivo Excel en OneDrive personal.
- **Hosting:** GitHub Pages.

**Repo:** https://github.com/jacobtanori-lab/pep-manager
**URL pública:** https://jacobtanori-lab.github.io/pep-manager/

---

## Arquitectura de datos

Archivo Excel en OneDrive personal: `/Apps/PEP-Manager/database.xlsx`

Endpoint base de Graph:

```
https://graph.microsoft.com/v1.0/me/drive/root:/Apps/PEP-Manager/database.xlsx:/workbook
```

Tablas estructuradas (named tables):

| Tabla              | Descripción                                          |
|--------------------|------------------------------------------------------|
| `tbl_Dependencias` | Catálogo de entes públicos (262 filas)               |
| `tbl_Cargos`       | Catálogo de cargos PEP del Anexo 1 SHCP (431 filas)  |
| `tbl_Personas`     | Tabla viva con nombres concretos                     |

Más una hoja `CAT_Parametros` con celdas de configuración (`anios_remanencia`, etc.).

### Archivo separado: listas internacionales

Las listas de sanciones (OFAC, ONU, UE, etc.) viven en un **segundo archivo Excel**,
separado de `database.xlsx` a propósito: son datos importados en bloque (OFAC por sí
solo puede traer miles de filas) y no queremos que ese volumen infle o ponga en
riesgo el archivo de Personas.

Archivo: `/Apps/PEP-Manager/listas_internacionales.xlsx`

| Tabla           | Descripción                                                        |
|-----------------|---------------------------------------------------------------------|
| `CAT_Listas`    | Configuración: una fila por lista (URL de la fuente, formato, tabla destino, si está activa, última actualización, total de registros) |
| `tbl_<ID>`      | Una tabla destino por lista, con el mismo esquema de 13 columnas (`COLUMNAS_LISTA`) |

Al 26-nov-2026 hay **30 listas** configuradas, de sanciones (OFAC SDN y Consolidated,
ONU, UE, Reino Unido, Suiza, Francia, Canadá, Australia, FinCEN 311, los cinco bancos
multilaterales de desarrollo…), criminales (Interpol, FBI, DEA, US Marshals…),
terrorismo y fiscales (SAT 69-B, SFP). Cada lista nueva agrega una fila en
`CAT_Listas` + su propia tabla `tbl_*` + una entrada en `ADAPTADORES_LISTAS` en el
código; si el formato ya está soportado (p. ej. el CSV "simple" de OpenSanctions),
basta la entrada y **no hace falta código nuevo**.

#### Dos carriles según la licencia de la fuente

| Carril | Cómo se obtiene | Cuándo se usa |
|--------|-----------------|---------------|
| **Espejo** | Un GitHub Action baja la fuente y la commitea comprimida en `data/<ID>.<fmt>.gz`; la app la lee del mismo origen, así que **no hay CORS** | Fuentes de dominio público o con licencia abierta. Las fuentes a espejar se declaran en `sanctions-sources.json` |
| **En vivo** | La app descarga la fuente directo desde el navegador | Fuentes que **no se pueden redistribuir** (OpenSanctions es CC-BY-NC; los términos del AfDB reservan el copyright) y que sí mandan CORS |

Una fuente que ni permite espejo ni manda CORS no sirve por ninguno de los dos
carriles: es la razón de que ADB y EBRD se lean vía OpenSanctions y no de su banco.

El orden de descarga al actualizar es: espejo comprimido → espejo plano → fuente
original → y si todo falla, subida manual del archivo.

**Los nombres de tabla y de columna deben coincidir exactamente** con lo que espera
el código (aunque no importan mayúsculas/acentos en las columnas de este módulo,
sí importa el nombre exacto de la Tabla de Excel: `CAT_Listas`, `tbl_OFAC_SDN`, etc.
— debe ser una Tabla real, creada con Ctrl+T, no solo una hoja con encabezados).

---

## Configuración de Azure (App Registration)

| Parámetro     | Valor                                                        |
|---------------|-------------------------------------------------------------|
| Client ID     | `a5b306a7-4649-4395-b4ac-2b58094510df`                       |
| Authority     | `https://login.microsoftonline.com/common`                  |
| Redirect URI  | `https://jacobtanori-lab.github.io/pep-manager/`            |
| Scopes        | `User.Read`, `Files.ReadWrite`, `offline_access`            |
| Plataforma    | **Single-page application (SPA)** — Auth Code Flow + PKCE   |

> **IMPORTANTE:** se usa `authority = common` (no el tenant específico) para que
> funcione con cuentas **MSA personales**. El tenant específico solo aplicaría en un
> escenario *workplace*.

### Pasos en Azure Portal

1. **Azure Active Directory → App registrations → New registration.**
2. Supported account types: *Accounts in any organizational directory and personal Microsoft accounts*.
3. En **Authentication → Add a platform → Single-page application**, registra **ambos** Redirect URIs:
   - `https://jacobtanori-lab.github.io/pep-manager/` (producción)
   - `http://localhost:5500/` (desarrollo local con Live Server)
4. En **API permissions** agrega los scopes delegados de Microsoft Graph:
   `User.Read`, `Files.ReadWrite`, `offline_access`.

---

## Desarrollo local

No hay build tools: es HTML + JS plano. **No se requiere `npm install`.**

1. Abre la carpeta en VS Code.
2. Instala la extensión **Live Server**.
3. Click en **"Go Live"** (sirve en `http://localhost:5500/`).
4. Asegúrate de que `http://localhost:5500/` esté registrado como Redirect URI SPA en Azure.

---

## Deploy a GitHub Pages

1. Crea el repo `pep-manager` en GitHub y sube los archivos:

   ```bash
   git init
   git add .
   git commit -m "MVP: login MSAL + lectura de tbl_Personas"
   git branch -M main
   git remote add origin https://github.com/jacobtanori-lab/pep-manager.git
   git push -u origin main
   ```

2. En el repo: **Settings → Pages → Build and deployment → Source: Deploy from a branch**, rama `main`, carpeta `/ (root)`.
3. La app queda publicada en `https://jacobtanori-lab.github.io/pep-manager/`.

---

## Estado actual

La app se organiza en cinco pestañas: **Personas**, **Catálogos**, **Listas**,
**Fotos** y **Bitácora**.

### Autenticación
- Landing con login **"Iniciar sesión con Microsoft"** (popup MSAL, con fallback a interacción).
- Tras el login: nombre del usuario + botón **Cerrar sesión** en el header sticky.
- Indicador de carga, estado vacío y manejo de errores de Graph.

### Personas
- Lectura de `tbl_Personas` vía Graph y render en tabla responsiva.
- **CRUD completo:** alta, edición y **baja lógica (soft delete)**.
- Formulario con validación, lógica condicional de dependencia/entidad/municipio
  según el cargo, y campos extra para tipo PEP (Familiar/Asociado).
- Generación automática de `persona_id`, formato forzado `dd/mm/yyyy` en fechas,
  y control de cambios sin guardar (dirty tracking).
- Búsqueda/filtro en vivo.

### Catálogos
- Vista de `tbl_Dependencias` y `tbl_Cargos` con subtabs y búsqueda propia.

### Listas Internacionales
- Configuración vía `CAT_Listas` con una card por lista.
- **30 listas** configuradas, con **29 entradas** en `ADAPTADORES_LISTAS` resueltas por
  **21 adaptadores** distintos (varias listas comparten uno: siete usan el CSV "simple"
  de OpenSanctions y tres el de organizaciones terroristas). La lista restante
  (`AM_LPB`, señales de adverse media de la LPB de la UIF) es de **curación manual** y
  por eso no lleva adaptador ni URL.
- Formatos cubiertos: XML, CSV y JSON, más fuentes que exigen armarse aparte (un XLSX
  que el Action convierte a JSON, y algunos scrapes publicados como espejo estático).
- Descarga con reintento entre candidatos (espejo comprimido, espejo plano, fuente
  original) + respaldo por **subida manual** del archivo; escritura por bloques de 500
  filas a la tabla destino y visor de registros con filtro.
- Buscador de coincidencias que lee los espejos **directo en el navegador**, sin pasar
  por Excel, con ponderación por IDF y caché entre sesiones.
- Las listas que conservan historial (Suiza, AfDB, CSL, SAT 69-B) marcan lo que ya no
  está vigente con `estatus` **"Histórico"** *y* un aviso al inicio de
  `programa_motivo`, porque `estatus` no es una de las columnas que el buscador
  muestra.
- Tolerancia a mayúsculas/acentos en columnas y diagnóstico de la tabla real al
  fallar con 404.

## Próximos pasos (roadmap)

- **Dashboard** (conteos por estatus, vencimientos de remanencia, etc.).
- Joins legibles por nombre en catálogos y filtros avanzados (entidad federativa,
  tipo PEP).
- Más listas: CNBV/CONDUSEF, proveedores sancionados de la SABG y PEPs
  internacionales masivos. **SAT art. 69** quedó investigado y aplazado: sus cinco
  padrones "core" suman 508,239 filas y no caben en la arquitectura actual, porque el
  buscador cachea cada lista completa en el navegador y Excel se escribe de 500 en 500.
- Familiares y asociados de PEPs: el modelo ya existe (`tipo_pep` +
  `persona_id_relacionado`), falta poblarlo; las fuentes medidas hasta ahora dan muy
  pocos vínculos.
- Cobertura de fotografías: ampliar el repositorio de retratos con procedencia y
  licencia verificable (hoy cubre una fracción pequeña de la base).
