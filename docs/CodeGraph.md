# CodeGraph

Questo progetto usa [CodeGraph](https://www.npmjs.com/package/@colbymchenry/codegraph) come indice del codice: simboli, chiamanti, chiamati e impatto di una modifica. Vale per chiunque ci lavori, persone e agenti. Prima di cercare a mano in `src/`, interroga l'indice.

## Dove si trova il database

Il database sta sempre in questo percorso, relativo alla radice del repo:

```
.codegraph/codegraph.db
```

Sul PC Windows del manutentore il percorso completo è:

```
C:\Games\Eden\src\eden\.codegraph\codegraph.db
```

Il database non è versionato: pesa circa 200 MB, supera il limite di GitHub ed è locale a ogni macchina. Chi clona il repo non lo trova e deve crearlo una volta.

## Creare l'indice

Serve Node.js. Dalla radice del repo:

```
npm install -g @colbymchenry/codegraph
codegraph init -y
```

Sul PC del manutentore, con la versione 1.6.2, l'indicizzazione richiede circa 30 secondi e copre circa 3400 file.

## Tenerlo aggiornato

Dopo un `pull`, un cambio di branch o una modifica ai sorgenti:

```
codegraph sync
```

`codegraph status` mostra file, simboli e dimensione del database.

## Comandi utili

| Domanda | Comando |
|---|---|
| Dove è definito questo simbolo? | `codegraph query CollectPhysicalMemoryInfo` |
| Sorgente di un simbolo, con chiamanti e chiamati | `codegraph node CreateImage` |
| Chi chiama questa funzione? | `codegraph callers CreateImage` |
| Cosa si rompe se la cambio? | `codegraph impact CreateImage` |
| Panoramica di un'area del codice | `codegraph explore texture cache garbage collector` |

Da fuori dalla radice del repo aggiungi `-p <percorso-del-repo>`.

## Agenti

- CodeGraph è l'unico indice del codice di questo progetto. Non indicizzarlo in altri strumenti.
- Con `codegraph install` CodeGraph si collega come server MCP agli agenti supportati. È facoltativo: i comandi qui sopra funzionano anche senza.
- Da WSL, se il repo sta su un disco Windows e CodeGraph è installato su Windows, `codegraph` si blocca. Lancialo così:

```
cmd.exe /c "codegraph status C:\Games\Eden\src\eden"
```

## Dove sta il resto della conoscenza

Stato, decisioni e problemi aperti del progetto stanno nella documentazione del repo, così chiunque può leggerli. Il lavoro sulla memoria Vulkan è in [mali_g720_vram_optimization.md](./mali_g720_vram_optimization.md).
