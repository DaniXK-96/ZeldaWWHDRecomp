# Prueba de la rama `devel/render-thread-cpu`

> Archivo de trabajo para probar la rama. **Quitarlo antes de abrir el PR** (`git rm docs/testing/render-thread-cpu.md`).

Nada de esta rama se ha compilado ni ejecutado con el juego. Solo pasó: `clang++ -fsyntax-only` de
`draw.cpp` y `buffer_cache.cpp`, y `buffer_cache_test` (también con ASan/UBSan).

## Qué contiene

| Commit | Interruptor | Por defecto | Verificación |
|---|---|---|---|
| Partial fast path por unidad de textura | `WWHD_VK_FASTPATH_PARTIAL=1` (ya existente) | apagado | `WWHD_VK_FASTPATH_VERIFY=N` |
| Tabla frontal de la caché de búfers | `WWHD_VK_BUFFER_CACHE_FRONT=1` | apagado | `WWHD_VK_BUFFER_CACHE_VERIFY=1` |
| Memo de stage (incluye el parche de offsets de la hierba) | `WWHD_VK_STAGE_MEMO=1` | apagado | `WWHD_VK_FASTPATH_VERIFY=N` |

Con todo apagado, el juego debe comportarse exactamente como upstream `devel`.

## Android: dónde poner las variables

En `Android/data/<paquete>/files/env.txt`, una por línea (`NOMBRE=valor`). El log queda en
`captures/wwhd.log`.

## Paso 1: compila y arranca con todo apagado

Arrancar, cargar una partida, jugar un minuto. Debe verse y funcionar igual que antes.

## Paso 2: verificación (no mide rendimiento, es lento a propósito)

`env.txt`:

```
WWHD_VK_FASTPATH_PARTIAL=1
WWHD_VK_STAGE_MEMO=1
WWHD_VK_BUFFER_CACHE_FRONT=1
WWHD_VK_FASTPATH_VERIFY=16
WWHD_VK_BUFFER_CACHE_VERIFY=1
WWHD_VK_STATS=1
```

Recorrer varios lugares, unos minutos en cada uno: aldea (Outset), bosque de hierba alta, Windfall, un
interior, una mazmorra. Pasar por cargas de zona y, si se puede, cargar un save state.

Buscar en el log:

| Línea | Qué debe decir |
|---|---|
| `[vulkan] continued-draw fast path on (verify mode) (partial continuation) (stage memo)` | que todo está activo |
| `[vulkan fastpath] ... stage memo: vs X% ... ps Y% ...` | porcentaje de stages tomados del memo. Interesa que no sea ~0 % |
| `[vulkan fastpath] ... (verify: ... mismatches; total 0 of N)` del stage memo | **0 mismatches** |
| `[vulkan fastpath] ... ; verify: ... mismatches (total 0 of N)` | **0 mismatches** |
| `[vulkan buffer cache] front table: ...; verify: 0 mismatches` | **0 mismatches** |
| `[vulkan buffer cache] verify: ... 0 mismatches` | **0 mismatches** |
| `[vulkan stage memo] verify mismatch` | **no debe aparecer**. Si aparece, copiar las líneas |
| `[vulkan fastpath] verify mismatch` | **no debe aparecer** |

Nota: un mismatch del stage memo con `set same` y los mismos offsets en número, pero valores distintos,
puede ser benigno (el bloque se volvió a subir a otra región con los mismos bytes). Igual hay que
reportarlo con la línea completa.

## Paso 3: medir (sin verificación)

Medir cada variante en el mismo lugar y con la misma cámara. Mejor en el bosque de hierba alta (el
peor caso) y en la aldea.

Variantes (una por arranque):

| Nombre | `env.txt` |
|---|---|
| A (base) | `WWHD_VK_PROF_SPLIT=1` |
| B | A + `WWHD_VK_FASTPATH_PARTIAL=1` |
| C | B + `WWHD_VK_BUFFER_CACHE_FRONT=1` |
| D | C + `WWHD_VK_STAGE_MEMO=1` |

En todas, encender **Settings > Graphics > Detailed performance profiling** (o `WWHD_PROFILE=1`).
Esperar al menos 2 informes (cada 120 frames) y copiar del log:

- `[prof] draw phases us/draw by fast-path class: ...` (coste por draw continuado y completo)
- el tiempo de CPU del render thread del informe del perfilador
- FPS

Para la estadística del memo sin verificación, añadir `WWHD_VK_STATS=1` en D (añade un poco de coste).

En PC (Linux/Windows/macOS) también sirve `tools/bench/run_bench.py` con
`--variant A: --variant D:WWHD_VK_FASTPATH_PARTIAL=1,WWHD_VK_BUFFER_CACHE_FRONT=1,WWHD_VK_STAGE_MEMO=1`.

## Paso 4: imagen idéntica

En PC: `tools/bench/run_picture_check.py` con la variante D frente a la base. En el teléfono, capturas
(F10 o el atajo de captura) en el mismo punto con A y con D.

## Paso 5: contadores para decidir las optimizaciones de GPU (solo diagnóstico)

No cambian el renderizado. Un arranque con:

```
WWHD_VK_CLEAR_STATS=1
WWHD_VK_FEEDBACK_STATS=1
```

En la aldea, el bosque, un interior y una mazmorra, copiar del log (cada 120 frames):

- `[vulkan clears] X/frame, Y of whole images; ... render pass ... Z/frame (could load as a clear), another use W/frame`.
  Si Z es alto, convertir esos clears en `LOAD_OP_CLEAR` ahorra una escritura y una lectura completas de
  cada imagen por frame en la GPU del teléfono.
- `[vulkan feedback copies] ... N repeats of an unchanged source/frame (M M texels/frame) ...`.
  Si N es alto, guardar la copia y reutilizarla ahorra una copia completa y un corte de pass por cada
  repetición.

Son estimaciones: cuentan lo que se podría ahorrar, no lo ahorran.

## Qué informar

1. Si compila (y el error, si no).
2. Las líneas de verificación del paso 2 (los totales de mismatches).
3. Las líneas `[prof] draw phases ...` y el tiempo del render thread de A, B, C y D en cada lugar.
4. Las líneas `[vulkan clears]` y `[vulkan feedback copies]` del paso 5.
5. Cualquier diferencia visible.
