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

## MVP actual

- Landing con login **"Iniciar sesión con Microsoft"** (popup MSAL, con fallback a interacción).
- Tras el login: nombre del usuario + botón **Cerrar sesión** en el header sticky.
- Lectura de `tbl_Personas` vía Graph y render en tabla responsiva:
  `persona_id`, `nombre_completo`, `cargo_id`, `dependencia_efectiva_id`, `estatus`, `fecha_inicio_cargo`.
- Indicador de carga, estado vacío y manejo de errores de Graph.

## Próximos pasos (roadmap)

- CRUD de personas (alta/edición/baja con `rows/add`, `itemAt(index)` PATCH/DELETE).
- Vista de catálogos (`tbl_Dependencias`, `tbl_Cargos`) con joins legibles por nombre.
- Búsqueda y filtros (por nombre, estatus, entidad federativa, tipo PEP).
- Dashboard (conteos por estatus, vencimientos de remanencia, etc.).
