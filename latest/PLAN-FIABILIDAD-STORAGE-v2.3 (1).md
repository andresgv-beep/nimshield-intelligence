# PLAN · Fiabilidad de Storage v2.3 (Info · Fallos · Detención)

> **v2.3 — SPRINT CERRADO (junio 2026).** Los cinco FIX están **codeados,
> testeados y desplegados**; FIX-4 validado en hardware real (el bloque `[nimos]`
> se genera solo en la Pi de producción). Esta versión marca el estado de
> ejecución (§4 y §10), e integra la **§9 (resiliencia de config)** que quedó
> pendiente desde el incidente. Lo único sin codear: el **backup event-driven**.
>
> **v2.2:** añadió §8 (Lecciones de OpenMediaVault). La investigación de OMV
> —el NAS libre más maduro— **valida** que el modelo de NimOS (BD = lo gestionado
> + lectura en vivo de la realidad) es el estándar de la industria, y aportó dos
> piezas que se nos escapaban: el **estado MISSING de primera clase** y el
> **fstab generado desde la BD** (ascendido a FIX-4 en el camino crítico).


**Estado:** ✅ **EJECUTADO** — FIX-1…5 hechos, testeados y desplegados; FIX-4
validado en hierro real. Ver §10 (estado de ejecución). Pendiente: backup
event-driven (mejora, no rescate).
**Origen:** Plan v1 (Beta 9 alpha) tras el incidente del disco ausente que
aparecía "ok". **Actualizado** tras el incidente de reparación de `data8`
(junio 2026), que validó en hierro real, paso a paso, exactamente el escenario
que el plan v1 anticipó. Esta v2 incorpora las causas-raíz confirmadas leyendo
el código de Beta 9 y afina los fixes a 3 concretos de alto valor.

**Cambio clave en v2.1:** tras seguir el hilo completo del código se confirmó que
NimOS **NO es ciego** — el observer (`storage_observer.go`) YA calcula la verdad
en tiempo real desde btrfs y el frontend YA la pide cada 20s. El bug real es un
**SPLIT-BRAIN** (§1.5): la tarjeta del pool se pinta desde la caché (SQLite +
enrich) y la verdad del observer vive en un panel aparte, sin reconciliar. La UI
te enseña el cerebro equivocado. Esto **abarata** FIX-2 y FIX-3: el cross-check
contra btrfs ya existe, falta **cablearlo a la salud de la tarjeta**, no
construirlo.

**Pregunta que responde:** ¿cómo hacer que NimOS sea fiable de verdad —que no
mienta sobre el estado, detecte fallos siempre, y se DETENGA antes de dañar
datos— y, lo nuevo, que **no sea ciego a un pool sano que existe en disco**?

**Hallazgo confirmado:** la base de fiabilidad EXISTE (la mayoría de piezas del
v1 están codeadas, incluida la puerta `assertPoolWritable`). El trabajo es
**conectarlas y cerrar 3 huecos concretos**, no reconstruir. El incidente de
`data8` señaló con precisión cuáles 3.

---

## 0 · QUÉ PASÓ — Incidente `data8` (validación en hardware real)

Reparación de un pool BTRFS RAID1 en la Pi de producción. Reconstrucción de los
hechos, porque **cada paso es la especificación de lo que NimOS debería hacer
solo**:

1. Un `btrfs replace` (sustituyendo un disco viejo, devid 2) se estaba ejecutando
   cuando la Pi **se reinició a media operación**.
2. Tras el reinicio, el pool quedó con **3 devids**: `sda` (devid 1, presente),
   el SSD `sdb` (devid 0 temporal del replace, presente) y el viejo **devid 2
   MISSING** (ya no existe físicamente; el replace nunca lo soltó).
3. El pool **no estaba en `/etc/fstab`** → no se automontó al arrancar. El
   reconcile de montaje no lo rescató.
4. La **UI de NimOS mostró `data8` como "single · 1 disco · HEALTHY"** mientras
   btrfs tenía 3 devids y uno MISSING. Contradicción directa BD↔realidad.
5. El botón **"Mejorar a RAID1"** lanzó `btrfs device add` sobre un pool **no
   montado** → error críptico `ERROR: not a btrfs filesystem: /nimos/pools/data8`.
   El usuario no tenía forma de saber que el problema era simplemente "no montado".
6. La reparación **tuvo que hacerse 100% a mano por terminal**: `btrfs device
   scan` (la Pi no había registrado `sdb`), `mount -o degraded`, `btrfs device
   remove missing` (15.9%→100%, 0 errores), verificación de RAID1, y `fstab`.
7. Resultado: pool sano (2 discos, sin MISSING, 24.85 GiB intactos) — pero
   **NimOS seguía sin verlo bien**, porque su BD conservaba el estado fantasma
   de la pelea.

**Daño a datos: cero.** En ningún momento se perdió un byte. **Fallo de
producto: alto.** Un NAS que no sabe reparar su propio pool, y que miente sobre
su salud, obliga al usuario a la terminal en su peor momento. Eso es justo lo
que esta v2 cierra.

---

## 1 · QUÉ FALLA EN NIMOS — Causas-raíz confirmadas en el código (Beta 9)

No son hipótesis: son lecturas directas del código. Las cuatro son la **misma
enfermedad** —confiar en el estado guardado (SQLite/caché) en vez de en la
realidad del kernel/btrfs—, que es exactamente la violación de la Regla 16.

### F1 · NimOS lee la EXISTENCIA de pools de la BD, no de los discos
`StorageService.ListPools` (`storage_service.go`) hace `s.repo.ListPools(ctx)`
→ **SQLite**. El kernel solo se consulta *después*, en `enrichPool`, para pools
que YA están en la BD. **Consecuencia:** si la fila de la BD se pierde o se
corrompe, btrfs puede tener el pool montado y sano y NimOS es **ciego** a él.
Para NimOS "existir" = "tener fila", no "estar en los discos". *(Hallazgo nuevo
de este incidente; no estaba explícito en el v1.)*

### F2 · Las ops de layout no verifican montaje antes de tocar btrfs
`AddDevice`, `ConvertProfile`, `ReplaceDevice`, `RemoveDevice`
(`storage_service_device.go`, `storage_service_profile.go`) ejecutan su
`btrfs ... <mountpoint>` **sin comprobar que el pool esté montado**, y **sin
llamar a `assertPoolWritable`** (que ya existe). **Consecuencia:** cuando hay
drift BD↔kernel, el usuario recibe `not a btrfs filesystem` en vez de un honesto
*"el pool no está montado, móntalo primero"*. La info para dar el mensaje bueno
ya está (`isPoolMounted`), solo falta cablearla.

### F3 · La salud DEGRADED de la TARJETA se calcula contra la BD, no contra btrfs
`storage_health.go` (~L405-466) cuenta `disk_missing` comparando los discos que
la BD cree que tiene el pool contra los reales. **Consecuencia:** si la BD
conserva un disco fantasma (el caso de `data8`: BD con 3 discos, btrfs con 2),
la tarjeta reporta **DEGRADED falso** — o, al revés, si la BD cree que es un
single de 1 disco, reporta **HEALTHY falso** ignorando que btrfs ve más devids.
**Matiz v2.1:** el observer YA cuenta los missing bien (desde btrfs); el problema
es que la **tarjeta no usa la cuenta del observer**, usa la suya sobre la BD.

### F4 · La tarjeta puede afirmar "HEALTHY/single" sobre un pool con MISSING
La contradicción de la UI ("single · HEALTHY" con devid MISSING real) es el
síntoma del split-brain (§1.5): la tarjeta se deriva de la caché y **no consume
las divergencias que el observer ya calculó**. Ninguna vista debería poder
afirmar "sano" si el observer dice `DivPoolMissingDevice` para ese UUID.

### Secundario · Sin persistencia en fstab + flag `degraded` que se queda pegado
El pool no estaba en `fstab` (no automonta tras reinicio). Y un montaje hecho con
`-o degraded` **arrastra el flag indefinidamente** hasta el siguiente remount,
aunque el pool ya esté sano — montaje "mentiroso" que conviene limpiar.

---

## 1.5 · EL HALLAZGO CENTRAL — SPLIT-BRAIN (por qué la UI miente en tiempo real)

**NimOS NO es ciego. Calcula la verdad en tiempo real. Pero tiene dos cerebros y
la UI pinta el equivocado.** Esta es la causa-raíz que unifica F1-F4.

**Cerebro A — la realidad (correcto, ya existe, ya corre):**
- `storage_observer.go` lee `btrfs filesystem show` (la autoridad) y produce
  **divergencias** vía `analyzeDivergences`:
  - `DivOrphanFilesystem` → pool en disco sin fila en BD (F1).
  - `DivPoolMissingDevice` → pool managed con discos missing, contados desde
    `fs.DevicesMissing` **leído de btrfs** (F3).
  - `DivPoolUnmounted` → pool registrado pero no montado.
  - Pool en BD que btrfs no detecta → crítico.
- El frontend (`StorageApp.svelte`) YA pide `/observed` en cada poll (**20s
  normal, 3s en reparación**) y YA calcula `orphanFilesystems` y `divergences`.

**Cerebro B — la caché (lo que se pinta como titular):**
- La **tarjeta del pool** se renderiza desde `/pools` = SQLite + `enrichPool`.
- Su badge de salud sale de `storage_health.go`, que cuenta missing **sobre el
  set de discos de la BD**, no sobre el del observer.

**El bug:** los dos cerebros **no están reconciliados**. La tarjeta NO consume las
divergencias del observer. Así que el badge "HEALTHY" (caché) es el titular y la
verdad ("falta 1 disco", observer) queda en un panel secundario de "Observados"
que el usuario puede ni ver. **La mentira y la verdad coexisten en la misma
pantalla; gana la mentira porque es el titular.**

Es un split-brain literal: **dos rutas de código distintas** calculan "¿falta un
disco?" —una desde la BD (tarjeta), otra desde btrfs (observer)— y pueden
contradecirse. Y NO es un problema de tiempo real / polling: el poll de 20s es
casi instantáneo y `enrichPool` SÍ lee presencia de disco en vivo
(`resolveDeviceState`). El fallo es que **la decisión de "¿está sano?" se toma
sobre la caché, no sobre lo que el observer acaba de leer de btrfs.**

> NimOS lee la realidad en tiempo real, pero juzga la salud sobre la caché.
> Esa es toda la mentira, en una frase.

---

## 2 · QUÉ DEBATÍAMOS ARREGLAR — Los 3 fixes que cierran el incidente

El plan v1 estima 8-10 sesiones para fiabilidad completa. El incidente reveló
que **el 90% del dolor de `data8` lo cierran estos 3**, y uno es casi gratis
porque la función ya existe.

### FIX-1 · Cablear `assertPoolWritable` en las 4 ops de layout  *(cierra F2)*
**Mapea a:** v1 · CAPA 3 · Detención · punto 2 (guard unificado) — que **ya está
escrito** en `storage_writable_guard.go` y ya lo usa `files.go`. Solo falta que
lo crucen las ops de disco.
**Trabajo:** al inicio de `AddDevice`/`ConvertProfile`/`ReplaceDevice`/
`RemoveDevice`, llamar a `assertPoolWritable(pool.MountPoint)` (o un
`ensurePoolMounted` que verifique con `isPoolMounted` y, si procede, intente
montar y reverifique). Si no está montado/seguro → error claro y accionable, no
`not a btrfs filesystem`.
**Coste:** ~3 líneas por función. La función ya existe. **El más rentable.**

### FIX-2 · La salud de la tarjeta CONSUME las divergencias del observer  *(cierra F3+F4, el split-brain)*
**Mapea a:** v1 · CAPA 2 · Fallos · punto 3, **pero el cross-check YA EXISTE** en
`analyzeDivergences`. No hay que construirlo: hay que **cablearlo a la tarjeta.**
**Spec concreto:**
- Regla dura: si el observer emite `DivPoolMissingDevice`, `DivPoolUnmounted` o
  el crítico "registrado pero no detectado" para el UUID de un pool, ese pool
  **NO puede** mostrar `health = healthy`. La realidad (observer) **sobrescribe**
  la salud derivada de la caché.
- Implementación mínima: en el handler que sirve `/pools` (o en `enrichPool`),
  cruzar cada pool con las divergencias del observer por `BtrfsUUID` y **degradar
  el estado** si hay divergencia. Backend, una función, una autoridad.
- UI mínima: banner bloqueante en la tarjeta — "⚠ La realidad no concuerda con
  nuestros registros: <detalle de la divergencia>" — en cuanto exista divergencia
  para ese pool. Quita el badge HEALTHY mentiroso de un plumazo.
**Coste:** **bajo** (era "medio" en v2). La maquinaria de verdad ya está; es
reconciliar dos fuentes que ya se cargan en el frontend.

### FIX-3 · La existencia del pool: no esconder los huérfanos + re-adoptar filas rotas  *(cierra F1)*
**Mapea a:** Regla 16 aplicada a la *existencia*. El observer YA marca
`DivOrphanFilesystem` (pool en disco sin BD) y el import YA sabe re-adoptarlo.
**Spec concreto (lo que falta):**
- **No esconder:** un `DivOrphanFilesystem` debe ser visible como tarjeta
  "importable", no relegado a un panel que se ignora. Que un pool sano en disco
  nunca sea invisible.
- **Trampa a corregir:** el import rechaza con `ALREADY_MANAGED` si la fila existe
  aunque esté corrupta (caso `data8`: fila con devices fantasma). Falta una vía de
  **reconciliar una fila existente** — resincronizar su set de devices y profile
  desde `btrfs filesystem show` — sin obligar a destruir y recrear.
**Coste:** medio. Observer + import existen; falta exponer la re-adopción y el
reconcile de fila corrupta.

### FIX-4 · fstab generado desde la BD, con marcas de propiedad, atómico al montar  *(cierra la causa secundaria — promovido en v2.2)*
**Mapea a:** lección directa de OMV (§8). La causa secundaria de `data8` (pool no
estaba en fstab → no automontó tras reinicio) deja de ser "secundaria": es la
raíz de que el reboot a media operación dejara todo sin montar.
**Spec concreto:**
- NimOS **genera** sus líneas de fstab desde la BD (no las acumula ad-hoc), entre
  marcadores de propiedad tipo `# >>> [nimos]` … `# <<< [nimos]`, para poder
  reescribirlas de forma segura sin tocar las del usuario.
- **Montar/crear/importar escribe BD Y fstab en el mismo acto** (atómico): así la
  BD y el fstab nunca divergen. El montaje es el único embudo.
- Opción `nofail` en las líneas de pools de datos (como OMV) para que un disco
  ausente no bloquee el arranque del sistema.
**Coste:** bajo-medio. Ya existe `appendFstab`; el trabajo es pasar a *generar*
(no *anexar*) desde la BD, con marcadores, en el flujo de create/import.

### FIX-5 · Estado MISSING de primera clase  *(lección de OMV — UX de F1/F3)*
**Mapea a:** §8. OMV nunca muestra un filesystem ausente como "healthy" ni lo
esconde: lo marca **MISSING** (gris, distinto de "degraded").
**Spec concreto:** añadir un estado de tarjeta `missing` distinto de `degraded`,
alimentado por el crítico del observer "registrado pero btrfs no lo detecta" y por
`DivPoolMissingDevice`. Texto accionable: "el disco <serial> no está presente".
Un pool cuyo filesystem entero no está ≠ un RAID degradado: son dos cosas y la UI
debe distinguirlas.
**Coste:** bajo. Es un valor de estado nuevo + su render; la detección ya existe
en el observer.


> **Nota de estado del v1:** la pieza estrella del plan v1 (FIX-1 del orden de
> prioridad, `assertPoolWritable` unificado) **ya está codeada**. La tabla de
> "ESTADO ACTUAL" del v1 sigue vigente; esta v2 solo confirma que la base es aún
> más sólida de lo que parecía y reenfoca el esfuerzo en conectar.

---

## 3 · TESTS DE DESTRUCCIÓN — Reproducir el incidente en CI

`tests/destruction/` ya existe (T02–T10). Faltan los tres que reproducen
`data8`. Cada uno **falla hoy** y debe pasar tras su fix. Convierten "me pasó
algo raro durante dos días" en 30 segundos de CI determinista.

| Test | Reproduce | Assert | Cierra |
|------|-----------|--------|--------|
| `T11_layout_op_unmounted.sh` | op de layout (add/convert) sobre pool **no montado** | error claro "no montado", **no** `not a btrfs filesystem`; no se toca el disco de sistema | FIX-1 / F2 |
| `T12_card_ignores_divergence.sh` | observer emite `DivPoolMissingDevice` para un pool, pero la BD lo cree sano | la tarjeta **NO** muestra HEALTHY; el banner de divergencia aparece (la realidad sobrescribe la caché) | FIX-2 / F3+F4 / split-brain |
| `T13_orphan_pool_readopt.sh` | pool sano en disco con **fila de BD borrada** | el pool aparece como observado/importable y la re-adopción lo deja **HEALTHY** con sus discos reales | FIX-3 / F1 |

**Workflow (el de siempre):** crear pool → meter archivos → provocar el estado de
fallo (desmontar / inyectar fila fantasma / borrar fila) → arrancar daemon →
assert. Escribir el test **primero**, verlo fallar, luego el fix.

---

## 4 · PRIORIZACIÓN (actualizada con el incidente)

| Orden | Qué | Capa | Valor | Esfuerzo | Estado |
|-------|-----|------|-------|----------|--------|
| 1 | **FIX-1** · `assertPoolWritable` en las 4 ops de layout | Detención | 🔴 alto | **muy bajo** | ✅ **hecho + desplegado** |
| 2 | **FIX-2** · la salud de la tarjeta consume divergencias del observer | Fallos/Info | 🔴 alto | **bajo** | ✅ **hecho + desplegado** (8h Pi OK) |
| 3 | **FIX-3** · no esconder huérfanos + re-adoptar fila corrupta | Info | 🔴 alto | medio | ✅ **hecho** (backend + UI, permisos preservados) |
| 3.5 | **FIX-4** · fstab generado desde la BD (marcadores + nofail, atómico) | Detención | 🔴 alto | bajo-medio | ✅ **hecho + VALIDADO en Pi** |
| 3.6 | **FIX-5** · estado MISSING de primera clase (gris, ≠ degraded) | Info | 🟠 medio | bajo | ✅ **hecho** (backend + UI) |
| — | **Auto-restore de config** (Capa 2, §9) | Info | 🔴 alto | bajo | ✅ **hecho** |
| — | **Backup WAL-safe** (VACUUM INTO, §9) | Fallos | 🔴 alto | — | ✅ **ya existía** en el código |
| — | **Backup event-driven** (`markConfigDirty` + debounce, §9) | Fallos | 🟠 medio | bajo | ⬜ **pendiente** (única tarea de código viva) |
| 4 | `resolveDeviceState` única (v1 CAPA 1) | Info | 🔴 alto | medio | parcial (deviceIsPresent) |
| 5 | Serial como ancla en todos los puntos (v1 CAPA 2.1) | Fallos | 🟠 medio | bajo | reconciler ya lo hace |
| 6 | Evento `device_swapped` + notif (v1 CAPA 2.2) | Fallos | 🟠 medio | bajo | — |
| 7 | No operar sobre disco de identidad dudosa (v1 CAPA 3.3) | Detención | 🔴 alto | medio | — |
| 8 | Read-only forzado al borde del colapso (v1 CAPA 3.1) | Detención | 🟡 alto | alto | detecta, no actúa |
| 9 | Freno de operación si estado incoherente (v1 CAPA 3.4) | Detención | 🟠 medio | medio | — |

**Camino crítico CERRADO.** FIX-1/2/3/4/5 + auto-restore + WAL-safe convierten el
incidente de `data8` en un no-evento por todos sus eslabones. Los puntos 4-9 son
mejoras futuras, no rescate.

---

## 5 · PRINCIPIO RECTOR (Regla 16, extendida a la existencia)

> La realidad del **kernel/btrfs/SMART** es la ÚNICA autoridad. La BD es una
> caché. Cuando difieren: **la realidad gana**, y ante la duda, NimOS **NO actúa**
> (no escribe, no borra, no afirma "sano"). Y, lo nuevo: NimOS **NO es ciego** a
> lo que la realidad contiene aunque la caché no lo registre — un pool que existe
> en btrfs existe, lo sepa la BD o no.

Antes de **actuar** sobre un pool: ¿está montado de verdad? (FIX-1).
Antes de **afirmar** la salud de un pool: ¿qué dice btrfs de sus discos? (FIX-2).
Antes de **negar** que un pool existe: ¿qué ve btrfs en los discos? (FIX-3).

Tres preguntas, una sola autoridad. Eso convierte NimOS de "NAS que te avisa" en
"NAS en el que confías".

---

## 6 · LÍMITES (qué NO entra)

- No rehace piezas que ya funcionan (reconciler por serial, SMART monitor, health
  base, observer, import, `assertPoolWritable`).
- El reconciliador de montaje al arranque y la persistencia automática en `fstab`
  van en su propio frente (complementario; el incidente confirma que es
  necesario — `data8` no estaba en fstab). Aquí se asume y se conecta.
- El flujo completo de reemplazo de disco con LEDs/TXT es plan aparte; esta v2 le
  da la base fiable (identidad por serial, membresía por btrfs) sobre la que se
  apoya.

---

## 7 · ESTIMACIÓN

- **FIX-1 + FIX-2 + FIX-3 (el grueso del valor, cierran `data8`):** ~2 sesiones
  (v2.1 bajó FIX-2 de medio a bajo: el cross-check del observer ya existe, solo
  se cablea), con sus tests T11/T12/T13 escritos primero.
- Puntos 4-7: ~3-4 sesiones.
- Puntos 8-9 (read-only forzado + freno): ~3 sesiones, requieren un disco que
  falle de verdad para validar en hardware.
- **Total fiabilidad completa: ~8-11 sesiones.** Pero el incidente demostró que
  el 90% del dolor real lo quitan FIX-1/2/3. Se puede parar ahí y NimOS ya no
  repite `data8`.

---

*v2.1 trazada tras validar el plan v1 en el incidente real de `data8` (junio
2026, Beta 9 alpha) y seguir el hilo completo del código. La causa-raíz que
unifica todo es el **SPLIT-BRAIN** (§1.5): NimOS YA lee la realidad en tiempo real
(observer + divergencias, polling de 20s), pero la tarjeta del pool juzga la salud
sobre la caché de SQLite en vez de sobre lo que el observer acaba de leer de btrfs.
No hay que construir la detección de verdad — ya existe — hay que **reconciliar
los dos cerebros**: la realidad (observer) sobrescribe la caché (enrich). Eso, más
el gate de montaje en las ops de layout (FIX-1) y la re-adopción de filas
corruptas (FIX-3), cierra el incidente entero. El camino crítico es de ~2
sesiones, tests primero.*

---

## 8 · LECCIONES DE OPENMEDIAVAULT (adoptar / evitar)

Investigación del código y la doc de OMV (el NAS libre más maduro, ~15 años). El
hallazgo de fondo: **el modelo de NimOS es el de OMV** — `config.xml` (su BD) es
la fuente de verdad de lo gestionado, y un filesystem montado fuera del web UI
no existe para OMV. NimOS no hace nada raro; es el estándar de la industria.
Sobre esa validación, OMV enseña qué adoptar y qué evitar.

### ADOPTAR (OMV lo hace mejor; se nos escapaba)

1. **Estado MISSING de primera clase.** Cuando una entrada de `config.xml`
   referencia un disco ausente (falló / se quitó / se durmió por spindown), OMV
   muestra el filesystem como **"missing"** en gris — nunca "healthy", nunca
   oculto. Lección: `missing` es un estado propio, distinto de `degraded`. → FIX-5.

2. **fstab generado desde la BD, con marcadores, atómico al montar.** OMV escribe
   sus líneas entre `# >>> [openmediavault]` … `# <<< [openmediavault]`, generadas
   desde `config.xml`. Montar por la UI escribe BD + fstab en el mismo acto, así
   nunca divergen. Cierra de raíz el "pool no estaba en fstab" de `data8`. → FIX-4.

3. **`nofail` en pools de datos.** OMV monta los volúmenes de datos con `nofail`
   para que un disco ausente no bloquee el arranque del sistema. → parte de FIX-4.

### EVITAR (errores de OMV; NimOS ya iguala o supera)

4. **`getList` todo-o-nada.** En OMV, una sola entrada mala en `config.xml` (un
   backend inexistente, un fs corrupto) lanza "Failed to get filesystems" y deja
   la lista ENTERA vacía. El `enrichPool` por-pool de NimOS es más robusto: un
   pool roto no debe vaciar la lista. **No copiar el monolito.**

5. **"Missing" como callejón sin salida.** En OMV un filesystem missing no se
   puede borrar (queda "en uso") y se acaba editando `config.xml` a mano — misma
   trampa que el `ALREADY_MANAGED` de NimOS. **FIX-3 (re-adoptar/reconciliar fila
   corrupta desde la UI) le gana**: es una queja de OMV abierta hace una década.

6. **Sin vista de "qué usa este disco".** Los usuarios de OMV llevan años pidiendo
   ver qué referencia un volumen para entender por qué está bloqueado. Lección:
   cuando NimOS bloquee una op por estar el pool referenciado (shares/docker/
   torrent), **mostrar qué lo referencia**. Barato y diferenciador.

### Síntesis

Lo nuevo que aporta OMV: (a) **MISSING de primera clase** (FIX-5) y (b) **fstab
desde la BD** ascendido a pieza central (FIX-4). El resto —robustez por-pool,
re-adopción desde UI, vista de referencias— son frentes donde el diseño de NimOS
ya iguala o supera a OMV. La conclusión tranquilizadora: la arquitectura es
correcta; lo que falla es la reconciliación realidad↔caché, y eso es justo lo que
FIX-1…5 cierran.

---

## 9 · RESILIENCIA DE CONFIG (la otra mitad del incidente)

El incidente de `data8` tuvo **dos** daños: el drift de storage (FIX-1…5) y,
aparte, la **pérdida de las filas de `shares`** tras restaurar un backup de config
viejo. Esta sección cierra la segunda mitad: que la config (SQLite) sobreviva y se
recupere sola.

### 9.1 · Backup WAL-safe — ✅ ya existía

`storage_config_backup.go` ya respalda la BD con **`VACUUM INTO`**, que captura un
snapshot consistente **incluyendo lo que vive en el WAL** (escritura a temporal +
rename atómico, `chmod 0600`). Copiar el `.db` a pelo —la causa probable de la
pérdida de shares— ya no ocurre. Se respalda a cada pool montado cada 30 min.

### 9.2 · Auto-restore al arrancar (Capa 2) — ✅ hecho

`storage_config_restore.go`: si la BD viva **no existe o está vacía** y hay un
backup **válido** (con tablas críticas) en un pool montado, lo restaura **antes de
abrir la BD**. Regla de oro: **solo** sobre BD ausente/vacía — jamás pisa una BD
con contenido. Convierte "muerte de SD = drama manual" en "reinstalas y NimOS se
reconstruye solo". *Límite conocido:* no cubre la reinstalación de SD con los
pools **sin montar** todavía (eso requiere montarlos antes — familia FIX-3).

### 9.3 · Re-adopción de shares huérfanas (FIX-3, shares) — ✅ hecho

Backend + UI. `findOrphanShares` escanea los subvolúmenes bajo `<pool>/shares/`
ausentes de la BD; `readoptOrphanShare` los re-registra **preservando dueño/grupo**
(reutiliza `CreateShare` saltándose el re-chown). Banner ámbar en `CPShares.svelte`
con "Re-adoptar". Lo que fue una noche de terminal, ahora es un botón.

### 9.4 · Backup event-driven — ⬜ PENDIENTE (única tarea de código viva)

Hoy el backup es por **timer de 30 min** → ventana de pérdida de hasta 30 min.
Propuesta (idea de Andrés): `markConfigDirty()` solo en cambios de config durable
(pools, shares, users, permisos — **no** sesiones/notifs/health) + flusher con
**debounce de 2-5s** + coalescencia, más un **backstop diario**. Ortogonal al
WAL-safe (este es *cuándo* copiar; aquel es *cómo*). Baja la ventana de minutos a
segundos. Bajo riesgo, autocontenido.

---

## 10 · ESTADO DE EJECUCIÓN (cierre del sprint, junio 2026)

| Pieza | Estado | Validación |
|-------|--------|------------|
| FIX-1 · gate de montaje en ops de layout | ✅ hecho + desplegado | suite verde; 5 tests T11 |
| FIX-2 · split-brain (tarjeta consume divergencias) | ✅ hecho + desplegado | 8 h en Pi intacto |
| FIX-3 · re-adopción de shares (backend + UI) | ✅ hecho | 6 tests; CSS+Svelte validados |
| FIX-4 · fstab desde la BD | ✅ hecho + **validado en Pi** | bloque `[nimos]` generado OK |
| FIX-5 · estado MISSING de primera clase | ✅ hecho (backend + UI) | 3 tests; CSS+Svelte validados |
| Backup WAL-safe (VACUUM INTO) | ✅ ya existía | 3 tests |
| Auto-restore de config (Capa 2) | ✅ hecho | 7 tests |
| Shares de `data8` recuperadas | ✅ hecho (a mano) | 3 carpetas, 0 bytes perdidos |
| **Backup event-driven** | ⬜ **pendiente** | — |

**Limpieza menor pendiente** (no urgente): BD huérfana en `/var/lib/nimos/nimos.db`
(ruta vieja sin `/config/`), código muerto de ZFS en `storage_health.go`, vista de
"qué referencia este pool/share".

**Conclusión:** la saga de `data8` está cerrada por todos sus eslabones —ni mentira
de salud, ni op sobre pool no montado, ni shares perdidas sin re-adopción, ni pool
fuera de fstab, ni ausente disfrazado de degradado. Lo que queda es mejora, no
rescate.

---

*Apéndice — secciones de diseño originales (§2-§3, specs de cada FIX y tests
T11/T12/T13) conservadas abajo como referencia de cómo se construyó cada pieza.*
