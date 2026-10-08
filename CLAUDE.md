# CLAUDE.md — Proyecto IMAX CALI

> Este archivo lo lee Claude Code automáticamente al iniciar.
> Contiene el contexto y las reglas de trabajo del proyecto. Mantenerlo actualizado.

---

## Qué es este proyecto

PWA (Progressive Web App) de un solo archivo para la operación de **IMAX CALI**
(venta y servicio de equipos iPhone). Nació como copia de Skadii (SKADIICELL S.A.S.)
y hoy es un proyecto **independiente**. Toda la aplicación vive en **`index.html`**
(~6.470 líneas): HTML + CSS + JavaScript puro, sin framework ni build step.

- **Sin frameworks**: JavaScript vanilla. No hay React, Vue, ni bundlers.
- **Sin compilación**: el archivo se sirve tal cual. No hay `npm build`, no hay `dist/`.
- **Backend = Google Drive + Google Sheets** vía OAuth (Google API JS + GIS).
  Cada documento se guarda como JSON en su carpeta de Drive y se registra en un spreadsheet.
- **Repositorio**: `davidpar146/imax-cali`. Se publica con GitHub Pages en
  **https://davidpar146.github.io/imax-cali/** (se actualiza solo al mergear a `main`).

---

## Independencia respecto a Skadii (muy importante)

- Este proyecto es **IMAX CALI**. Trabajar SOLO en el repositorio `davidpar146/imax-cali`.
  **Nunca** leer, editar ni hacer push al repositorio `davidpar146/Skadii`, ni copiar
  cambios de un proyecto al otro salvo que el usuario lo pida expresamente.
- Las carpetas de Drive (`FOLDER_IDS`, `CONFIG_FILES`) son las de IMAX CALI.
  Nunca poner IDs de carpetas de Skadii.
- Ambas páginas viven en el mismo dominio (`davidpar146.github.io`) y comparten el
  `localStorage` del navegador. Por eso **toda clave de `localStorage` de este proyecto
  lleva el prefijo `imax_`** (p. ej. `imax_spreadsheet_id`). Cualquier clave nueva debe
  llevarlo también, o se mezclarían los datos con Skadii.

---

## Regla de oro

**NO romper lo que ya funciona.** Este es un sistema en producción real.

- Hacer **cambios quirúrgicos y aislados**, no reescrituras.
- Un cambio a la vez; explicar qué se tocó y por qué.
- Preferir **funciones e IDs nuevos con prefijo propio** antes que modificar lógica existente.
- Cuando se reutilice lógica existente (p. ej. la lista de prestadores), **reutilizar los
  helpers compartidos** en lugar de duplicar datos.
- Ante la duda, preguntar antes de editar.

---

## Arquitectura de módulos

La app se organiza por módulos, cada uno con su prefijo de funciones e IDs.
Respetar SIEMPRE el prefijo del módulo al añadir código:

| Módulo               | Prefijo JS / IDs | Qué hace |
|----------------------|------------------|----------|
| Remisiones           | (base) `item-`, `payment-`, `estado-` | Venta: consecutivo, ítems propios/prestados, pagos, fotos, OCR IMEI, código de barras |
| Mantenimientos       | `mant…`          | Diagrama de daños sobre SVG del iPhone, firma, estados |
| Garantías / Cambios  | `gar…`           | Cambio de equipo, diferencia de precios, autorización por clave |
| Pagar Prestadores    | `prest…`         | Agrupa equipos prestados pendientes y registra pagos |
| Ingresos Pendientes  | `ing…`           | Cobros a empresas financieras |
| Inventario / Gastos  | `inv…`, `gas…`   | Inventario y gastos con categorías y aprobaciones |

### Convenciones de naming ya en uso
- Selector de prestadores en Remisiones: `prest-chips-list`, `item-prestado-por`, `prestadoresLista` (lista compartida).
- Selector de prestadores en Garantías (equipo que sale): `gar-sale-…` (reutiliza `prestadoresLista`).
- Modales de éxito: `success-modal` (remisiones), `gar-success-modal` (garantías).

---

## Integraciones con Google (puntos sensibles)

- **OAuth**: `gapi` (Google API) + `google.accounts` (GIS). La inicialización está al final
  del script, en `window.addEventListener('load', …)`. No tocar el flujo de auth salvo necesidad.
- **Sesión de Google (token)**: dura ~1 hora. `sesionRegistrarExpiracion()` guarda cuándo vence;
  `sesionRevisar()` (cada 30 s y al volver a la pestaña) muestra el aviso `sesion-aviso` 5 min antes,
  y `sesionRenovar()` renueva con un clic sin recargar. `subirArchivoDrive`/`subirImagenDrive`
  muestran el aviso si Drive responde 401. Cualquier flujo nuevo que llame a Google debe tolerar esto.
- **Cliente OAuth propio**: `GOOGLE_CLIENT_ID` pertenece al proyecto de Google Cloud **IMAX CALI**
  (ID `imax-cali`), ya no al heredado de Skadii. La app está en modo "Prueba": solo entran los
  correos agregados en Google Auth Platform → Público → Usuarios de prueba (máx. 100). Origen
  autorizado: `https://davidpar146.github.io`.
- **Carpetas de Drive**: definidas en el objeto `FOLDER_IDS`. Cada tipo de documento tiene su carpeta.
  No cambiar estos IDs sin confirmar con el usuario; apuntan a carpetas reales en producción.
- **Asesores y permisos**: salen SOLO del archivo de claves (`CONFIG_FILES.clavesAsesores`,
  `CLAVES-ASESORES-V2` en la carpeta principal). No escribir nombres de asesores en el código
  (`ASESORES_FIJOS` queda vacío a propósito).
- **Prestadores**: viven en `PRESTADORES-LISTA.json` (carpeta `prestamosPagos`); se agregan con
  "+ OTRO". `PRESTADORES_INICIALES` (hoy `JHONATAN`, `SIRLEY`, pedido por el usuario) solo
  siembra la lista si ese archivo no existe o está vacío.
- **Mantenimientos — origen del equipo**: al iniciar se elige `mantSetOrigen('imax' | 'cliente')`.
  IMAX oculta los datos y la firma del cliente (`mant-card-cliente`, `mant-card-firma`) y guarda
  `origen: 'imax'`, `cliente.nombre: 'IMAX CALI'`, `firma_cliente: null`. Todo el formulario está en
  `mant-form-cuerpo` (oculto hasta elegir). El botón Guardar se bloquea mientras sube (`mantGuardando`).
- **Mantenimientos — lista**: `mantAgruparLista()` separa En mantenimiento (abiertos sin novedades),
  Con novedad (abiertos con novedades), Finalizados de los últimos `MANT_DIAS_FINALIZADOS` (7) días
  y Finalizados anteriores (plegados en `<details>`). La fecha de cierre la da `mantFechaCierre()`.
  `mantQuitarDuplicados()` muestra una sola copia por número (no toca Drive).
- **Catálogo de accesorios**: `ACCESORIOS_CATALOGO` / `ACC_CATEGORIAS` (solo lo usa Remisiones).
  Los precios no van en el catálogo; se digitan en cada venta.
  El archivo se busca **por nombre** (`CLAVES-ASESORES…`, el más reciente) en la carpeta principal
  con `clavesBuscarArchivoId()`, porque al resubirlo a Drive cambia de ID; el ID de
  `CONFIG_FILES` es solo respaldo. `clavesParsear()` tolera comas faltantes/sobrantes.
- **Subida de archivos**: helpers `subirArchivoDrive(...)` y `subirImagenDrive(...)`.
- **Registro en Sheets**: `agregarFilaExcel(...)`.
- **Compatibilidad entre módulos**: el módulo "Pagar Prestadores" detecta archivos cuyo nombre
  empieza por `Prestado-` en la carpeta `prestamosPagos`, con `estado: 'pendiente'` y un array
  `equipos[]`. Cualquier registro nuevo de equipo prestado DEBE seguir ese mismo formato para que
  el reporte de pagos lo recoja. (Ver `registrarPrestadosEnDrive` y `garRegistrarPrestadoEntregado`.)

---

## Flujo de trabajo con Git (importante)

Como toda la app es un único archivo crítico:

1. **Trabajar en una rama**, nunca directo en `main`:
   ```
   git checkout -b ajuste/<descripcion-corta>
   ```
2. Aplicar el cambio en `index.html`.
3. Revisar SIEMPRE el diff antes de confirmar:
   ```
   git diff
   ```
4. Commit con mensaje claro en español describiendo el ajuste.
5. Abrir Pull Request hacia `main`, mergear (squash) y borrar la rama.

### Política de automatización (acordada con el usuario)

El usuario pidió el flujo **totalmente automático**: el asistente ejecuta
solo los pasos 1–5 (rama → commit → push → PR → **merge a `main`** → limpieza
de la rama) sin pedir confirmación manual para cada paso. Aun así:

- Cada cambio se evalúa primero **dónde encaja** (dónde se agrega o reescribe)
  y se aplica de forma **quirúrgica y aislada**, sin tocar lo que ya funciona.
- Se respetan SIEMPRE las validaciones de la sección siguiente antes de mergear.
- El merge se hace por **squash** para mantener `main` limpio.
- Los PR/merge se gestionan por la **API de GitHub** usando las credenciales ya
  guardadas (Git Credential Manager); no hay `gh` instalado en el equipo.

---

## Validaciones antes de dar por bueno un cambio

Antes de declarar terminado cualquier ajuste a `index.html`, verificar:

- [ ] **IDs únicos**: ningún `id="..."` nuevo duplica uno existente.
- [ ] **Etiquetas balanceadas**: `<div>`/`</div>`, `<button>`/`</button>`, `<script>`/`</script>`.
- [ ] **Sintaxis JS válida**: extraer el bloque `<script>` principal y pasar `node --check`.
- [ ] **onclick → función real**: cada `onclick="fn(...)"` nuevo apunta a una función definida.
- [ ] **No se rompió otro módulo**: confirmar que los IDs/funciones de los demás prefijos siguen intactos.
- [ ] **Reset de formularios**: si se añaden campos, agregarlos a la función de limpieza del módulo.

---

## Tono y estilo

- Comentarios y mensajes de UI en **español**.
- Mantener el esquema visual existente (variables CSS `--primary`, `--success`, `--danger`, etc.).
- No introducir dependencias externas nuevas sin confirmar.
