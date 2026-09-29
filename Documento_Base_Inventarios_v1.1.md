# PROYECTO: Sistema de Inventarios para PYMES

## Documento Base de Desarrollo

**Versión:** 1.1
**Estado:** Pendiente de aprobación
**Cambios respecto a v1.0:** Windows como plataforma principal (servidor y clientes), rediseño de despliegue e instalación, cambios de arquitectura, modelo de datos, seguridad y operación. Ver sección 15.

---

## 1. Visión del Proyecto

Desarrollar un **sistema comercial de gestión de inventarios** para PYMES que sea:

- **Funcional y simple** para el usuario final
- **Nativo del ecosistema Windows**: se instala y opera en equipos y servidores Windows, que es lo que la mayoría de las PYMES ya tiene
- **Escalable** en el tiempo
- **Mantenible** por cualquier desarrollador Python
- **On-premise**: funciona en red local, sin depender de internet
- **Multi-almacén** con administración centralizada
- **Preparado para multi-empresa** desde el diseño, aunque empiece con una sola
- **Instalable por un técnico no desarrollador**, mediante un instalador `.exe`

El sistema permitirá que un administrador gestione **distintos almacenes desde una misma terminal**, con capacidad futura de sincronización a la nube.

---

## 2. Plataforma objetivo: Windows

### 2.1 Definición de "soporte Windows"

| Rol | Plataforma soportada | Notas |
|-----|---------------------|-------|
| **Servidor** (producción) | Windows 10/11 Pro o Windows Server 2019/2022/2025 (64 bits) | Instalación nativa mediante instalador `.exe` |
| **Clientes** | Cualquier equipo con Microsoft Edge o Google Chrome | No se instala nada; opcionalmente como app instalable (PWA) |
| **Lectores de código de barras** | USB y Bluetooth en modo teclado (HID) | Funcionan sin controladores |
| **Desarrollo** | Windows con Docker Desktop, o Linux/Mac | Docker solo para desarrollo y pruebas |
| **Servidor Linux** | Soportado como opción secundaria | Mediante Docker Compose |

### 2.2 Integraciones con el ecosistema Windows

| Integración | Release | Descripción |
|-------------|---------|-------------|
| Servicios de Windows | v0.1 | API, frontend, proxy y base de datos corren como servicios con inicio automático y recuperación ante fallos |
| Programador de tareas | v0.1 | Respaldos automáticos programados |
| Firewall de Windows | v0.1 | El instalador abre solo el puerto 443 |
| Certificado HTTPS en clientes | v0.1 | Instalación del certificado raíz interno en el almacén de Windows |
| Importación y exportación Excel (.xlsx) | v0.1 | Carga masiva de productos y reportes exportables |
| Impresión (reportes y etiquetas) | v0.5 | Impresión desde el navegador; etiquetas ZPL/térmicas en v1.0 |
| Visor de eventos de Windows | v0.5 | Errores críticos del servicio registrados en el Event Log |
| Inicio de sesión con Active Directory / LDAP | v1.0 (opcional) | Para clientes con dominio |

### 2.3 Modos de despliegue

**Modo A: Windows nativo (producción recomendada para clientes Windows)**

No requiere Docker en el servidor del cliente.

```
Servidor Windows del cliente
├── PostgreSQL 18            → servicio de Windows (instalador oficial)
├── API FastAPI (Uvicorn)    → servicio de Windows (WinSW)
├── Frontend Reflex          → servicio de Windows (WinSW)
├── Caddy (HTTPS)            → servicio de Windows, único puerto expuesto (443)
└── Tarea programada         → respaldo diario (pg_dump)
```

**Modo B: Docker Compose (desarrollo, pruebas y servidores Linux)**

Misma aplicación, mismo código; solo cambia el empaquetado.

**Justificación:** Docker Desktop en Windows depende de WSL2 y de virtualización, no está soportado en Windows Server, exige una sesión de usuario iniciada y tiene condiciones de licencia para empresas grandes. Para un producto comercial que se instala en equipos ajenos, es una fuente constante de soporte. El modo nativo elimina esa dependencia.

### 2.4 Puntos técnicos específicos de Windows (deben resolverse desde el inicio)

- **Bucle de eventos asíncrono:** psycopg 3 en modo asíncrono requiere `SelectorEventLoop` en Windows, y Python usa `ProactorEventLoop` por defecto. Se configura explícitamente al arranque y se cubre con una prueba automatizada en CI sobre Windows.
- **Rutas y codificación:** uso de `pathlib` en todo el código, UTF-8 explícito y `.gitattributes` para normalizar saltos de línea.
- **Tareas de proyecto:** `just` (o `uv run` con tareas) en lugar de `Makefile`, que en Windows es problemático.
- **Empaquetado de Python:** distribución con intérprete embebido y dependencias fijadas (lockfile de `uv`), sin exigir Python instalado en el servidor.
- **Reflex en producción:** el frontend se exporta y se sirve de forma que el servidor del cliente no requiera Node.js instalado; esto se valida en la Fase 0.
- **Hostname en LAN:** los clientes acceden por nombre (por ejemplo `https://inventarios.local` o el nombre del equipo) y no por IP; se documenta la configuración de IP fija o reserva DHCP.
- **Energía y actualizaciones:** los servicios deben arrancar solos tras un reinicio (por ejemplo, por Windows Update). Se documenta desactivar suspensión e hibernación en el servidor.
- **Antivirus:** se documentan las exclusiones recomendadas para la carpeta de datos de PostgreSQL.
- **Firma de código:** el instalador se firma con un certificado de firma de código para evitar advertencias de SmartScreen. Es un costo comercial a presupuestar.

---

## 3. Stack Tecnológico

| Capa | Tecnología | Propósito |
|------|-----------|-----------|
| **Backend** | Python 3.12+ | Lenguaje principal |
| **Framework API** | FastAPI | API REST, documentación automática |
| **ORM** | SQLAlchemy 2.0 | Modelos en Python |
| **Migraciones** | Alembic | Versionado de la base de datos |
| **Base de datos** | PostgreSQL 18 | Motor relacional, `uuidv7()` nativo |
| **Frontend** | Reflex (sujeto a validación en Fase 0) | Interfaz web en Python |
| **Plan B de frontend** | FastAPI + Jinja2 + HTMX | Si Reflex no supera el prototipo |
| **Proxy / HTTPS** | Caddy | HTTPS con certificado interno, único punto de entrada |
| **Servicios Windows** | WinSW | Ejecuta API, frontend y Caddy como servicios |
| **Instalador** | Inno Setup | `.exe` de instalación y actualización |
| **Contenedores** | Docker | Desarrollo, pruebas y servidores Linux |
| **Gestión de dependencias** | uv | Instalación reproducible |
| **Tareas** | just | Comandos únicos multiplataforma |
| **Autenticación** | JWT (access corto) + refresh token en BD | Sesiones revocables |
| **Hash de contraseñas** | Argon2id | Estándar actual |
| **Excel** | openpyxl | Importación y exportación .xlsx |
| **Calidad** | ruff, pre-commit, pytest | Lint, formato y pruebas |
| **CI** | GitHub Actions (runners Windows y Linux) | Pruebas en ambas plataformas |

**Filosofía del stack:** un solo lenguaje (Python), con el empaquetado adaptado a Windows para que el cliente final nunca necesite saber que existe Python.

---

## 4. Arquitectura General

```
┌──────────────────────────────────────────────────────────────┐
│                       RED LOCAL (LAN)                        │
│                                                              │
│  ┌─────────────┐        ┌─────────────────────────────────┐  │
│  │ Admin       │        │  Servidor Windows               │  │
│  │ Edge/Chrome │◄──────►│                                 │  │
│  └─────────────┘  HTTPS │  ┌───────────────────────────┐  │  │
│                    443  │  │ Caddy (proxy, TLS)        │  │  │
│  ┌─────────────┐        │  └────────────┬──────────────┘  │  │
│  │ Almacén 1   │◄──────►│       ┌───────┴────────┐        │  │
│  │ Edge/Chrome │        │       ▼                ▼        │  │
│  └─────────────┘        │  ┌─────────┐    ┌──────────┐   │  │
│                         │  │ Reflex  │───►│ FastAPI  │   │  │
│  ┌─────────────┐        │  │(frontend)│HTTP│  (API)   │   │  │
│  │ Almacén 2   │◄──────►│  └─────────┘    └────┬─────┘   │  │
│  │ + lector    │        │                      ▼         │  │
│  └─────────────┘        │            ┌──────────────────┐ │  │
│                         │            │  PostgreSQL 18   │ │  │
│                         │            └──────────────────┘ │  │
│                         │  Tarea programada: respaldo     │  │
│                         └─────────────────────────────────┘  │
│                                                              │
│           (Futuro opcional: sincronización a nube)          │
└──────────────────────────────────────────────────────────────┘
```

**Decisiones de arquitectura:**

- **Reflex consume FastAPI por HTTP** mediante un cliente (`httpx`). Toda la lógica de negocio vive en la API, de modo que una app móvil o un lector dedicado puedan usarla después sin cambios.
- **Caddy es el único puerto expuesto (443).** PostgreSQL y FastAPI escuchan solo en `localhost`.
- **Backend organizado por módulos de dominio**, no por tipo de archivo.
- **Sin dependencia de internet** para operar.

---

## 5. Estructura del Proyecto

```
inventarios/
├── README.md
├── justfile                       ← tareas: up, migrate, test, backup, build-installer
├── pyproject.toml                 ← workspace uv (backend + frontend)
├── .env.example
├── .gitattributes                 ← normalización de saltos de línea
├── .pre-commit-config.yaml
│
├── docker-compose.yml             ← postgres, api, frontend, caddy (modo B)
├── docker-compose.dev.yml         ← adminer, recarga en caliente
│
├── deploy/
│   ├── caddy/Caddyfile
│   ├── windows/
│   │   ├── installer.iss          ← script de Inno Setup
│   │   ├── services/              ← configuración WinSW (api, frontend, caddy)
│   │   ├── install.ps1            ← configura BD, servicios, firewall, certificado
│   │   ├── upgrade.ps1            ← respaldo + migraciones + reinicio de servicios
│   │   ├── backup.ps1             ← pg_dump, rotación, copia a ruta externa
│   │   ├── restore.ps1
│   │   └── uninstall.ps1
│   └── linux/                     ← equivalentes para modo B
│
├── docs/
│   ├── 01_arquitectura.md
│   ├── 02_modelo_datos.md
│   ├── 03_manual_instalacion_windows.md
│   ├── 04_reglas_negocio.md
│   ├── 05_operacion_respaldos.md
│   ├── 06_guia_soporte_windows.md ← firewall, antivirus, energía, certificados
│   └── decisiones/                ← ADRs (una hoja por decisión)
│
├── backend/
│   ├── alembic/versions/
│   ├── app/
│   │   ├── main.py
│   │   ├── core/                  ← config, database, security, errores, dependencias
│   │   └── modules/
│   │       ├── auth/              ← usuarios, roles, permisos, sesiones
│   │       ├── empresas/
│   │       ├── catalogo/          ← productos, categorías, unidades
│   │       ├── almacenes/
│   │       ├── inventario/        ← documentos, movimientos, existencias
│   │       ├── importacion/       ← Excel/CSV
│   │       ├── reportes/
│   │       └── auditoria/
│   │           (cada módulo: models.py, schemas.py, service.py, router.py)
│   └── tests/
│       ├── unit/
│       └── integration/           ← PostgreSQL real, concurrencia, Windows
│
├── frontend/                      ← Reflex, con api_client/
│   └── inventarios/
│       ├── pages/
│       ├── components/
│       ├── state/
│       └── api_client/
│
└── .github/workflows/             ← CI en Windows y Linux
```

---

## 6. Modelo de Datos (Alto Nivel)

### 6.1 Entidades

| Entidad | Propósito |
|---------|-----------|
| **empresas** | Multi-tenant; datos aislados por empresa |
| **roles** | Plantillas de permisos (ADMINISTRADOR, GERENTE, ALMACEN, VENTAS, CONSULTA) |
| **permisos / rol_permisos** | Permisos granulares (`inventario.entrada.crear`, `productos.editar`) |
| **usuarios** | Personas con acceso al sistema |
| **sesiones** | Refresh tokens revocables |
| **almacenes** | Ubicaciones físicas |
| **usuario_almacenes** | Acceso N:M usuario-almacén |
| **categorias** | Clasificación de productos |
| **unidades_medida** | Pieza, kg, litro, caja, etc. |
| **productos** | SKU, código de barras, unidad, precios, mínimo/máximo |
| **existencias** | Stock y costo promedio por almacén y producto (caché derivada) |
| **tipos_movimiento** | ENTRADA, SALIDA, AJUSTE, TRASPASO, DEVOLUCIÓN |
| **documentos_inventario** | Encabezado: tipo, usuario, fecha, referencia, observaciones |
| **movimientos_inventario** | Líneas del kardex (append-only) |
| **importaciones** | Historial de cargas masivas y sus errores |
| **auditoria** | Quién cambió qué, con valores antes y después |
| **dispositivos** | Terminales y lectores (v1.0) |

### 6.2 Reglas de diseño

- **Claves primarias UUIDv7** con `uuidv7()` de PostgreSQL 18: ordenables por tiempo, buenas para índices y para sincronización futura.
- **`TIMESTAMPTZ`** en todas las fechas; el servidor guarda UTC y la interfaz muestra la zona horaria local de la empresa.
- **Columnas estándar** en tablas de negocio: `id`, `empresa_id`, `created_at`, `updated_at`, `created_by`, `activo`.
- **UNIQUE compuestos** con `empresa_id`: `(empresa_id, sku)`, `(empresa_id, codigo_barras)`.
- **Cantidades** en `NUMERIC(14,4)`.
- **Valuación por costo promedio ponderado.** `existencias.costo_promedio` y, por movimiento, `costo_unitario`, `existencia_anterior` y `existencia_nueva`. PEPS queda descartado por ahora y registrado como decisión.
- **Kardex append-only**: sin `UPDATE` ni `DELETE`, reforzado con trigger. Las correcciones son movimientos inversos.
- **Existencias como caché derivada**: se actualiza en la misma transacción y se verifica con pruebas que coincide con la suma de movimientos.
- **Campos de sincronización desde el inicio:** `updated_at`, `version` (control optimista) y borrado lógico, para habilitar la sincronización a nube sin rediseñar.
- **Row Level Security** por `empresa_id` como segunda barrera (v1.0). La columna existe desde el día uno.
- **Lotes y caducidad:** fuera de alcance por ahora, documentado como extensión futura.

---

## 7. Modelo de Seguridad

- **Autenticación:** usuario y contraseña; access token de 15-30 minutos y refresh token guardado en `sesiones`, de modo que un administrador pueda cerrar la sesión de cualquier usuario.
- **Contraseñas:** Argon2id. Nunca texto plano.
- **Límite de intentos** de login con bloqueo temporal.
- **Autorización:** RBAC con permisos granulares; los cinco roles son plantillas iniciales.
- **Aislamiento multi-tenant:** filtro por empresa en código y RLS en v1.0.
- **Control por almacén:** acceso limitado por usuario.
- **HTTPS obligatorio** en LAN mediante Caddy con CA interna. El instalador genera el certificado raíz y una guía documenta su distribución a los clientes (instalación manual o por directiva de grupo si hay dominio).
- **Secretos:** la clave JWT y la contraseña de la BD se generan aleatoriamente en la instalación y se almacenan con permisos restringidos (cuenta de servicio dedicada, sin privilegios de administrador).
- **Puertos:** PostgreSQL y API solo escuchan en `localhost`. Firewall de Windows abre únicamente el 443.
- **Adminer:** solo en el perfil de desarrollo.

---

## 8. Alcance por Releases

| Release | Contenido | Objetivo |
|---------|-----------|----------|
| **v0.1 MVP** | Instalador Windows, HTTPS, login con refresh, productos, categorías, unidades, 1 almacén, entradas/salidas/**ajustes**, existencias, reporte básico, **importación y exportación Excel**, **respaldos automáticos y restauración** | Usable en producción en un cliente real |
| **v0.5** | Multi-almacén, traspasos, alertas de stock mínimo, permisos granulares, auditoría, impresión de reportes, Event Log de Windows, **actualizador** | Cubre la operación diaria completa |
| **v1.0** | Multi-empresa con RLS, dispositivos, etiquetas, reportes avanzados, kardex valorizado, integración con Active Directory (opcional), licenciamiento | **Producto vendible a PYMES** |
| **Futuro** | Sincronización a nube (patrón outbox), lotes y caducidad, app móvil | Expansión |

**Estrategia:** cada release es funcional y usable. No se avanza al siguiente hasta que el anterior esté estable en un entorno Windows real.

---

## 9. Reglas de Negocio Clave

- Todo movimiento afecta existencias y queda en el kardex con trazabilidad completa.
- **No se permite stock negativo**: constraint en BD, más bloqueo de filas (`SELECT ... FOR UPDATE`) o `UPDATE ... WHERE cantidad >= X` para evitar carreras.
- Cada operación es transaccional.
- Los traspasos son un documento con dos líneas en una sola transacción; las filas se bloquean siempre en el mismo orden para evitar deadlocks.
- Los productos no se borran, se desactivan.
- Los costos y precios se conservan en cada movimiento.
- El kardex es inmutable; las correcciones se hacen con movimientos inversos.

---

## 10. Fases de Desarrollo

| Fase | Entregable | Descripción |
|------|-----------|-------------|
| **0** | Prototipo de frontend en Windows | Reflex con login, tabla de 5,000 productos con búsqueda y edición en línea. Se valida también que corra como servicio de Windows sin Node.js instalado. Decisión: Reflex o plan B (HTMX) |
| **1** | Infraestructura base | Repositorio, `just`, CI en Windows y Linux, Docker (modo B), FastAPI con `/health` conectado a PostgreSQL 18, Caddy, prueba de `SelectorEventLoop` |
| **2** | Modelo de datos | Modelos SQLAlchemy y migraciones Alembic |
| **3** | Lógica de negocio | Servicios de inventario con pruebas de concurrencia |
| **4** | API REST | Endpoints, autenticación, permisos, importación Excel |
| **5** | Frontend | Pantallas operativas |
| **6** | Empaquetado Windows | Servicios WinSW, instalador Inno Setup, respaldos, actualización, firma de código |
| **7** | Piloto | Instalación en un cliente real y corrección de hallazgos antes de vender |

La Fase 6 se prototipa en versión mínima al terminar la Fase 1 (un instalador que levante solo `/health`), para descubrir problemas de Windows temprano y no en el cierre.

---

## 11. Principios de Desarrollo

1. **Simplicidad primero.**
2. **Un solo lenguaje:** Python en backend y frontend.
3. **La base de datos es la fuente de verdad.**
4. **Transaccionalidad** en operaciones críticas.
5. **Windows es ciudadano de primera clase:** cada funcionalidad se prueba en Windows, no solo en Linux.
6. **El cliente final no es técnico:** instalación, actualización y respaldo deben hacerse sin línea de comandos.
7. **Documentación automática:** Swagger UI en `/docs` (desactivado o protegido en producción).
8. **Sin dependencia de internet.**
9. **Preparado para crecer:** multi-empresa, multi-almacén y sincronización desde el diseño.
10. **Decisiones registradas** como ADR.

---

## 12. Operación y Respaldos

- **Respaldo diario** con `pg_dump` mediante el Programador de tareas, con rotación (por ejemplo, 7 diarios, 4 semanales, 6 mensuales).
- **Copia externa obligatoria:** ruta de red (NAS) o unidad USB; el instalador pide configurarla.
- **Cifrado** opcional de los respaldos.
- **Restauración** con un script y una guía; se prueba en una máquina limpia como criterio de éxito.
- **Respaldo automático antes de cada actualización.**
- **Recomendaciones de hardware:** SSD, UPS obligatorio, `fsync` activo, suspensión e hibernación desactivadas.
- **Monitoreo básico:** alerta visible en la aplicación si el último respaldo tiene más de 48 horas.

---

## 13. Documentación

- `README.md`: inicio rápido para desarrolladores
- `docs/01_arquitectura.md`
- `docs/02_modelo_datos.md`
- `docs/03_manual_instalacion_windows.md`: para técnicos de campo
- `docs/04_reglas_negocio.md`
- `docs/05_operacion_respaldos.md`
- `docs/06_guia_soporte_windows.md`: firewall, antivirus, certificados, energía, hostname
- `docs/decisiones/`: ADRs
- Swagger UI

---

## 14. Criterios de Éxito

- Un técnico instala el sistema en un **Windows 11 limpio en menos de 30 minutos** usando solo el instalador.
- Un desarrollador clona el repositorio y lo corre en menos de 30 minutos.
- Tras reiniciar el servidor Windows, **todos los servicios levantan solos** sin iniciar sesión.
- El sistema opera **sin internet** en red local, con HTTPS válido en Edge y Chrome.
- Un administrador gestiona múltiples almacenes desde una sola terminal.
- Toda operación queda auditada en el kardex.
- **50 salidas simultáneas** sobre el mismo producto nunca producen stock negativo ni descuadre.
- La suma de movimientos siempre coincide con las existencias.
- **Restauración de respaldo verificada** en una máquina limpia.
- Una actualización de versión conserva los datos y es reversible desde el respaldo previo.
- La suite de pruebas pasa en Windows y Linux en CI.

---

## 15. Registro de Cambios v1.0 → v1.1

| Área | Cambio | Motivo |
|------|--------|--------|
| Plataforma | Windows como plataforma principal; modo nativo sin Docker en producción | Es el entorno del cliente objetivo; evita dependencias frágiles |
| Instalación | Instalador `.exe` (Inno Setup + WinSW) | El cliente no es técnico |
| Arquitectura | Reflex consume FastAPI por HTTP; Caddy como único punto de entrada | Una sola lógica de negocio, HTTPS en LAN |
| Frontend | Fase 0 de validación con plan B (HTMX) | Reflex es joven y las tablas grandes son un riesgo |
| Modelo | Documentos + líneas; kardex append-only; existencias derivadas; decimales; unidades; costo promedio | Integridad y trazabilidad |
| Modelo | Campos de sincronización desde el inicio | Habilitar la nube sin rediseño |
| Seguridad | Refresh tokens revocables, Argon2id, permisos granulares, límite de intentos, RLS en v1.0 | Producto comercial |
| Operación | Respaldos y restauración en v0.1 | Ningún cliente perdona perder datos |
| Herramientas | `just` en lugar de `Makefile`; `uv`; `ruff`; CI en Windows y Linux | Compatibilidad con Windows |
| Estructura | Backend por módulos de dominio; carpeta `deploy/windows` | Mantenibilidad |
| Alcance | Ajustes e importación Excel pasan a v0.1 | Necesarios para operar en producción |
| Fases | Nueva Fase 0 y Fase 7 (piloto) | Reducir riesgo |

---

## 16. Supuestos y Decisiones Pendientes

**Supuestos tomados:**

1. "Sincronización con Windows" se interpreta como que el producto se instala y opera de forma nativa en Windows (servidor y clientes) y se integra con su ecosistema. La sincronización a la nube queda como extensión futura.
2. Costo promedio ponderado como método de valuación.
3. Reflex sujeto al resultado de la Fase 0.

**Pendientes por definir con el negocio:**

- Modelo de licenciamiento (por empresa, por almacén, por usuario) y cómo se valida sin internet.
- Presupuesto de certificado de firma de código.
- Versiones mínimas de Windows que se prometerán comercialmente.
- Necesidad real de Active Directory en los primeros clientes.
- Giro de los primeros clientes (si requieren lotes y caducidad antes de lo previsto).

---

## 17. Estado Actual y Siguiente Paso

**Estado:** Documento v1.1 listo para aprobación.
**Siguiente paso:** **Fase 0** (prototipo de Reflex en Windows) y, en paralelo, **Fase 1** (infraestructura base con Docker para desarrollo y FastAPI con `/health`).
