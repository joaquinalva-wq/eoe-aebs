# Sistema EOE — Austin eco bilingual school
## Contexto para Claude Code

---

## PRINCIPIO FUNDAMENTAL

Este es un sistema **single-file HTML** (~535 KB). Todo vive en `index.html`:
CSS + HTML + JS vanilla + Firebase SDK via CDN. **No hay build tools, no hay
frameworks, no hay npm.** Cada cambio es una edición quirúrgica sobre ese archivo.

**Regla de oro:** mantener la estructura simple. Antes de agregar algo, preguntar si
se puede resolver con lo que ya existe. No introducir dependencias externas nuevas
sin necesidad real.

---

## STACK TÉCNICO

| Capa | Tecnología |
|------|-----------|
| Frontend | HTML + CSS + JS vanilla (ES6+) |
| Base de datos | Firebase Firestore (v8 compat SDK) |
| Auth | Firebase Auth (email/password) |
| IA | Anthropic API `/v1/messages` (claude-sonnet-4-20250514) |
| Hosting | Netlify (deploy arrastrando el archivo) / GitHub Pages |
| Dependencias externas | Firebase SDK 8.10.1 via CDN únicamente |

**Firebase config embebida en el archivo:**
```js
apiKey: "AIzaSyAqJhEjbLu_hi-iclbdC1vbf33geQL2oYM"
authDomain: "eoe-aebs.firebaseapp.com"
projectId: "eoe-aebs"
```

**Usuarios del sistema:**
- `eoe@austinebs-ah.edu.ar` / `eoeaustin123`
- `joaquin.alva@austinebs-ah.edu.ar` / `Austin123`

---

## IDENTIDAD DE MARCA — AUSTIN EBS

**Paleta exacta (variables CSS):**
```css
--brand-orange:  #F58634;   /* cuadrado A del logo — naranja */
--brand-green:   #A9CF46;   /* cuadrado e/b — verde lima */
--brand-slate:   #3B5C6A;   /* cuadrado s — azul pizarra */
--brand-navy:    #293F54;   /* wordmark Austin — azul navy */
```

**Derivadas de interfaz:**
```css
--accent:        #3B5C6A;   /* = brand-slate, acento principal */
--accent-light:  #E8EFF2;
--orange:        #F58634;
--orange-light:  #FEF0E2;
--lime:          #A9CF46;
--lime-light:    #EEF7D6;
```

**Reglas de diseño:**
- Topbar: fondo `--brand-navy`, logo Austin en contenedor blanco con `border-radius:8px`
- Sidebar: ítem activo tiene franja naranja `--brand-orange` a la izquierda con `::before`
- Badges/notificaciones: naranja Austin
- Tabs activos: línea naranja inferior (`--brand-orange`)
- Modales: borde superior naranja `3px solid var(--brand-orange)` en `.modal-header`
- Botón primario: `--brand-slate` con hover `--brand-navy`
- Botón de acción importante: `--brand-orange` (btn-orange)
- Día actual en calendario: círculo `--brand-orange`
- **NUNCA** usar azules genéricos hardcodeados (#1a3f8f, #2a6abf, etc.)

---

## ARQUITECTURA DEL ARCHIVO

```
index.html
├── <style>          CSS completo (~600 líneas)
│   ├── :root        Variables CSS (paleta, spacing, tipografía)
│   ├── Login        Pantalla de ingreso
│   ├── Topbar       Barra superior navy
│   ├── Sidebar      Navegación lateral
│   ├── Pages        Páginas (.page, .page.active)
│   ├── Modales      .modal-overlay, .modal
│   ├── Componentes  Cards, tables, badges, tabs, forms
│   └── Calendario   Grid mensual
│
├── <body>
│   ├── #login-screen    Pantalla pública
│   └── #app             App autenticada
│       ├── .topbar
│       ├── .sidebar      Navegación
│       ├── .content      Páginas (solo .active es visible)
│       │   ├── #page-dashboard
│       │   ├── #page-estudiantes
│       │   ├── #page-cursos
│       │   ├── #page-calendario
│       │   ├── #page-protocolos      (tabs: situaciones / diagnósticos)
│       │   ├── #page-seguimiento     (layout 2 col: buscador | form nota)
│       │   ├── #page-emergencias     (historial observaciones urgentes)
│       │   ├── #page-matriculados    (integración admisiones)
│       │   └── #page-carga-masiva
│       ├── Modales       (fuera del flow, z-index alto)
│       └── #fab-observacion   Botón flotante naranja (solo logueado)
│
└── <script>         JS completo (~3500 líneas)
    ├── Firebase init
    ├── Auth listener (onAuthStateChanged)
    ├── initApp()
    ├── showPage(page)
    ├── Módulos por área (ver sección JS)
    └── Constantes y datos (ESTUDIANTES_PRECARGADOS, etc.)
```

---

## NAVEGACIÓN Y ESTADO

```js
// Cambiar página
showPage('estudiantes');   // activa .page#page-estudiantes y nav-item

// Abrir/cerrar modal
openModal('modal-nuevo-estudiante');
closeModal('modal-nuevo-estudiante');

// Modales disponibles:
// modal-nuevo-estudiante, modal-ficha, modal-trayectoria,
// modal-nuevo-protocolo, modal-alta-matriculado,
// modal-emergencia, modal-marcar-realizado,
// modal-nueva-alerta, modal-ver-alerta
```

**Patrón de navegación:**
```js
// showPage integra hooks para cargar datos on-demand
function showPage(page) {
  // activa page y nav-item
  if(page==='cursos') renderCursos();
  if(page==='emergencias') { loadEmergencias(); renderHistorialEmergencias(); }
  if(page==='matriculados') loadMatriculados();
  // etc.
}
```

---

## COLECCIONES FIRESTORE

| Colección | Descripción | Índices necesarios |
|-----------|-------------|-------------------|
| `estudiantes` | 666 estudiantes activos | `apellido` ASC |
| `notas` | Notas de seguimiento por estudiante | `estudianteId` ASC + `fecha` DESC |
| `alertas-seguimiento` | Seguimientos del calendario | `estudianteId` ASC + `proximaFecha` ASC |
| `alertas-seguimiento/{id}/historial` | Ocurrencias completadas | `timestamp` DESC |
| `emergencias` | Observaciones urgentes | `timestamp` DESC |
| `protocolos` | Biblioteca de protocolos generales | `titulo` ASC |
| `protocolos-diag` | Protocolos por diagnóstico | `diagnostico` ASC |
| `matriculados-pendientes` | Integración desde admisiones | `estado` ASC + `timestamp` DESC |

**Documento `estudiantes` — estructura:**
```js
{
  apellido, nombre, nivel, curso, division, estado,
  nacimiento, email, diagnosticos: [], observaciones,
  familia: { padre: {nombre,email,tel,ocupacion}, madre: {...}, observaciones },
  pago, catering, transporte,
  anioIngreso, origenAdmisiones, matriculadoId,
  createdAt, updatedAt, createdBy
}
// PDFs en subcolección: estudiantes/{id}/documentos
// {nombre, base64, tamaño, tipo, tipoLabel, icon, fecha, subidoPor}
```

---

## VARIABLES GLOBALES CLAVE

```js
let allEstudiantes = [];        // todos los estudiantes (cargados con onSnapshot)
let todasLasAlertas = [];       // alertas/seguimientos (con caché 5min)
let todasLasEmergencias = [];   // observaciones urgentes
let todosLosMatriculados = [];  // pendientes de admisiones
let protocolos = [];            // protocolos generales
let currentUser = null;         // usuario Firebase Auth actual
let seguimientoEstudianteId = null;  // estudiante seleccionado en seguimiento
let docFileData = null;         // archivo pendiente de subir en ficha

// Cachés de performance
let _alertasCache = null;
let _alertasCacheTs = 0;
const ALERTAS_CACHE_TTL = 5 * 60 * 1000;  // 5 minutos

const _notasCache = {};  // { [estId]: { notas, ts } } — TTL 2 minutos
```

---

## MÓDULOS JS PRINCIPALES

### Estudiantes
```js
loadEstudiantes()          // onSnapshot, excluye pdfBase64 de memoria
renderTablaEstudiantes()   // tabla con filtros activos
verFicha(id)               // abre modal-ficha con tabs
editarEstudiante(id)       // pre-llena modal-nuevo-estudiante
guardarEstudiante()        // crear/actualizar en Firestore
confirmarEliminar(id,nombre)
irASeguimiento(id)         // navega a seguimiento con estudiante pre-seleccionado
abrirAlertaRapida(id,nombre)  // abre modal-nueva-alerta pre-llenado
```

### Notas de seguimiento
```js
cargarHistorialNotas(estId)   // con caché 2min, llama a renderNotasHTML
renderNotasHTML(el, notas, estId)
guardarNota()
```

### Calendario / Alertas
```js
loadAlertas(force)         // .get() con caché, force=true invalida
renderCalendario()         // mini y completa
marcarRealizado(id, fechaOcurrencia)
confirmarMarcarRealizado() // lee modal-marcar-realizado, maneja solo/todos
renderHistorialRealizado() // en el calendario completo
```

### Observaciones urgentes (Emergencias)
```js
loadEmergencias()          // .get() on-demand
abrirObservacionUrgente(estId, estNombre)  // alias de abrirEmergencia
abrirEmergencia(estId, estNombre)
guardarEmergencia()        // guarda en emergencias + nota en estudiante + alerta auto
renderHistorialEmergencias()
renderDashUrgencias()      // muestra últimas 7 días en dashboard
```

### Protocolos
```js
loadProtocolos()           // onSnapshot, excluye pdfBase64 (_hasPdf flag)
renderProtocolos()
guardarProtocolo()         // maneja protocolos generales y diagnósticos
abrirNuevoProtocolo()
editarProtocolo(id)
actualizarPreviewProtocolo()  // preview en tiempo real (2 columnas)
renderDiagLib()            // tab diagnósticos
```

### Matriculados (Admisiones)
```js
loadMatriculados()
renderTablaMatriculados()
abrirAltaMatriculado(matriculadoId)
confirmarAltaMatriculado()  // crea estudiante + marca dado de alta en admisiones
```

### Trayectoria
```js
abrirTrayectoria(estId, est)  // timeline notas + observaciones en paralelo
```

### Documentos IA
```js
previewDocumento(event)    // acepta PDF, Word, JPG/PNG — muestra panel IA
analizarDocumentoIA(file, base64Data)  // llama a Anthropic API
subirDocumento()           // guarda en subcolección documentos
```

### Carga masiva
```js
cargarEstudiantesPrecargados()  // batch de 666 estudiantes en Firestore
handleCSV(event)           // importar CSV con preview
```

---

## OPTIMIZACIONES DE FIRESTORE (MANTENER)

1. **`loadEstudiantes`** usa `onSnapshot` pero excluye `pdfBase64` de los docs en memoria
2. **`loadAlertas`** usa `.get()` con caché de 5 minutos — `loadAlertas(true)` para forzar
3. **`loadEmergencias`** usa `.get()` on-demand (no tiempo real)
4. **`loadProtocolos`** usa `onSnapshot` pero excluye `pdfBase64` (_hasPdf flag)
5. **Notas** tienen caché de 2 minutos por estudiante (`_notasCache`)
6. **PDFs** siempre en subcolecciones, nunca en el documento principal
7. **Escrituras** con batch cuando son múltiples documentos

---

## REGLAS FIRESTORE (PRODUCCIÓN)

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
    // Admisiones puede crear sin auth
    match /matriculados-pendientes/{docId} {
      allow create: if true;
      allow read, update, delete: if request.auth != null;
    }
  }
}
```

---

## INTEGRACIÓN ADMISIONES → EOE

El sistema de admisiones hace POST a Firebase REST:
```
POST https://firestore.googleapis.com/v1/projects/eoe-aebs/databases/(default)/documents/matriculados-pendientes
Content-Type: application/json
// Sin Authorization (permitido por reglas)
```

Campos esperados: `nombre`, `apellido`, `nivel`, `curso`, `division`,
`anioIngreso`, `email`, `telefono`, `nombrePadre`, `emailPadre`,
`nombreMadre`, `emailMadre`, `fechaNacimiento`, `observaciones`,
`estado: "Pendiente de alta"`, `timestamp` (ms Unix).

El EOE muestra estos registros en **🎒 Matriculados** y permite dar de alta
con modal pre-llenado.

---

## PATRONES DE CÓDIGO A RESPETAR

### Agregar una nueva página
```html
<!-- En sidebar -->
<button class="nav-item" onclick="showPage('mi-pagina')" data-page="mi-pagina">
  <span class="nav-icon">📋</span> Mi Página
</button>

<!-- En .content -->
<div class="page" id="page-mi-pagina">
  <div class="page-header">
    <div><h1>Título</h1><p>Descripción</p></div>
  </div>
  <!-- contenido -->
</div>
```
```js
// En showPage()
if(page==='mi-pagina') cargarDatosMiPagina();
```

### Agregar un modal
```html
<div class="modal-overlay" id="modal-mi-modal">
  <div class="modal" style="max-width:600px;">
    <div class="modal-header">
      <h2>Título</h2>
      <button class="btn btn-ghost btn-icon" onclick="closeModal('modal-mi-modal')">✕</button>
    </div>
    <div class="modal-body"><!-- contenido --></div>
    <div class="modal-footer">
      <button class="btn btn-secondary" onclick="closeModal('modal-mi-modal')">Cancelar</button>
      <button class="btn btn-primary" onclick="miAccion()">Confirmar</button>
    </div>
  </div>
</div>
```

### Agregar JS nuevo
Siempre antes de `</script>`, en un bloque comentado claro:
```js
// ===================================================================
// ===== NOMBRE DEL MÓDULO ==========================================
// ===================================================================
```

### Toast de feedback
```js
toast('Mensaje de éxito');           // verde
toast('Error', 'error');             // rojo
```

---

## VERIFICACIÓN ANTES DE ENTREGAR

Siempre verificar que el JS no tiene errores de sintaxis:
```bash
# Extraer el bloque JS del HTML y verificar con Node
python3 -c "
c=open('index.html').read()
start=c.find('<script>\n// ===== FIREBASE CONFIG')
end=c.rfind('</script>')
open('test.js','w').write(c[start+8:end])
"
node --check test.js
```

**Errores comunes a evitar:**
- `${{ ... }[key]}` — objeto literal dentro de template literal → usar función auxiliar
- Template literals con saltos de línea reales y comillas simples → usar concatenación
- `const` redeclaradas en el mismo scope
- Overrides con `const _orig = fn; fn = function(){}` — en JS estricto esto falla
  → integrar la lógica directamente en las funciones originales

---

## ESTADO ACTUAL DEL SISTEMA (Junio 2026)

**Archivo principal:** `austin-ebs-v5.html` (~4423 líneas)

**Funcionalidades completas:**
- ✅ Login con Firebase Auth (multi-usuario, cada psicóloga ve lo mismo)
- ✅ Dashboard con calendario mini, stats, notificaciones Admisiones y Altas
- ✅ Listado de 666 estudiantes con filtros (nivel, estado, diagnóstico, curso)
- ✅ Ficha completa del estudiante (tabs: datos, familia, salud, documentos, info)
- ✅ Documentos adjuntos en ficha (PDF, Word, JPG — hasta 10MB + análisis IA)
- ✅ Trayectoria completa del estudiante (timeline cronológico)
- ✅ Calendario de seguimientos (mini + completo, recurrentes, historial)
- ✅ Marcar realizado (solo este / todos los futuros)
- ✅ Registro de notas de seguimiento (layout 2 columnas)
- ✅ Biblioteca de protocolos (tabs: situaciones / diagnósticos, PDF adjunto, preview live, autor/fecha)
- ✅ Carga masiva CSV + botón carga inicial 666 estudiantes
- ✅ **Revisión de Admisiones** (`page-admisiones`): EOE aprueba/rechaza casos enviados por Admisiones. Colección `admisiones-revision`. Badge rojo en sidebar cuando hay pendientes.
- ✅ **Altas / Matriculados** (`page-matriculados`): Matriculados de Admisiones pendientes de alta en EOE. EOE crea el estudiante en el sistema. Tab "Dados de alta" con botón "Pasar de año" individual. Colección `matriculados-pendientes`.
- ✅ Identidad de marca Austin EBS aplicada en todo el sistema

**Colecciones Firestore (actualizado):**
| Colección | Descripción |
|-----------|-------------|
| `estudiantes` | 666+ estudiantes activos |
| `notas` | Notas de seguimiento |
| `alertas-seguimiento` | Seguimientos del calendario |
| `emergencias` | Observaciones urgentes |
| `protocolos` | Protocolos generales |
| `protocolos-diag` | Protocolos por diagnóstico |
| `admisiones-revision` | **NUEVO** Casos de admisiones para revisión EOE (estado: pendiente/aprobado/rechazado) |
| `matriculados-pendientes` | Matriculados desde Admisiones (estado: "Pendiente de alta"/"Dado de alta") |

**Flujo Admisiones → EOE:**
1. Admisiones crea doc en `admisiones-revision` con `estado:"pendiente"`
2. EOE ve notificación en dashboard y badge rojo en sidebar
3. EOE abre "Revisión EOE", aprueba o rechaza (con motivo)
4. El doc queda con `estado:"aprobado"/"rechazado"`, `revisadoPor`, `fechaRevision`

**Flujo Altas / Matriculados:**
1. Admisiones crea doc en `matriculados-pendientes` con `estado:"Pendiente de alta"`
2. EOE ve notificación en dashboard y badge en sidebar
3. EOE abre "Dar de alta", revisa datos, confirma
4. Se crea estudiante en `estudiantes` + se marca `estado:"Dado de alta"` con `estudianteId`
5. En tab "Dados de alta" aparece el botón "⬆️ Pasar de año" para promoción individual

**Pendientes / mejoras posibles:**
- Índice Firestore para `admisiones-revision`: `timestamp DESC`
- Índice Firestore para `notas`: `estudianteId ASC + fecha DESC`
- Integración real-time con onSnapshot en Admisiones (actualmente usa .get() on-demand)

---

## DEPLOY

1. Editar `index.html` localmente
2. Arrastrar a Netlify → Deploys (sitio: https://eoe-austin.netlify.app)
3. O hacer commit al repo GitHub privado → GitHub Pages lo publica solo

**Cuando GitHub Pages esté activo:**
- El archivo debe llamarse `index.html`
- Settings → Pages → Branch: main → / (root)
