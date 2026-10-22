# Sistema de módulos instalables · APP_IVJP27 (diseño, borrador 1)

Estado: **propuesta para revisar antes de implementar**. Módulo de ejemplo: `ejemplos/modulos/tv-ads/`
(«Panel de TV para anuncios»).

---

## 1. Diagnóstico de la aplicación actual

| Aspecto | Qué hay hoy | Consecuencia para los módulos |
|---|---|---|
| Lenguaje y framework | Node 22 (ESM) + Express 4 en `backend/src`; React 18 + Vite + Tailwind en `frontend` | El backend de un módulo es un módulo ESM que recibe un `Router` de Express |
| Rutas | Un único `routes.js`; lo público va antes de `rutas.use(auth.requiereSesion)` | Los módulos se montan en `/api/m/<nombre>/…`, **después** de `requiereSesion` (y con un hueco público opcional y declarado) |
| Autenticación | Cookie de sesión (`sesion`), `req.usuario` y contexto `AsyncLocalStorage` (`services/contexto.js`); `auth.soloAdminCuenta`; candado con pase personal (`modoAdmin.js`) | Se reutiliza tal cual: un módulo nunca gestiona sesiones |
| Perfiles | `usuario.perfiles TEXT[]` + perfiles creados `p_*`; mapa de perfiles en `permisosService.js` (`MOSAICOS`, `NOTIFICACIONES`, `deUsuario`) | Los mosaicos de módulos se suman a `MOSAICOS` en tiempo de ejecución; los permisos del módulo se guardan en el mismo mapa |
| Navegación | `TILES` fijo en `AppOriginal.jsx`; las zonas salen de la **forma** del mosaico en `Panal()`: `hex` → «Horarios y espacios», `cuadrado` → «El día a día», `circulo/triangulo` → «El centro» | Hay que sacar las zonas a la base de datos (`app_zona`) y la ubicación de cada mosaico (`app_mosaico`) |
| Iconos | Diccionario `Ico` de SVG en línea (unos 40) | Un módulo usa un icono del diccionario por nombre, o trae uno propio en SVG (saneado) |
| Estilos | Tailwind compilado en el bundle + clases propias (`card`, `btn`, `field`, `lbl`, `pill`, `chip`) en `original.css`; Montserrat local; **no hay variables CSS de diseño** | Se crea un «kit» CSS público con tokens y esas mismas clases para las vistas de los módulos |
| Base de datos | PostgreSQL con `pg`; DDL idempotente en `models/*.js` registrado en `MODELOS` (`migrar()`), sin tabla de migraciones | Los módulos necesitan **migraciones numeradas con registro** (`modulo_migracion`); no se toca `MODELOS` |
| Configuración | `.env` en `/etc/gestion-horarios`; ajustes en tabla `ajuste` | Rutas nuevas: `MODULOS_DIR`, `MODULOS_CLAVE_PUBLICA` |
| Despliegue | Servidor del centro, `/var/www/App`, `actualizar-app.sh` compila el frontend; datos en `/var/lib/gestion-horarios` | Los módulos se guardan **fuera** de la carpeta web (`/var/lib/gestion-horarios/modulos`) y **no** necesitan recompilar el frontend |
| Seguridad | CSP «solo código propio», CSRF por origen, registro de accesos, cierre de curso | Las vistas de los módulos se sirven desde el propio servidor (cumplen la CSP); instalar/actualizar/desinstalar va al registro de accesos |
| Tests | `node:test` de integración contra un esquema temporal de PostgreSQL (`npm test`) | Los tests de módulos siguen el mismo patrón |

**Limitación principal:** el frontend es un bundle compilado con un componente muy grande (`AppOriginal.jsx`). Un módulo instalado
en caliente **no puede añadir componentes React al bundle** sin recompilar. Por eso la interfaz de un módulo se carga en
**tiempo de ejecución** como una vista HTML propia (ver §6).

## 2. Puntos de integración (archivos que se reutilizan o cambian)

| Archivo | Cambio previsto |
|---|---|
| `backend/src/models/Modulo.js` (nuevo) | Tablas `modulo`, `modulo_migracion`, `app_zona`, `app_mosaico`; se añade a `MODELOS` |
| `backend/src/services/modulos/` (nuevo) | `manifiesto.js`, `paquete.js` (ZIP), `instalador.js`, `migraciones.js`, `registro.js`, `navegacion.js`, `autorizacion.js`, `cargador.js` |
| `backend/src/routes.js` | Monta `/api/modulos/*` (administración) y `/api/m/:modulo/*` (rutas de cada módulo) |
| `backend/src/app.js` | Sirve `/modulos/:nombre/(views|assets)/…` y `/modulos/kit/` con tipos de fichero permitidos |
| `backend/src/services/permisosService.js` | `MOSAICOS` = núcleo + módulos activos; mapa con `permisos` de módulos por perfil |
| `backend/src/services/registroAccesos.js` | Acciones `modulo_instalar`, `modulo_actualizar`, `modulo_desinstalar`, `modulo_config` |
| `frontend/src/original/AppOriginal.jsx` | `Panal` pinta zonas y mosaicos desde `GET /api/navegacion`; pantalla genérica `PantallaModulo` (iframe); mosaico de administración «Módulos» |
| `frontend/public/modulos-kit/` (nuevo) | `modulo.css` (tokens + clases de la app) y `modulo.js` (puente: `pedir()`, tamaño, avisos) |
| `deploy/instalar-en-var-www.sh` | Crea `/var/lib/gestion-horarios/modulos` con permisos del servicio |

## 3. Formato del paquete ZIP (versión de formato 1)

```text
tv-ads-1.0.0.zip
└── tv-ads/                      ← carpeta raíz = nombre del módulo (obligatoria y única)
    ├── module.yaml              ← manifiesto (obligatorio)
    ├── README.md                ← obligatorio
    ├── migrations/              ← SQL numerado: 001_….sql, 002_….sql (solo hacia delante)
    ├── src/servidor.js          ← opcional: rutas del backend (código de confianza, ver §8)
    ├── views/*.html             ← vistas que se abren dentro de la app
    ├── assets/                  ← css, js, svg, png, jpg, webp, woff2
    └── module.sig               ← firma Ed25519 del paquete (obligatoria si hay src/)
```

Reglas: un solo directorio raíz con el mismo `name` del manifiesto; nombres en minúsculas `[a-z0-9-]{2,40}`; sin enlaces
simbólicos; extensiones permitidas: `.yaml .md .sql .js .mjs .html .css .svg .png .jpg .jpeg .webp .woff2 .json .txt .sig`;
como máximo 20 MB comprimido, 60 MB descomprimido, 2 000 ficheros, ratio de compresión ≤ 100 (anti «zip bomb»).

### Manifiesto `module.yaml`

```yaml
formato: 1                       # versión del formato de paquete
name: tv-ads                     # identificador estable (no cambia entre versiones)
display_name: Panel de TV para anuncios
version: 1.0.0                   # SemVer
description: Pantallas de TV del centro con anuncios e imágenes que gestiona Jefatura.
author: Jefatura de Estudios · IES Valle del Jerte
min_app_version: "69.0.0"        # versión mínima de APP_IVJP27
max_app_version: "<100.0.0"      # opcional
dependencies: {}                 # otros módulos: { nombre: ">=1.2.0" }
entrypoint: src/servidor.js      # opcional; si existe, el paquete debe ir firmado

permissions:                     # permisos que define el módulo (se asignan a perfiles en la app)
  - key: ver                     # → tv-ads.ver
    label: Ver los anuncios
    default_profiles: [profesor, no_docente, conserjeria]
  - key: gestionar               # → tv-ads.gestionar
    label: Crear y publicar anuncios
    default_profiles: [jefe_estudios, direccion]

database:
  table_prefix: mod_tv_ads_      # todas sus tablas deben empezar así
  migrations:
    - migrations/001_crear_tablas.sql
    - migrations/002_pantallas.sql
  on_uninstall: keep             # keep (por defecto) | drop_if_confirmed

personal_data:                   # para el registro de tratamientos (RGPD art. 30)
  categories: [ninguno]          # o: [identificativos, contacto, …]; nunca salud/menores sin revisión del DPD
  retention_courses: 1

navigation:
  tile:
    id: tv-ads
    label: Panel de TV
    icon: { builtin: video }     # o { svg: assets/iconos/tv.svg }
    colors: ["#0f766e", "#2dd4bf"]
    suggested_zone: el-centro    # sugerencia: la ubicación final la decide la administración
    view: views/gestion.html
    permission: ver              # quién ve el mosaico
  public_views:                  # vistas sin sesión (pantallas de TV), con enlace secreto que genera la app
    - id: pantalla
      view: views/pantalla.html
      token: true

notifications: []                # avisos de portada que define el módulo (opcional)
```

Validación (en este orden y **antes** de ejecutar nada): estructura del ZIP → manifiesto (esquema) → `name`/`version`
→ compatibilidad con la versión de la app → dependencias instaladas y compatibles → existencia y ruta de cada fichero
declarado → migraciones (solo tablas con `table_prefix`, sin sentencias prohibidas) → firma si hay `entrypoint`.

## 4. Base de datos

Tablas nuevas (en `models/Modulo.js`):

```sql
CREATE TABLE IF NOT EXISTS modulo (
  nombre      TEXT PRIMARY KEY,                 -- tv-ads
  version     TEXT NOT NULL,
  estado      TEXT NOT NULL,                    -- instalado | activo | desactivado | error
  manifiesto  JSONB NOT NULL,
  instalado   TIMESTAMPTZ NOT NULL DEFAULT now(),
  actualizado TIMESTAMPTZ NOT NULL DEFAULT now(),
  por         TEXT NOT NULL DEFAULT '',
  error       TEXT NOT NULL DEFAULT ''
);
CREATE TABLE IF NOT EXISTS modulo_migracion (
  modulo   TEXT NOT NULL REFERENCES modulo(nombre) ON DELETE CASCADE,
  fichero  TEXT NOT NULL,                        -- migrations/001_crear_tablas.sql
  sha256   TEXT NOT NULL,                        -- no se puede cambiar una migración ya aplicada
  aplicada TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (modulo, fichero)
);
CREATE TABLE IF NOT EXISTS app_zona (
  id      TEXT PRIMARY KEY,                      -- horarios-y-espacios | el-dia-a-dia | el-centro | …
  nombre  TEXT NOT NULL,
  icono   TEXT NOT NULL DEFAULT '',
  forma   TEXT NOT NULL DEFAULT 'cuadrado',      -- hex | cuadrado | circulo (cómo se pintan sus mosaicos)
  orden   INT  NOT NULL DEFAULT 0,
  sistema BOOLEAN NOT NULL DEFAULT FALSE         -- las 3 actuales: se pueden renombrar/ordenar, no borrar con mosaicos dentro
);
CREATE TABLE IF NOT EXISTS app_mosaico (
  id       TEXT PRIMARY KEY,                     -- profesores, guardias… o el id del mosaico del módulo
  zona     TEXT NOT NULL REFERENCES app_zona(id),
  orden    INT  NOT NULL DEFAULT 0,
  etiqueta TEXT NOT NULL DEFAULT '',             -- vacío = la del núcleo o la del manifiesto
  icono    TEXT NOT NULL DEFAULT '',             -- vacío = el de origen
  origen   TEXT NOT NULL DEFAULT 'nucleo',       -- nucleo | modulo
  modulo   TEXT REFERENCES modulo(nombre) ON DELETE CASCADE,
  visible  BOOLEAN NOT NULL DEFAULT TRUE
);
```

- **Siembra:** al migrar, si `app_zona` está vacía se crean las 3 zonas actuales y se coloca cada mosaico de `TILES`
  donde está hoy (el aspecto de la portada no cambia hasta que la administración lo modifique).
- **Migraciones de módulo:** SQL plano, numerado, solo hacia delante. Se ejecutan **todas las pendientes dentro de una
  transacción** junto con el registro del módulo (en PostgreSQL el DDL es transaccional): si una falla, no queda nada a medias.
- **Reglas de las migraciones:** solo `CREATE/ALTER/DROP` de objetos con `table_prefix`, `CREATE INDEX`, `INSERT` de datos
  iniciales en sus tablas. Prohibido: tablas del núcleo, `GRANT`, `CREATE EXTENSION`, `COPY … PROGRAM`, funciones en
  lenguajes no confiables, `SET ROLE`. Se comprueba con un analizador sencillo de sentencias y además las migraciones se
  ejecutan con un **rol de PostgreSQL limitado** (`horarios_modulos`) que solo tiene permiso sobre el esquema `modulos`. Se ejecutan con `SET LOCAL ROLE horarios_modulos` y
  `SET LOCAL search_path TO modulos`, así sus tablas quedan en ese esquema aunque el SQL no lo indique.
- **Desinstalar:** por defecto **se conservan las tablas** (`on_uninstall: keep`); borrarlas es una acción aparte,
  explícita y confirmada escribiendo el nombre del módulo, y queda en el registro de accesos. El cierre de curso
  (`rgpdService`) se amplía para que cada módulo declare su plazo de conservación.

## 5. Perfiles y permisos

Se distinguen cuatro niveles:

1. **Instalado:** el paquete está en el servidor y sus migraciones aplicadas.
2. **Habilitado (activo):** sus rutas se montan y su mosaico puede aparecer.
3. **Visible para un perfil:** el mosaico figura en `tiles` del perfil en el mapa de perfiles (el mismo que hoy).
4. **Permiso para una acción:** `tv-ads.gestionar`, etc., asignado por perfil en «Perfiles y permisos» (nueva columna
   «Permisos de módulos»), con los `default_profiles` del manifiesto como valor inicial.

La comprobación es **siempre en el servidor**: `/api/m/:modulo/*` pasa por `requiereSesion` → módulo activo →
`autorizacion.puede(usuario, 'tv-ads.ver')`. Cada ruta del módulo declara el permiso que exige
(`ctx.ruta.get('/anuncios', 'ver', handler)`); una ruta sin permiso declarado no se monta. La cuenta de administración
puede todo. Las vistas públicas (`public_views`) usan `/api/m/<módulo>/publico/…`: sin sesión, pero con el enlace secreto de cada
pantalla (se guarda su hash y se compara en tiempo constante). Ocultar el icono no da acceso ni lo quita: el servidor responde 404 si falta el permiso.

## 6. Interfaz de los módulos e integración visual

- El mosaico del módulo abre `PantallaModulo`: la cabecera de sección de la app (botón volver, icono, título) y, debajo,
  la vista del módulo en un `<iframe sandbox="allow-scripts allow-same-origin allow-forms allow-downloads">` servido desde
  `/modulos/<nombre>/views/…` (mismo origen: cumple la CSP «solo código propio»).
- `frontend/public/modulos-kit/modulo.css` define los **tokens** de la app (`--ink`, `--ink-soft`, `--paper`, `--divider`,
  `--morado`, radios, sombras, Montserrat) y las mismas clases (`card`, `btn`, `btn-dark`, `btn-ghost`, `field`, `lbl`,
  `pill`, `chip`, tablas y mensajes). Las vistas de los módulos solo enlazan este kit; no traen su propio sistema visual.
- `modulo-kit/modulo.js` da a la vista: `Modulo.pedir(metodo, ruta, cuerpo)` (llama a `/api/m/<nombre>/…` con la sesión),
  `Modulo.usuario()`, `Modulo.permisos()`, `Modulo.aviso(texto)` (usa el aviso de la app) y ajuste automático de altura.
- Iconos: `builtin:` usa el diccionario `Ico` de la app; un SVG propio se sanea (sin `<script>`, `on*`, `href` externos).

## 7. Navegación y zonas configurables

- Nueva pantalla de administración **«Módulos y portada»** con dos pestañas:
  - **Módulos:** lista (nombre, versión, estado, autor, firma), botones *Instalar ZIP*, *Actualizar*, *Activar/Desactivar*,
    *Desinstalar*, *Borrar datos* (aparte, confirmado), y enlace a sus permisos.
  - **Portada:** zonas (crear, renombrar, icono, forma, ordenar arrastrando, borrar si está vacía) y mosaicos
    (mover de zona, ordenar, cambiar etiqueta/icono, ocultar). Vista previa de la portada.
- `GET /api/navegacion` devuelve zonas y mosaicos ya filtrados por los permisos del usuario; `Panal()` deja de agrupar
  por forma y pinta lo que recibe. La sugerencia `suggested_zone` del manifiesto solo se usa al instalar; después manda
  la configuración.

## 8. Modelo de confianza y seguridad del instalador

- **Módulos declarativos** (sin `entrypoint`): migraciones + vistas + assets. Sus vistas solo pueden llamar a la API
  genérica del módulo (CRUD sobre sus propias tablas, declarado en el manifiesto). Se pueden instalar sin firma.
- **Módulos con código de servidor** (`src/servidor.js`): ese código se ejecuta dentro del proceso de la app con su mismo
  acceso; **solo se aceptan firmados** con la clave privada del centro (Ed25519). El servidor tiene la pública en
  `MODULOS_CLAVE_PUBLICA`. La firma cubre el SHA-256 de cada fichero del paquete (`module.sig` = lista + firma).
  Un paquete con código y sin firma válida se rechaza con un mensaje claro.
- El código del módulo **no recibe** el `pool` de la base de datos: recibe `ctx.db` limitado a sus tablas (consultas
  parametrizadas con prefijo comprobado) y `ctx.usuario`, `ctx.registrar()`, `ctx.config` (solo sus ajustes).
- ZIP no confiable: lectura en streaming (`yauzl`), rechazo de rutas absolutas, `..`, `\`, nombres con NUL, enlaces
  simbólicos, extensiones no permitidas, tamaños/ratios excesivos; extracción a una carpeta temporal propia
  (`MODULOS_DIR/.tmp/<uuid>`), validación completa y solo entonces `rename` atómico a `MODULOS_DIR/<nombre>/<versión>/`.
- Solo la **cuenta de administración con el candado abierto** instala, actualiza o desinstala; todo queda en el registro de
  accesos con el SHA-256 del ZIP.

## 9. Ciclo de vida

```text
Instalar:   ZIP → validar → manifiesto → compatibilidad/dependencias → firma (si hay código) → extraer a .tmp
            → BEGIN: migraciones pendientes + INSERT modulo + permisos por defecto + app_mosaico (zona sugerida) COMMIT
            → mover a MODULOS_DIR/<nombre>/<versión> → cargar rutas → estado «activo»
Actualizar: igual, comprobando versión destino > actual, mismo `name`, migraciones ya aplicadas sin cambios (sha256),
            aplicando solo las nuevas; la versión anterior se conserva en disco para volver atrás si falla la carga.
Desactivar: se desmontan sus rutas y se oculta su mosaico; datos y configuración intactos.
Desinstalar: desactivar → borrar registro, mosaico y permisos → borrar ficheros del módulo → tablas: se conservan
            (borrarlas es la acción aparte «Borrar datos del módulo»).
Arranque:   se cargan los módulos «activos»; si uno falla, pasa a «error» sin impedir que arranque la app.
```

Versionado (SemVer): **PATCH** = correcciones sin migraciones; **MINOR** = funciones nuevas y migraciones que solo añaden;
**MAJOR** = cambios incompatibles (la actualización muestra aviso y exige confirmación). La app pasa a tener versión
SemVer (`69.0.0` = v69) para `min_app_version`.

## 10. Tests previstos (`backend/test/modulos.test.js`)

Manifiesto válido e inválido · ZIP malformado · path traversal (`../`, ruta absoluta, enlace simbólico) · zip bomb ·
extensión no permitida · instalación · instalación duplicada · actualización (y bajada de versión rechazada) ·
migraciones pendientes y ya aplicadas · migración modificada (sha distinto) · fallo de migración → rollback sin registro ·
migración que toca una tabla del núcleo → rechazada · paquete con código sin firma → rechazado · desinstalación sin borrar
datos · borrar datos con confirmación · permisos por perfil · acceso a `/api/m/…` sin permiso aunque el icono esté oculto ·
zonas (crear, ordenar, no borrar con mosaicos) · mover y ordenar mosaicos · icono SVG saneado · compatibilidad de versiones.

## 11. Fases de implementación propuestas

1. **Zonas y mosaicos en base de datos** (sin módulos): `app_zona`, `app_mosaico`, `GET /api/navegacion`, pantalla «Portada».
   La portada queda igual que hoy.
2. **Núcleo de módulos:** manifiesto, validación del ZIP, instalador transaccional, migraciones, registro, permisos.
3. **Vistas y kit visual:** `PantallaModulo`, `modulos-kit`, pantalla «Módulos».
4. **Módulos con código firmado:** firma Ed25519 y herramienta `deploy/firmar-modulo.sh`.
5. **Módulo de ejemplo** `tv-ads` instalado desde su ZIP y tests de extremo a extremo.

Dependencias nuevas en el backend: `yauzl` (lectura segura de ZIP) y `yaml` (manifiesto).

## 12. Requisitos que no se pueden cumplir tal cual

- **Componentes React dentro del bundle:** no sin recompilar; se sustituye por vistas en iframe con el kit visual (§6).
- **Ejecutar código arbitrario de un ZIP:** no se permite; solo código firmado por el centro (§8).
- **«Desinstalar» borrando datos automáticamente:** contrario a la política de datos de la app; es una acción aparte.
