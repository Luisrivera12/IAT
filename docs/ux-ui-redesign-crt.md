# Rediseño UX/UI — Administrador CRT orientado a validación y picking

## Objetivo
Reordenar la experiencia actual para que la tarea principal en piso de venta sea:

1. Tomar CRT priorizado.
2. Ejecutar picking por artículo con confirmación.
3. Cerrar con estado final o incidencia.

Sin romper el backend actual, se propone un **refactor incremental** de frontend con dos vistas por rol.

---

## Arquitectura de información (target)

### Modos de operación

- **Modo Piso (default picker):**
  - Cola de trabajo priorizada por promesa.
  - CTA principal por CRT: `Iniciar picking` / `Continuar`.
  - Progreso visible por CRT (`x/y artículos`).

- **Modo Admin:**
  - Tabla completa actual (auditoría/facturación/transacción).
  - Herramientas de historial, reclamos y edición de fechas.

> Principio aplicado: progressive disclosure (primario operativo, secundario administrativo).

---

## Estructura exacta de componentes (incremental)

## 1) Shell principal

- `CrtPage`
  - `StickyHeaderMetrics` (se conserva)
  - `ModeToggle` (`Piso | Admin`) **nuevo**
  - `FilterBar` (se conserva + mejoras)
  - `StatusTabs` (se conserva con mapeo de etiquetas)
  - `MainContent`
    - si `mode === piso`:
      - `WorkQueueList` (tabla reducida o tarjetas)
      - `CrtDetailPanel` (con `PickListSection`)
    - si `mode === admin`:
      - `AdminDataTable` (tabla actual)
      - `CrtDetailPanel` (actual + secciones extendidas)

## 2) ModeToggle (nuevo)

**Ubicación:** debajo de métricas, encima de filtros.

**Estados:**
- `piso`
- `admin`

**Persistencia:** `localStorage['crt_ui_mode']`.

**Accesibilidad:**
- teclado con flechas/Tab
- `aria-pressed` en cada opción
- foco visible

## 3) FilterBar (evolución)

Conservar controles actuales y añadir:
- atajo `/` para foco en búsqueda principal
- Enter aplica filtros
- chips activos de filtros (`Estado: Pendiente`, `Fecha: Hoy`)

**No romper API:** seguir enviando mismos parámetros actuales.

## 4) StatusTabs (mapeo visual)

Mapeo solo UI (sin cambio backend):
- `Generada` → `Pendiente`
- `Planificada` → `En picking`
- `Cancelada` → `Cancelado`
- `Con reclamo` / flag → `Incidencia`
- `Listo` (derivado por progreso completo o estado actual equivalente)

## 5) WorkQueueList (modo piso)

Columnas mínimas sugeridas:
- CRT ID
- Cliente
- Entrega cliente (promesa)
- Artículos (total)
- Progreso (`recogidos/total`)
- Estado operativo
- Acción primaria (`Iniciar`/`Continuar`)

Reglas:
- Orden default: promesa ascendente.
- Badge de urgencia:
  - rojo: vencido
  - ámbar: hoy
  - gris: futuro
- Acciones secundarias a menú `⋯`.

## 6) AdminDataTable (modo admin)

Conservar tabla actual con:
- columnas administrativas completas
- acciones masivas actuales
- panel de detalle lateral existente

Mejora sugerida:
- selector mostrar/ocultar columnas y persistencia por usuario.

## 7) CrtDetailPanel (refactor del panel lateral)

Secciones:
1. `Resumen CRT` (cliente, promesa, origen/destino)
2. `Timeline` (actual)
3. `PickListSection` **nuevo núcleo en modo piso**
4. `Incidencias/Reclamos`
5. `Transacción/Facturación` (solo admin o colapsado)

### 7.1) PickListSection (nuevo)

Cada ítem:
- checkbox / estado
- SKU
- nombre
- cantidad requerida
- cantidad recogida
- acciones rápidas: `Marcar`, `Faltante`, `Dañado`

Resumen sección:
- progreso total CRT
- botón `Finalizar picking` (habilita cuando regla se cumple)

**Regla de cierre MVP:**
- habilitar cierre si todos los ítems están completos **o** existe incidencia registrada por cada faltante.

---

## Flujos de usuario (MVP)

## Flujo A — Iniciar picking
1. Usuario abre modo Piso.
2. Ve CRT sugerido (promesa más cercana + no finalizado).
3. Click en `Iniciar picking`.
4. Panel lateral abre `PickListSection` con foco en primer ítem.

## Flujo B — Marcar artículo
1. Selecciona ítem.
2. Acción `Marcar` incrementa recogidos.
3. Progreso se actualiza en panel y lista principal en tiempo real.

## Flujo C — Incidencia
1. En ítem, usuario elige `Faltante` o `Dañado`.
2. Se registra incidencia mínima (tipo + nota opcional).
3. CRT pasa a estado operativo `Incidencia`.

## Flujo D — Finalizar
1. Usuario pulsa `Finalizar picking`.
2. Validación de regla de cierre.
3. Estado cambia a `Listo` (o mantiene `Incidencia` según negocio).

---

## Contrato de datos UI (sin romper backend)

ViewModel frontend por CRT:

```ts
interface CrtOperationalVM {
  crtId: string;
  cliente: string;
  promesaEntrega: string;
  estadoBackend: 'GENERADA' | 'PLANIFICADA' | 'CANCELADA' | string;
  estadoOperativo: 'Pendiente' | 'En picking' | 'Listo' | 'Incidencia' | 'Cancelado';
  totalItems: number;
  pickedItems: number;
  urgentLevel: 'overdue' | 'today' | 'future';
}
```

Derivaciones frontend:
- `estadoOperativo` calculado por estado backend + incidencias + progreso.
- `pickedItems/totalItems` desde artículos existentes (o mock si aún no viene completo).

---

## Plan de implementación por sprint

## Sprint 1 (alto impacto)
- `ModeToggle`
- `WorkQueueList` reducido para piso
- orden por promesa
- chips de filtros activos

## Sprint 2
- `PickListSection` en panel lateral
- acciones por ítem (`Marcar`, `Faltante`)
- progreso sincronizado lista/panel

## Sprint 3
- menú `⋯` para acciones secundarias
- mostrar/ocultar columnas en Admin
- atajos teclado (`/`, `Enter`, `Esc`) + hardening de foco

---

## Criterios de aceptación

- Picker abre CRT y marca 1 ítem en < 10 segundos sin modal.
- Se identifica siguiente CRT de un vistazo por promesa + urgencia.
- Flujo completo usable por teclado con foco visible.
- Modo Admin conserva auditoría existente sin regresión.

---

## Riesgos y mitigación

- **Riesgo:** datos de artículos incompletos en endpoint actual.
  - **Mitigación:** VM con fallback y carga diferida por CRT.

- **Riesgo:** resistencia al cambio por usuarios administrativos.
  - **Mitigación:** modo Admin intacto por defecto para ese perfil.

- **Riesgo:** regresión en acciones masivas.
  - **Mitigación:** pruebas de regresión solo en modo Admin.

---

## Checklist técnico mínimo

- [ ] Toggle de modo persistente
- [ ] Orden default por promesa
- [ ] Progreso por CRT visible en lista
- [ ] Pick list por artículo en panel lateral
- [ ] Registro de incidencia por ítem
- [ ] Cierre de picking con validación
- [ ] Atajos `/`, `Enter`, `Esc`
- [ ] `:focus-visible` consistente en botones/inputs
