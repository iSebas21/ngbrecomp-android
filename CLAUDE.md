# CLAUDE.md — Port de Ninja Gaiden Black Recompiled a Android

Este fork tiene un objetivo: llevar el port de PC (Linux/Windows/macOS) a **Android ARM64** como APK
nativo, sin emulación de CPU.

**Lee primero `AGENTS.md`**: ahí están el layout del proyecto, cómo compilar y las reglas de pruebas
(no tocar saves/config reales, correr sin audio y en gamescope headless, matar procesos solo por PID,
limitar memoria en builds grandes). Esas reglas siguen vigentes. `docs/TECHNICAL.md` explica cómo
funciona todo el runtime.

## Reglas de este fork

- **Nunca hacer commit de código generado ni datos del juego**: `src/recomp/gen*/`, `game_files/`,
  `disc/`, `*.xbe`, `*.afs`, `*.iso`, saves, capturas. El código C del juego se genera desde el disco
  del usuario con `tools/pipeline.sh`.
- No romper los targets existentes. Todo cambio para ARM/Android va detrás de
  `#if defined(__linux__) && defined(__aarch64__)` o `#if defined(__ANDROID__)`, y Linux x86-64,
  Windows y macOS deben seguir compilando igual que antes.
- Imitar el estilo del código vecino (ver "Conventions" en `AGENTS.md`). Commits pequeños.
- El proyecto es GPL-3.0-or-later: si se publica el fork, se publica con código fuente.

## Estado de partida (lo que ya es portable)

- **Memoria del guest**: el código generado accede vía offset, no en direcciones fijas
  (`XBOX_PTR(addr) = (uint32_t)addr + g_xbox_mem_offset`, en
  `xboxrecomp/templates/runtime/recomp_types.h`). Reservar la ventana de 4 GB en Android ARM64 es viable.
- **x87**: emulado con `double`; los intrínsecos SSE en `recomp_types.h` ya tienen rama no-x86.
- **MMIO por trampa**: `xboxrecomp/src/platform/mmio_decode_a64.h` ya decodifica AArch64 y **ya
  contempla Linux arm64** (`uc_mcontext.regs`, `fpsimd_context` en `__reserved`). Android usa el mismo
  layout de `ucontext`.
- **Fibras**: `src/fibers.c` ya tiene `ngb_ctx_switch` en AArch64 (AAPCS64), pero solo para Apple.
- **Ventana/input/audio**: SDL2, que tiene soporte oficial de Android (`SDLActivity`).
- **Gráficos**: renderer propio de D3D sobre Vulkan 1.3 (`src/d3d/vk/backend.cpp`) más el modelo NV2A
  de xemu (`xboxrecomp/src/nv2a_vk`).
- El port de macOS (#49) resolvió casi todo lo de ARM64: **úsalo como plantilla**. Busca
  `__APPLE__` (~57 apariciones en `src/` y `xboxrecomp/src/`) y para cada una decide si Linux
  aarch64/Android necesita la misma rama, la rama Linux existente o una nueva.

## Plan por fases

### Fase 1 — Linux aarch64 (sin Android todavía)

Objetivo: que el juego corra en Linux ARM64 (Raspberry Pi 5, VM ARM o similar). Valida CPU, memoria,
señales y fibras sin pelear aún con drivers móviles. Hoy el código asume "macOS = ARM" y
"Linux = x86-64", así que en Linux aarch64 cae en ramas x86 y no compila.

Puntos conocidos a cambiar:

1. `xboxrecomp/src/platform/win32_compat.c`, `veh_ctx_from` / `veh_ctx_to` (~línea 2367): la rama no
   Apple usa `gregs[REG_RIP]` etc. (x86-64). Añadir rama `__linux__ && __aarch64__` análoga a la de
   Apple: `c->Rip = uc_mcontext.pc`, `Rsp = sp`, `Rbp = regs[29]`, `native_context = uc`, y en
   `veh_ctx_to` dejar que el decodificador ARM64 actualice el contexto nativo. Revisar también
   `veh_code` (`TRAP_TRACE`/single-step no existe igual en ARM: confirmar cómo lo resolvió macOS).
2. `xboxrecomp/src/platform/vcpu.c` (~líneas 267-345): la señal de preempción (`SIG_PREEMPT`, en Linux
   `SIGRTMIN + 4`). En Apple usa `SA_SIGINFO` y filtra por PC dentro de `__TEXT` para no ceder mientras
   libc tiene un lock. Evaluar si bionic (Android) necesita el mismo filtro; en Linux usar
   `dl_iterate_phdr` o los símbolos `__executable_start`/`etext` para el rango de texto.
3. `src/fibers.c` (~línea 60): la rutina `ngb_ctx_switch` en AArch64 tiene sintaxis Mach-O
   (`_ngb_ctx_switch`). Duplicarla en sintaxis ELF (`ngb_ctx_switch`, `.type ...,@function`, `.size`)
   para `__linux__ && __aarch64__`. Revisar también la línea ~195 (`#ifdef __APPLE__`).
4. `xboxrecomp/src/nv2a_vk/host/nv2a_dirty.c` (~línea 283): en macOS el rastreo de páginas sucias está
   desactivado porque las páginas del host son de 16 KB. En Linux con páginas de 4 KB debería poder
   quedar activo; comprobar `sysconf(_SC_PAGESIZE)` en tiempo de ejecución en lugar de depender de
   `__APPLE__`.
5. `CMakeLists.txt`: el comentario de la línea 6 dice "Linux (x86-64)". Revisar flags específicos de
   x86 y cualquier ruta que asuma x86 en Linux. Comprobar `tools/pipeline.sh` y
   `xboxrecomp/CMakeLists.txt`.
6. Resto de `#ifdef __APPLE__` en `src/main.c`, `src/port.h`, `src/install.c`, `src/launch.c`,
   `src/ingame_menu.c`, `xboxrecomp/src/kernel/xbox_memory_layout.c`,
   `xboxrecomp/src/nv2a_vk/host/nv2a_host.c`, `present_sdl.c`, `mmio_decode.h`, `xbox_winnt.h`.

Primer paso útil sin el disco: compilar solo el runtime (`xboxrecomp`) con un cross-compiler
aarch64 (`clang --target=aarch64-linux-gnu` + sysroot, o en un contenedor ARM) para obtener la lista
real de errores.

Criterio de salida: llega al menú principal y se puede jugar el capítulo 1 en Linux aarch64.

### Fase 2 — Dependencias con el Android NDK

- **SDL2**: usar su proyecto Android (`android-project/`) como esqueleto del APK.
- **GLib**: la exige el modelo NV2A (`xboxrecomp/src/nv2a_vk/CMakeLists.txt`, `pkg_check_modules glib-2.0`).
  Es la dependencia más difícil de compilar para Android (meson con cross-file del NDK). Valorar si
  basta con un subconjunto o un shim.
- **glslang + SPIRV-Tools**: los shaders se compilan en tiempo de ejecución; compilar ambos con el NDK.
- **Vulkan**: volk/VMA ya vienen de `xboxrecomp/third_party/fetch.sh`; en Android se carga
  `libvulkan.so` del sistema.
- **Qt** (`ngb-setup`): no existe en Android; queda fuera del build (ya es opcional en `CMakeLists.txt`).
- Toolchain: `-DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-29` (ajustar API mínima según `memfd_create`
  y Vulkan 1.3). El ejecutable pasa a ser una biblioteca compartida (`libmain.so`) que carga
  `SDLActivity`.

### Fase 3 — GPU móvil (el mayor riesgo)

- `src/d3d/vk/backend.cpp:690` **exige `textureCompressionBC`** (texturas DXT, `BC1/BC2/BC3` en
  ~línea 161). Los drivers propietarios de Adreno y Mali normalmente no lo exponen, y entonces falla
  la creación del dispositivo. Opciones:
  - Fallback: si `textureCompressionBC` no está, decodificar DXT a RGBA8 en CPU al subir la textura.
    El renderer ya decodifica DXT para generar mipmaps: reutilizar ese código.
  - Soportar drivers Turnip en Adreno (como hace el port Android de Unleashed Recompiled).
- Otras features usadas cuyo soporte varía en móvil: `VK_KHR_push_descriptor`, `fillModeNonSolid`,
  `logicOp`, `depthClamp`, `customBorderColor`, `geometryShader` (en el modelo xemu). Hacer opcional
  todo lo que no sea imprescindible y degradar con aviso en el log.
- Formato depth/stencil: `backend.cpp` ~líneas 479 y 743 eligen `D24_UNORM_S8_UINT` o
  `D32_SFLOAT_S8_UINT`; verificar que al menos uno esté disponible y, si no, añadir
  `D32_SFLOAT` + stencil aparte o el formato que sí exista.
- `xboxrecomp/src/nv2a_vk/xemu/pgraph/vk/instance.c` (~línea 580): en macOS relaja las features
  requeridas del dispositivo NV2A cuando se usa el renderer `d3d`. Android necesita lo mismo.
- Probar en Adreno primero (mejor soporte de Vulkan), luego Mali como experimental.

### Fase 4 — Capa Android

- Instalador en la app: elegir el ISO/7z/zip o una carpeta con el Storage Access Framework y extraer
  (~6 GB) al almacenamiento de la app. Sustituye la ventana Qt.
- Rutas: saves, `settings.ini`, `controls.ini` y caché de shaders en
  `SDL_AndroidGetInternalStoragePath()` / `SDL_AndroidGetExternalStoragePath()` (ver la tabla
  "Where things are kept" del `README.md` y `src/port.h`).
- `src/launch.c`: el cambio al arcade Ninja Gaiden hace `exec` en Linux. En Android no hay `exec` de
  un segundo binario; habrá que empaquetar el arcade como otra `.so` y cambiar en el mismo proceso,
  o lanzar otra Activity.
- Ciclo de vida: pausar el juego y el audio en `onPause`, recrear la swapchain al volver.
- Controles: mando Bluetooth vía SDL primero; controles táctiles en pantalla después.
- Rendimiento: `PerformanceHintManager` / modo de rendimiento sostenido; vigilar throttling térmico.

## Páginas de 16 KB

El Xbox usa páginas de 4 KB y el runtime depende de esa granularidad (alias/espejos de memoria,
páginas MMIO con `PROT_NONE`, rastreo de páginas sucias). La mayoría de Android actuales usan 4 KB,
pero los dispositivos nuevos están pasando a 16 KB (y Google Play lo exige a las apps nativas nuevas).
macOS ya tiene ese problema y lo mitigó parcialmente: estudiar sus cambios y no asumir 4 KB; leer el
tamaño de página en tiempo de ejecución.

## Pruebas en Android

- `adb logcat` para el log del juego (redirigir `stderr`/`log.txt` a logcat con `__android_log_print`
  o la tag de SDL).
- Capa de validación de Vulkan para Android (`libVkLayer_khronos_validation.so` en `jniLibs`) cuando
  algo falle solo en un GPU.
- Las variables de entorno de depuración del `AGENTS.md` (`NV2A_FPS_LOG`, `NGB_RENDERER=xemu`,
  trazas con F7, etc.) se pueden fijar desde la Activity antes de cargar `libmain.so`.

## Referencias

- Port Android de Unleashed Recompiled (Xbox 360, XenonRecomp): estructura del APK, instalador en la
  app, carga de drivers Turnip. https://github.com/SansNope/UnleashedRecomp-Android
- xboxrecomp upstream: https://github.com/sp00nznet/xboxrecomp
- Plantilla Android de SDL2: `android-project/` en el repo de SDL2.
