# PLAN — Pasada de Optimización (Perf Pass 1)

**Place:** Brainrot Room Rush [BETA] — hub `73079695240690`
**Fecha:** 2026-07-16
**Base:** Profiling de Dan (MicroProfiler) + auditoría de código vía MCP Studio.

---

## 1. Diagnóstico (qué encontramos y por qué pasa)

### 1.1 `Script_Blaster` — 5.44ms en picos, 9–17 calls/frame (cliente)

El label del profiler corresponde al LocalScript `Blaster` dentro de cada Tool de arma
(`ServerStorage.WeaponPrefabs.*.Scripts.Blaster`), que instancia `BlasterController`
(`ReplicatedStorage.Blaster.Scripts.BlasterController`). Todo el trabajo de sus
conexiones se atribuye a ese script. Costos reales, por orden de impacto:

| # | Fuente | Frecuencia | Costo |
|---|--------|-----------|-------|
| 1 | `autoShootTick` → `AimSnap.resolve` (`BlasterController:317`) | **Cada Heartbeat** mientras el arma está equipada y no hay cooldown pendiente | `CollectionService:GetTagged` (aloca tabla nueva), por candidato: `FindFirstChild("HumanoidRootPart")` + `FindFirstChildOfClass("Humanoid")` + `GetAttribute`, `isPlayerHumanoid` (FindFirstAncestorOfClass), y **un raycast LOS por candidato** dentro del cono (`AimSnap.losClear`, con `RaycastParams.new()` por llamada). Escala linealmente con # de enemigos. |
| 2 | AimAssist (`AimAssistController:21-31`) | 2 binds a RenderStep siempre; en **touch/gamepad** `TargetSelector:selectTarget` cada frame | `GetTagged` + `GetBoundingBox` por modelo + raycast LOS por candidato (`_checkPointHit`), y una **segunda pasada de pesos que repite los raycasts**. Móvil = peor caso. |
| 3 | `shoot()` | Por disparo (ráfagas = picos) | `castRays` (N spherecasts), `drawRayResults` (clones de partículas), `FireServer` serializando tabla con Instances. Explica el pico de 7.5ms @frame 101. |
| 4 | `InputChanged` global (`BlasterController:377`) | Varios eventos por frame (deltas de mouse) | Handler barato (check gamepad). Infla el call count (9–17) pero NO es el costo dominante. |

**Bug encontrado (perf + correctness), `AimAssistController:62`:**
```lua
return self.lastActiveTouch :: number - os.clock() < Constants.AIM_ASSIST_INACTIVITY
```
La resta está invertida (pasado − ahora = negativo → siempre `< INACTIVITY`).
En touch, tras el primer TouchPan la elegibilidad queda `true` **para siempre** →
`selectTarget` (raycasts + GetBoundingBox) corre cada frame en móvil sin parar.
Debe ser `os.clock() - self.lastActiveTouch`.

### 1.2 `Script_GameSettings` se ejecuta varias veces por frame

Ese script **no existe en el hub place** (buscado por nombre en todo el árbol).
Está en otro place (¿playground `129584054856482`?). Nota técnica: N cajas del mismo
label en un frame = N callbacks/threads resumidos de ese script (varias conexiones o
varios loops con `task.wait`). No es necesariamente un bug, pero si son muchos loops
conviene consolidarlos en un solo Heartbeat. **Acción: localizarlo en el place correcto
antes de tocar nada.** Costo por caja ~0.014ms — prioridad baja.

### 1.3 Rendering — CullJob / VisibleQuery 5.9ms

El costo de occlusion culling escala con la cantidad de instancias cargadas/visibles.

- **StreamingTargetRadius 1024 → 512**: menos instancias en memoria = menos trabajo de
  culling y streaming. Config-only, reversible. (Decisión ya tomada por Dan.)
- **Corrección al hallazgo de Dan sobre Anchored**: es al revés — la geometría estática
  **anclada** es la barata; los parts NO anclados son dinámicos y obligan a actualizar
  las estructuras espaciales/culling cada frame. **No desanclar nada**; al contrario,
  verificar que toda la decoración estática tenga `Anchored = true`.
- `StreamingIntegrityMode = PauseOutsideLoadedArea` ya está bien.
- Si con 512 hay pop-in cerca del jugador, subir `StreamingMinRadius` 64 → 128.

### 1.4 Hallazgo nuevo — inventario de loops per-frame en cliente

~20+ conexiones Heartbeat/RenderStepped simultáneas (grep completo):

- **AnimControllers por tipo de enemigo** (14+ scripts: MeleeBasic, Brute, Duelist,
  Gunner, BonecaAmbalabu, …): cada instancia de enemigo crea su propia conexión
  Heartbeat para chequear `AssemblyLinearVelocity`. Con 20 enemigos = 20+ callbacks/frame.
  `TitleTestClient:15` ya tiene el patrón correcto (un `sharedConn` con registro de drivers).
- **TG_VHS_effect**: 4 LocalScripts (`StaticRGB`, `ScanLine`, `StaticLines`, `StaticNoise`)
  cada uno con su RenderStepped mutando UI cada frame → invalidaciones de UI continuas.
- Otros: EnemyOutline, DamageToast ×2, BoonHUD, WaypointTracker, SpeedLinesVFX,
  DistanceTracker, ViewModelJanitor, TimerController, OffScreenManager (este ya está
  bien cacheado), InputModeState, CardVisibility, DownSystem.
- **Server**: EnemyAI, BoonManager, DashService, WorldCardService, EffectRuntime
  (fuera de scope de esta pasada; revisar en pasada 2).

### 1.5 Replicación de disparos (server, `ServerScriptService.Blaster.Scripts.Blaster:227`)

Por cada disparo, el server hace `FireClient` a **todos** los demás jugadores con la
tabla `rayResults` completa (incluye referencias a Instances y campos que el receptor
no usa). Con SMGs (~10 disparos/s) × N jugadores = ancho de banda y costo de
deserialización + VFX en cada cliente.

---

## 2. Plan por fases (ordenado por riesgo ascendente)

### Fase 0 — Config + instrumentación (riesgo ~cero, hoy)

| Acción | Detalle | Validación |
|--------|---------|------------|
| 0.1 StreamingTargetRadius = 512 | ⚠️ **PENDIENTE — paso manual**: la propiedad es NotScriptable (no se puede setear por código/plugin). Cambiar a mano: Workspace → Streaming → StreamingTargetRadius = 512 | Recorrer mapa: sin pop-in molesto hasta cuarto de basket |
| 0.2 Auditoría Anchored | ✅ HECHO 2026-07-16: **21564/21689 parts anclados (99.4%)**. Los 125 sueltos son props físicos legítimos (DeskChair 45, Enemies 29, dressing rooms, meshes de armas display). Anchored NO es problema en este place | — |
| 0.3 Instrumentar con `debug.profilebegin/profileend` | ✅ HECHO: labels `AutoShoot_AimSnap`, `Shoot_SnapAndRays`, `Shoot_VFX` (BlasterController) y `AimAssist_SelectTarget` (AimAssistController). AnimControllers quedan para Fase 2 (se van a consolidar de todos modos) | Labels visibles en MicroProfiler para medir antes/después |

### Fase 1 — Cliente, bajo riesgo (sin cambio de gameplay perceptible)

| Acción | Detalle | Gain esperado | Riesgo |
|--------|---------|---------------|--------|
| 1.1 Fix bug touch eligibility | ✅ HECHO: `os.clock() - lastActiveTouch` (antes invertido). ⚠️ QA MÓVIL: el magnetismo de cámara touch ahora expira 0.1s después de dejar de panear (mismo patrón que thumbstick de gamepad, comportamiento diseñado del kit). Antes quedaba activo permanente por el bug | Elimina selectTarget per-frame permanente en móvil | Validar feel en playtest móvil |
| 1.2 Throttle `autoShootTick` | ✅ HECHO con diseño DPS-safe: solo se throttlea el re-escaneo cuando el scan anterior NO encontró target (`AUTO_SHOOT_SCAN_INTERVAL = 0.1`). Con target presente el scan corre apenas expira el cooldown del arma → **cadencia de disparo idéntica**. Único efecto: hasta 0.1s de latencia al detectar un enemigo NUEVO en el cono | Scan idle de 60/s a 10/s (~83% menos) | Imperceptible |
| 1.3 Cache en `AimSnap` | ✅ HECHO (scope reducido): `RaycastParams` reutilizado en `losClear`. El cache de candidatos por tag-signals se DESCARTÓ: con 1.2 el scan corre a 10/s y `GetTagged` deja de ser relevante — menos código, menos riesgo | Menos allocs por scan | Nulo |
| 1.4 LOS solo al mejor candidato | ✅ HECHO: `pickBest` en 2 fases (recolectar cono sin raycasts → LOS en orden de mejor dot, primer visible gana). Semántica idéntica: mismo resultado que antes (mejor dot CON LOS limpio). Verificado en runtime con enemigos sintéticos: mejor dot ✔, fallback con muro ✔, muerto ✔, 0.063ms/resolve | De N raycasts a típicamente 1 por scan | Nulo (equivalente) |
| 1.5 Consolidar TG_VHS_effect | ✅ HECHO: los 4 `anim` (StaticRGB/ScanLine/StaticLines/StaticNoise) quedaron como no-ops deprecados; `EnableInGame` ahora corre UN RenderStepped a 24Hz con los rangos idénticos y se salta el trabajo si el Gui está apagado | 4 loops 60Hz → 1 loop 24Hz | Verificar visual en playtest (ruido VHS a 24Hz ≈ igual) |

**Verificación runtime (2026-07-16, Studio play solo):** cero errores nuevos en consola
(server Blaster requirió AimSnap OK, todos los controllers cliente inicializaron).
AimSnap ejercitado server-side con enemigos sintéticos: los 3 casos pasan. El VHS no se
pudo ver animar porque RenderStepped no dispara con la ventana sin foco (Heartbeat=31,
RenderStepped=0 en 0.5s) — aplica igual al código viejo; validar visual en próximo playtest.

### Fase 2 — Cliente/red, riesgo medio (requiere playtest QA)

| Acción | Detalle | Gain | Riesgo |
|--------|---------|------|--------|
| 2.1 Cache targets en `TargetSelector` | Refrescar `getAllTargetPoints` a ~10 Hz (no por frame); usar `HumanoidRootPart.Position` en vez de `GetBoundingBox` cuando exista; no repetir raycasts en la pasada de pesos con 1 solo punto | Grande en móvil/gamepad | Feel del aim assist: validar con QA |
| 2.2 Payload compacto en `ReplicateShot` | Enviar solo `{origin: Vector3, hits: {Vector3}}` (posiciones), no `rayResults` con Instances; el receptor solo dibuja VFX | Menos red + deserialización | VFX de otros jugadores: verificar paridad visual |
| 2.3 Filtro de distancia en replicación | No replicar disparos a jugadores a >150-200 studs (con StreamingRadius 512 ni siquiera ven al shooter) | Menos FireClient por disparo | Bajo |
| 2.4 Consolidar AnimControllers | Un manager con un solo Heartbeat y registro de enemigos (patrón `TitleTestClient`); cada tipo aporta su función update | De N conexiones a 1 | Medio (refactor de 14 scripts, hacerlo incremental) |

### Fase 3 — Siguiente pasada (no bloquear esta)

- Pooling de VFX de balas (`particlePackEffect` / `drawRayResults`).
- Presupuesto de loops server (EnemyAI, EffectRuntime, WorldCardService).
- Localizar y revisar `Script_GameSettings` en el place correcto.
- Evaluar betas de rendering (RenderingCacheOptimizations) tras medir.

---

## 3. Validación (criterio de done por fase)

1. Mismo escenario de prueba: cuarto de la cancha de basket, misma cantidad de enemigos,
   arma automática equipada.
2. MicroProfiler antes/después (dump de 5s, comparar frames p50 y p99).
3. Targets: `Script_Blaster` avg < 0.2ms y picos < 2ms; grupo Script < 8ms en picos;
   frame total < 16.6ms sostenido.
4. QA funcional: auto-disparo detecta igual, aim assist móvil se siente igual,
   VFX de disparos ajenos se ven igual.

## 4. Correcciones a la info base de Dan

1. **Anchored**: mantenerlo, no quitarlo — estático anclado es lo barato para culling.
2. **GameSettings multi-ejecución/frame**: comportamiento normal del profiler con
   múltiples conexiones/threads; hay que ver el script (no está en el hub place).
3. **Input**: los handlers de input del Blaster ya son baratos (solo flags); el costo
   real está en los loops Heartbeat/RenderStep (autoShootTick, aim assist). El consejo
   de Dan es correcto como principio, pero ya lo cumplimos — el fix va por throttling
   y caching de los loops, no por mover lógica de input.
