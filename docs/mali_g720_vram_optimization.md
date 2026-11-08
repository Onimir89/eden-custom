# Ottimizzazione Memoria Vulkan & Fix OOM per ARM Mali-G720 (Android UMA)

Questo documento descrive l'analisi tecnica della root cause, l'obiettivo e i dettagli architetturali delle patch implementate per consentire l'esecuzione stabile di titoli complessi (es. Unreal Engine 5.3) su dispositivi Android con GPU ARM Mali e memoria unificata (UMA).

---

## 1. Dispositivo Target & Dettagli Tecnici

- **Piattaforma**: Android (Linux kernel con LowMemoryKiller / LMK daemon attivo).
- **Dispositivo**: POCO X7 Pro (`2412DPC0AG`), Android 16 (API 36).
- **SoC**: MediaTek MT6899, 8 core Cortex-A725.
- **GPU**: ARM Mali-G720 MC7.
- **Modello di Memoria**: Unified Memory Architecture (UMA) — la memoria fisica è interamente condivisa tra CPU e GPU (assenza di VRAM dedicata).
- **RAM**: `MemTotal` 11,6 GB, condivisa da sistema, emulatore e GPU. Eden riporta 15,0 GB perché conta anche la memoria virtuale aggiuntiva (da confermare).
- **Driver Vulkan**: Versione 1.3.278 (v49.1.0).
- **Titolo di Riferimento**: *FINAL FANTASY RESONANCE* (motore Unreal Engine 5.3).
- **Package Emulatore**: `dev.eden.eden_emulator` (branch `fix/mali-vram-clamp`, base upstream `6e24dcb`).

---

## 2. Il Problema: OOM / LMK Kill al passaggio Gameplay 3D

### Sintomo
Durante la transizione dalla schermata del Titolo al gameplay 3D, il processo dell'emulatore viene terminato improvvisamente con `SIGKILL` (exit code 137) da Android LowMemoryKiller (LMK).

### Rilievi Strumentali (`dumpsys meminfo`)
- `GL mtrack` (memoria Vulkan allocata dal driver GPU e tracciata dal kernel Android) saliva rapidamente fino a **~7.2 GB**.
- Il valore di `GL mtrack` risultava **completamente fisso e invariato** indipendentemente da:
  - `resolution_setup` (0.25X, 0.5X o 1X);
  - `astc_recompression` (Uncompressed o BC3);
  - `vram_usage_mode` (Conservative o Aggressive);
  - `memory_layout_mode` (Memory_4Gb).
- `TOTAL RSS` del processo al momento del kill raggiungeva **~10.7 GB**, superando la soglia critica di memoria disponibile del sistema (12 GB meno memoria del sistema operativo e servizi attivi).

### Log di Errore Concomitanti
```text
[   ...   ] HW.GPU <Warning> video_core/renderer_vulkan/vk_pipeline_cache.cpp:...: Geometry streams is not implemented
[   ...   ] HW.GPU <Error> shader_recompiler/frontend/maxwell/translate/impl/not_implemented.cpp:18: Instruction ATOM (cas) is not implemented
```

---

## 3. Analisi della Root Cause (Indagine nel Codice C++)

L'ispezione approfondita del sorgente ha identificato 5 concause critiche:

### A. Assenza di limiti sull'allocatore VMA (`vulkan_device.cpp:760`)
Nella creazione dell'allocatore `VmaAllocator` (`Device::Device`):
```cpp
const VmaAllocatorCreateInfo allocator_info{
    ...
    .pHeapSizeLimit = nullptr, // Nessun limite configurato!
};
```
Sui dispositivi Mali UMA, l'heap Vulkan riporta l'intera RAM di sistema (~11.5 GB). Poiché `pHeapSizeLimit` era `nullptr`, VMA allocava blocchi continui da 64 MiB senza mai ritornare `VK_ERROR_OUT_OF_DEVICE_MEMORY` e senza forzare la liberazione di memoria.

### B. Fallback al calcolo budget per GPU discreta (`vulkan_device.cpp:1648-1656`)
Nei driver Mali su Android, `properties.properties.deviceType` può non coincidere con `VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU` (spesso riportato come `OTHER`). In tal caso:
```cpp
const size_t normal_memory = 6_GiB;
const size_t scaler_memory = 1_GiB * Settings::values.resolution_info.ScaleUp(1);
device_access_memory = std::min<u64>(device_access_memory, normal_memory + scaler_memory);
```
Poiché `ScaleUp(1)` include `std::max(..., 1)`, restituisce sempre almeno 1 GiB anche a 0.25X/0.5X. Di conseguenza, `device_access_memory` veniva impostato esattamente a **7 GiB (circa 7,5 GB)** fissi. Questo valore da solo non spiega i 7,2 GB misurati: vedi la sezione 5.

### C. Somma RSS Fatale su Host UMA
La memoria totale del processo risultava insostenibile per un dispositivo da 12 GB:
$$\text{RSS Totale} = \underbrace{4.0\text{ GB}}_{\text{Guest Switch RAM (NCE)}} + \underbrace{1.5\text{ GB}}_{\text{Heap Eden / JIT}} + \underbrace{7.2\text{ GB}}_{\text{Vulkan GL mtrack}} = 12.7\text{ GB} > 12.0\text{ GB RAM}$$

### D. Buffer di Staging Spropositato (`vk_staging_buffer_pool.cpp:39`)
`MAX_STREAM_BUFFER_SIZE` era fissato a 256 MiB per tutte le piattaforme non-FreeBSD, allocando permanentemente un buffer di streaming da un quarto di gigabyte in memoria host/device.

### E. Crash su Shader di Geometria (`emit_spirv_special.cpp:157, 170`)
I driver Mali non supportano la feature Vulkan `geometryStreams`. Quando uno shader invocava `EmitEmitVertex` o `EmitEndPrimitive`, il compilatore lanciava `NotImplementedException("Geometry streams")`, impedendo l'emissione dei vertici standard su stream 0.

---

## 4. Obiettivo della Nuova Build & Modifiche Applicate

L'obiettivo della build è stabilizzare l'emulatore su Mali-G720 (12 GB RAM), facendo lavorare il garbage collector come se la memoria Vulkan disponibile fosse 3 GiB, così resta spazio per Android e per la memoria guest Switch.

### Modifiche al Codice Sorgente

1. **Nessun tetto rigido su VMA (`src/video_core/vulkan_common/vulkan_device.cpp`)**:
   - Forzatura `is_integrated = true` su `#ifdef __ANDROID__`.
   - Il tetto `pHeapSizeLimit` a 3 GiB della prima versione è stato tolto: faceva cadere l'emulatore (sezione 5).
   - In `vulkan_memory_allocator.cpp`, se immagini o buffer superano il budget, l'allocazione viene ritentata fuori budget, come già fa `Commit`.

2. **Clamp del Budget Memoria Locale (`src/video_core/vulkan_common/vulkan_device.cpp`)**:
   - Aggiunto `device_access_memory = std::min<u64>(device_access_memory, 3072_MiB)` in `CollectPhysicalMemoryInfo()` nel percorso integrato, l'unico usato su Android.

3. **Riduzione Stream Buffer Pool (`src/video_core/renderer_vulkan/vk_staging_buffer_pool.cpp`)**:
   - Su `#elif defined(__ANDROID__)`, `MAX_STREAM_BUFFER_SIZE = 64_MiB` (risparmio immediato di 192 MiB di memoria persistente).

4. **Fallback Geometry Stream 0 (`src/shader_recompiler/backend/spirv/emit_spirv_special.cpp`)**:
   - In `EmitEmitVertex` ed `EmitEndPrimitive`, se `!ctx.profile.support_geometry_streams` ma `stream == 0`, fallback su istruzioni SPIR-V standard `ctx.OpEmitVertex()` e `ctx.OpEndPrimitive()` invece di generare eccezione.

5. **Integrazione CI Cloud (`.github/workflows/android.yml`)**:
   - Pipeline GitHub Actions automatizzata su runner `ubuntu-latest`.
   - Installazione NDK `28.2.13676358`, CMake `3.31.5`, `glslang-tools` e `ninja-build`.
   - Compilazione tramite `./.ci/android/build.sh -t standard -b Release` e packaging dell'APK pronto all'installazione con firma debug automatica.

---

## 5. Stato e revisione del fix (8 novembre 2026)

### Prova sul dispositivo con il tetto a 3 GiB

Build `79cdc33410`, prova delle 18:10, log letti via `adb`.

- L'emulatore compila circa 107 shader, poi si chiude prima dell'avvio del gioco, dopo 607 secondi di processo.
- Non è un kill di Android. È `SIGABRT` sul thread `GPU`, con questo messaggio:

```text
terminating due to uncaught exception of type Vulkan::vk::Exception: VK_ERROR_OUT_OF_DEVICE_MEMORY
```

Causa: il tetto `pHeapSizeLimit` faceva fallire l'allocazione, `CreateImage` e `CreateBuffer` lanciavano `vk::Exception` e sul thread GPU nessuno la cattura. Nessun meccanismo di Eden libera memoria quando VMA rifiuta un'allocazione, quindi il tetto poteva solo trasformare l'eccesso in un crash. Il backtrace non ha simboli: quale allocazione sia fallita non è noto.

### Correzione applicata

- Tolto il tetto rigido su VMA.
- Immagini e buffer ritentano fuori budget quando VMA risponde `VK_ERROR_OUT_OF_DEVICE_MEMORY`.
- Resta il budget a 3 GiB, che guida il garbage collector: lavora di più da 1,2 GiB e in modo aggressivo da 2,4 GiB.
- Tolto il limite nel ramo non integrato, che su Android non veniva mai eseguito.

La correzione è compilata solo dalla CI. Va provata sul telefono.

### La causa originale non è ancora dimostrata

- `device_access_memory` non è un limite: imposta solo le soglie del garbage collector. Con 7 GiB la soglia critica è 6,2 GiB, non 7.
- Il documento riporta che `GL mtrack` non cambia con `vram_usage_mode`, mentre nel codice quell'opzione cambia il budget. Ipotesi: 7,2 GB è il livello a cui Android uccide il processo, oppure le impostazioni non venivano applicate.
- La somma della sezione 3.C (12,7 GB) non coincide con l'RSS misurato (10,7 GB) e assume che i 4 GB guest siano tutti residenti.
- Senza tetto, il rischio torna a essere il kill di Android se il garbage collector non basta.

### Limiti noti

- Il budget a 3 GiB porta `minimum_memory` a 0: il garbage collector gira a ogni frame e può causare scatti.
- Il budget e `is_integrated = true` valgono per ogni GPU Android, non solo Mali.
- `ATOM (cas)` non è implementata: alla prova compare per una pipeline compute, che non viene creata. Non è la causa del crash, ma può causare errori grafici.

### Prossimi passi, in ordine

1. Installare l'APK con la correzione e ripetere il passaggio al gameplay, registrando `dumpsys meminfo` e `logcat`.
2. Leggere l'esito:
   - ancora `VK_ERROR_OUT_OF_DEVICE_MEMORY`: la memoria è davvero finita nel driver, serve sapere quale allocazione fallisce;
   - exit 137 con kill di `lowmemorykiller`: misurare quale memoria cresce (`GL mtrack`, nativa, guest) e ridurre il budget o la RAM guest;
   - gameplay stabile: valutare gli scatti.
3. Limitare budget e `is_integrated` ai driver ARM (`VK_DRIVER_ID_ARM_PROPRIETARY`) e calcolare il budget dalla RAM.
4. Valutare l'implementazione di `ATOM (cas)`.

### Chiavi su Android

`title.keys` si installa dallo stesso pulsante di `prod.keys`: si sceglie il file e l'app lo copia con il suo nome nella cartella `keys`. Va installato dopo `prod.keys`, altrimenti il controllo finale segnala un errore di inizializzazione.
