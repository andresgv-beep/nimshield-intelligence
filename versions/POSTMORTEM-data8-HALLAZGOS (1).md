# POST-MORTEM · Recuperación de `data8` y hallazgos de fiabilidad

**Fecha:** junio 2026 · **Versión NimOS:** Beta 9 alpha
**Alcance:** reparación de un pool BTRFS RAID1 + recuperación de shares perdidas
en la BD + análisis de resiliencia de la configuración.
**Resultado:** datos 100% recuperados (cero bytes perdidos). Pero el proceso
destapó una cadena de problemas sistémicos, todos de la misma familia: **NimOS
confía en su caché (SQLite) por encima de la realidad (btrfs/kernel/disco)**.

**Documento hermano:** `PLAN-FIABILIDAD-STORAGE-v2.2.md` (diseño de los fixes).
Este post-mortem es el registro de QUÉ salió y QUÉ aplicar; el plan es el CÓMO.

---

## 1 · Resumen ejecutivo

Lo que empezó como "reparar un pool degradado" terminó destapando que el modelo
de fiabilidad de NimOS tiene un patrón roto repetido en cinco capas distintas:
la BD se trata como fuente de verdad cuando debería ser un caché reconstruible
desde la realidad física. El incidente validó en hardware real el principio que
ya estaba escrito (Regla 16) pero aplicado de forma incompleta.

**Daño a datos:** ninguno. Los datos (pool + subvolúmenes + apps docker) viven en
los SSD en RAID1 y nunca estuvieron en riesgo.
**Daño a la experiencia:** alto. El usuario tuvo que reparar el pool por terminal,
diagnosticar a mano una pérdida de registros de la BD, y descubrir que el backup
de config no era de fiar — todo lo que un NAS debería hacer solo.

---

## 2 · Línea de tiempo (condensada)

1. Un `btrfs replace` se cortó por un reinicio de la Pi → pool con un devid
   fantasma MISSING (el disco viejo que el replace no llegó a soltar).
2. El pool no estaba en `/etc/fstab` → no automontó tras el reinicio.
3. La UI mostró el pool como **"single · 1 disco · HEALTHY"** mientras btrfs tenía
   3 devids, uno MISSING. Contradicción directa.
4. El botón "Mejorar a RAID1" lanzó `btrfs device add` sobre un pool **no
   montado** → `ERROR: not a btrfs filesystem`.
5. Reparación 100% manual: `device scan`, `mount -o degraded`, `device remove
   missing` (0 errores), verificación RAID1, alta en fstab.
6. Tras reconstruir el daemon, el pool se mostró **correctamente** (healthy, 2
   discos, raid1). Pool cerrado.
7. Nuevo síntoma: **las carpetas compartidas no aparecían.** El Panel mostraba 0.
8. Diagnóstico (con un desvío: consultamos la BD equivocada por un `find` amplio)
   → la BD real está en `/var/lib/nimos/config/nimos.db`; las tablas `shares` y
   `share_permissions` existen pero **vacías**. Los 3 subvolúmenes (`data11`,
   `multimedia`, `musica`, 3.8 GB) intactos en disco.
9. Análisis de raíz: el backup de config a pool es **WAL-unsafe** y probablemente
   restauró un estado viejo sin las shares → registros perdidos, datos a salvo.

---

## 3 · Catálogo de hallazgos (cada uno: qué · causa-raíz · solución)

### Familia A — La UI miente sobre el estado (caché vs realidad)

**A1 · Split-brain: la tarjeta del pool dice "HEALTHY" con un disco MISSING.**
- *Causa:* `ListPools` (`storage_service.go`) lee de SQLite; la salud se computa
  desde la caché (`enrichPool` → `storage_health.go`). El observer
  (`storage_observer.go`) calcula la verdad desde `btrfs filesystem show` y emite
  divergencias, pero **la tarjeta no las consume**. Dos cerebros sin reconciliar.
- *Solución:* **FIX-2** — `reconcileHealthWithDivergences()` tras el enrich: si el
  observer reporta una divergencia para el pool, la salud no puede ser `healthy`
  (la realidad solo empeora, nunca mejora). **✅ YA IMPLEMENTADO** (3 archivos,
  8 tests, suite verde) — pendiente de validar en la Pi.

**A2 · La salud `DEGRADED` se cuenta contra los discos que ESPERA la BD.**
- *Causa:* `storage_health.go` (~L405-466) cuenta `disk_missing` comparando la BD
  con la realidad. Una fila de disco fantasma → DEGRADED falso; o al revés.
- *Solución:* parte de **FIX-2** + módulo hermano (alinear la membresía de discos
  con `btrfs filesystem show`, purgar fantasmas de la BD).

### Familia B — Operar sobre estado equivocado

**B1 · Ops de layout sin verificar montaje → `not a btrfs filesystem`.**
- *Causa:* `AddDevice` / `ConvertProfile` / `ReplaceDevice` / `RemoveDevice`
  ejecutan su `btrfs ... <mountpoint>` sin comprobar montaje y **sin llamar a
  `assertPoolWritable`** (que ya existe y usa `files.go`).
- *Solución:* **FIX-1** — gate `assertPoolWritable`/`ensurePoolMounted` al inicio
  de las 4 ops. Error claro y accionable en vez del críptico de btrfs. Coste muy
  bajo (la función ya existe).

### Familia C — Existencia y persistencia del pool

**C1 · El pool no estaba en fstab → no automontó tras el reinicio.**
- *Causa:* `appendFstab` (`storage_startup.go:454`) **anexa ad-hoc** (sin
  marcadores de propiedad) y solo se llama en `CreatePool` e `import`. La ruta de
  reparación (replace/convert manual) no pasa por ahí → el pool quedó fuera.
  *Nota positiva:* ya incluye `nofail` (bien) y hay restauración de fstab tras
  crash (P5 en `storage_boot.go`).
- *Solución:* **FIX-4** — **generar** fstab desde la BD (no anexar), entre
  marcadores `# >>> [nimos]`, atómico al montar/crear/importar. (Lección de OMV.)

**C2 · `ListPools` es ciego a un pool sano que existe en disco pero no en la BD.**
- *Causa:* la existencia de un pool = tener fila en SQLite. Si la fila se pierde,
  el pool es invisible aunque btrfs lo tenga montado.
- *Solución:* **FIX-3** — no esconder huérfanos (el observer ya marca
  `DivOrphanFilesystem`; el `import` ya re-adopta) + **vía para reconciliar una
  fila corrupta** (resincronizar devices/profile desde btrfs) sin destruir. Hoy
  el import rechaza con `ALREADY_MANAGED` → callejón sin salida (mismo error que
  arrastra OMV).

### Familia D — Estado MISSING

**D1 · No existe un estado MISSING de pool de primera clase.**
- *Causa:* `PoolHealth.Status` solo admite `healthy|at_risk|unstable|degraded|
  critical`. Un pool cuyo disco entero no está se mete en `critical`,
  indistinguible de un RAID degradado. (El "missing" sí existe a nivel disco en
  `storage_health.go:680-720`, pero no llega a la tarjeta.)
- *Solución:* **FIX-5** — estado `missing` de tarjeta, gris, distinto de
  `degraded`, con texto accionable ("el disco <serial> no está presente").
  (Lección de OMV: nunca mostrar ausente como "healthy", nunca esconderlo.)

### Familia E — Pérdida de los registros de shares

**E1 · Confusión de diagnóstico: BD equivocada.**
- *Qué:* un `find /var/lib/nimos -name 'nimos*.db'` devolvió un fichero huérfano
  `/var/lib/nimos/nimos.db` (ruta vieja, sin `/config/`). La BD real es
  `const dbPath = "/var/lib/nimos/config/nimos.db"` (`db.go:20`). Estuvimos
  consultando el fichero equivocado un rato.
- *Solución:* **G1** (limpieza, abajo). Lección: diagnosticar siempre contra
  `dbPath` del código, no contra lo que devuelva un `find` amplio.

**E2 · Tablas `shares`/`share_permissions` existen pero VACÍAS; datos intactos.**
- *Qué:* 3 subvolúmenes reales en `/nimos/pools/data8/shares/` (`data11`,
  `multimedia` 3.5 G, `musica` 294 M) sin fila en la BD → invisibles en el Panel.
- *Causa:* registros perdidos, casi seguro por restauración de un backup de config
  WAL-stale (ver Familia F). Los datos sobrevivieron porque los subvolúmenes son
  físicos.
- *Solución inmediata:* re-adopción. NimOS ya está preparado:
  `createBtrfsSubvolIfMissing` (`shares_service.go:184`) **"si ya existe lo
  respeta"** (comentario literal: "caso DB perdida pero datos quedaron"). Re-crear
  el share con el mismo nombre re-registra la fila sin tocar datos. Alternativa
  quirúrgica: `INSERT` directo en `shares` + `share_permissions`.
- *Solución sistémica:* **FIX-3 a nivel carpeta** — escanear subvolúmenes
  huérfanos bajo `shares/` y ofrecer "re-adoptar" con un botón.

**E3 · El filtro por usuario puede esconder shares.**
- *Causa:* `ListShares(ctx, filterByUser)` (`shares_service.go:459`) filtra a
  shares donde el usuario tiene permiso. Si `share_permissions` se pierde, el
  share existe pero es invisible en el Panel (aunque Files, sin filtro, lo vea).
- *Solución:* al re-adoptar, re-crear el permiso del owner; y para owner/admin,
  no esconder por falta de permiso (mostrar con aviso en vez de ocultar).

### Familia F — Resiliencia de la configuración (la mierda de fondo)

**F1 · La BD viva está en el disco de sistema (SD/boot).**
- *Qué:* `/var/lib/nimos/config/nimos.db` → muerte de la SD = pérdida de la copia
  viva. SPOF.
- *Mitigación existente:* backup a pool (F2).

**F2 · El backup de config a pool es WAL-unsafe. ← causa-raíz probable de E2.**
- *Qué:* `backupConfigToPoolGo()` (`storage_startup.go:392`) copia `nimos.db` con
  `os.ReadFile` cada 30 min a cada pool. Pero la BD usa `journal_mode(WAL)`
  (`db.go:39`): las escrituras recientes viven en `nimos.db-wal` hasta el
  checkpoint. La copia **no hace checkpoint ni copia el `-wal`** → snapshot viejo,
  sin los cambios recientes.
- *Impacto:* las shares creadas fueron al WAL; el backup copió un `.db` sin ellas;
  una restauración posterior trajo la tabla vacía. **Esto explica E2.**
- *Solución:* **§9 Capa 1** — antes de copiar, `PRAGMA wal_checkpoint(TRUNCATE)`,
  o usar la API de backup online de SQLite / `VACUUM INTO` para un snapshot
  consistente de un solo fichero. Es el fix más urgente de todos.

**F3 · No hay restauración automática.**
- *Qué:* si la BD viva está ausente/vacía/corrupta, NimOS no restaura solo desde
  el backup del pool. Es manual.
- *Solución:* **§9 Capa 2** — al arrancar, si la BD viva no es válida y hay un
  backup más nuevo y válido en un pool → restaurar (con log claro / confirmación).

**F4 · El trigger por timer (30 min) deja una ventana de pérdida.**
- *Qué:* cualquier cambio en los últimos ≤30 min puede no estar respaldado.
- *Solución:* **§9** — pasar a **event-driven acotado**:
  - `markConfigDirty()` lo llaman SOLO los cambios de config durable (shares,
    permisos, usuarios, pools/devices, app_registry, app_permissions). **NO** lo
    llaman sesiones, `notification_outbox`, ni estado de salud transitorio.
  - **Debounce + coalescencia:** marcar sucio despierta a un flusher que espera
    2-5 s, junta la ráfaga y hace UN backup (3 shares seguidas = 1 backup).
    Asíncrono (la petición del usuario no espera).
  - **Backstop:** mantener un backup periódico lento (6 h / diario) como red por
    si un hook se escapa o un backup falla en silencio.
  - *Ortogonal al WAL:* da igual el trigger, la copia debe ser WAL-safe (F2).
  - *Futuro opcional:* patrón litestream (replicación de frames del WAL) para
    durabilidad casi-cero-pérdida. Descartado de inicio por dependencia/piezas
    móviles; se prefiere el dirty-flag propio, simple y sin deps.

**F5 · Falta la red definitiva: reconstruir la BD desde la realidad.**
- *Qué:* si se pierden todos los backups, hoy no hay forma de reconstruir la
  config desde lo físico.
- *Solución:* **§9 Capa 3** = la familia **FIX-3** llevada al extremo: re-adoptar
  pools (btrfs), shares (subvolúmenes bajo `shares/`) y apps (contenedores docker)
  desde lo que está físicamente en el pool. La BD pasa a ser puro caché
  reconstruible. Es la máxima expresión de la Regla 16.

### Familia G — Deuda y limpieza menor (salió de paso)

**G1 · Fichero BD huérfano** `/var/lib/nimos/nimos.db` (ruta vieja, sin
`/config/`) confunde diagnósticos. → eliminar / migrar y documentar la ruta real.

**G2 · Código muerto ZFS** en `storage_health.go` (mapeo de estados
`online/faulted/offline/removed/unavail`) tras eliminar ZFS en mayo 2026. →
limpiar para BTRFS-only.

**G3 · Sin vista "qué referencia este pool/share"** (lección de OMV): cuando se
bloquea una op por estar referenciada, mostrar quién la usa (shares/docker/
torrent), en vez de un genérico "en uso".

---

## 4 · Soluciones priorizadas

| # | Solución | Familia | Valor | Esfuerzo | Estado |
|---|----------|---------|-------|----------|--------|
| 1 | **FIX-1** · `assertPoolWritable` en las 4 ops de layout | B | 🔴 alto | muy bajo | ✅ **hecho + desplegado** |
| 2 | **§9 Capa 1** · backup de config **WAL-safe** | F | 🔴 alto | bajo | ✅ **ya existía** (VACUUM INTO) |
| 3 | **FIX-2** · tarjeta consume divergencias del observer | A | 🔴 alto | bajo | ✅ **hecho** (8 h Pi OK) |
| 4 | **Re-adopción de shares** (data11/multimedia/musica) | E | 🔴 alto | trivial | ✅ **hecho** (manual) |
| 5 | **§9 Capa 2** · auto-restore de config al arrancar | F | 🔴 alto | medio | ✅ **hecho** (7 tests) |
| 6 | **§9 event-driven** · `markConfigDirty` + debounce | F | 🟠 medio | bajo-medio | ⬜ **pendiente** |
| 7 | **FIX-3** · re-adoptar huérfanos (pools y shares) desde disco | C/E/F | 🔴 alto | medio | ✅ **hecho** (shares: backend + UI) |
| 8 | **FIX-4** · fstab generado desde la BD (marcadores+nofail) | C | 🔴 alto | bajo-medio | ✅ **hecho + validado en Pi** |
| 9 | **FIX-5** · estado MISSING de primera clase | D | 🟠 medio | bajo | ✅ **hecho** (backend + UI) |
| 10 | G1 · limpiar BD huérfana / documentar ruta | G | 🟡 bajo | trivial | ⬜ pendiente (menor) |
| 11 | G2 · limpiar código muerto ZFS | G | 🟡 bajo | bajo | ⬜ pendiente (menor) |
| 12 | G3 · vista "qué referencia este pool/share" | G | 🟠 medio | medio | ⬜ pendiente (menor) |

> **Cierre (junio 2026):** 9 de las 12 soluciones implementadas. Lo único de código
> sustantivo que queda es la #6 (backup event-driven); el resto (#10-12) es limpieza
> menor no urgente. El plan de diseño y el estado vivo están en
> `PLAN-FIABILIDAD-STORAGE-v2.3.md` (§10).

**Orden recomendado de ataque:** #4 (recuperar tus shares ya) → #1 + #2 (los dos
fixes baratos de máximo valor, uno cierra el bug de la UI, el otro la pérdida de
config) → validar #3 en la Pi → luego #5/#6/#7/#8.

---

## 5 · La lección común (Regla 16, en una frase)

Todos los hallazgos —del split-brain a la pérdida de shares al backup WAL-stale—
son el mismo error: **tratar la BD como verdad cuando es un caché.** La realidad
(kernel/btrfs/disco) es la única autoridad. NimOS debe:

1. **No afirmar** salud sin preguntar a btrfs (FIX-2).
2. **No actuar** sobre un pool sin verificar montaje (FIX-1).
3. **No negar** que un pool/share existe sin mirar el disco (FIX-3).
4. **No perder** la config: respaldarla bien (WAL-safe, §9 C1), restaurarla sola
   (§9 C2) y, en última instancia, **reconstruirla desde el disco** (§9 C3).

El día entero fue la prueba de que la arquitectura es correcta pero la
reconciliación realidad↔caché estaba incompleta. Cerrar esos huecos convierte
NimOS de "NAS que te avisa" en "NAS en el que confías".

---

*Post-mortem redactado en frío tras la recuperación de `data8`. Datos: 0 bytes
perdidos. Hallazgos: 12 problemas catalogados, 9 de ellos ya con fix diseñado o
implementado. Ninguno requiere reconstrucción; todos son conexiones o refuerzos
sobre piezas que ya existen.*
