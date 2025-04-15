# 📊 Mini Guida a `perf` per la Lettura dei Contatori RAPL su Linux

Questa guida mostra **come utilizzare il sottosistema `perf`** per leggere i contatori energetici esposti tramite l’interfaccia **RAPL** su sistemi Linux.

---

## 🧠 Cos'è `perf`?

`perf` è un potente tool per il **performance monitoring** e il **profiling** a basso livello, incluso nel kernel Linux.  
Può essere utilizzato per accedere direttamente agli eventi RAPL definiti dal processore, usando `perf_event_open()`.

---

## ⚙️ Requisiti

### ✅ Permessi richiesti

Per accedere ai contatori RAPL con `perf` servono **privilegi elevati**, a seconda del kernel:

| Versione Kernel | Privilegio richiesto |
|------------------|----------------------|
| ≥ 5.8            | `CAP_PERFMON`        |
| < 5.8            | `CAP_SYS_ADMIN`      |

Oppure imposta:

```bash
echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid
```

---

## 📁 Dove trovare gli eventi RAPL?

Gli eventi disponibili sono elencati in:

```bash
ls /sys/devices/power/events/
```

📌 Ogni file corrisponde a un dominio RAPL (es. `energy-pkg`, `energy-core`, `energy-dram`, ecc.).

Per visualizzare il significato e l'unità di misura:

```bash
cat /sys/devices/power/events/energy-pkg/scale
cat /sys/devices/power/events/energy-pkg/event
```

---

## 🔍 Comandi pratici con `perf`

### 1. 🧪 Leggere il consumo energetico totale del package (per core)

```bash
perf stat -a -e power/energy-pkg/ sleep 1
```

- `-a` → monitora tutti i core  
- `-e` → specifica l'evento da misurare (`power/energy-pkg/`)  
- `sleep 1` → periodo di misura (1 secondo)

**Output tipico:**

```
152.37 J   power/energy-pkg/   # consumo totale CPU (package)
```

---

### 2. 📦 Monitorare più eventi contemporaneamente (es. package + dram)

```bash
perf stat -a \
    -e power/energy-pkg/ \
    -e power/energy-dram/ \
    sleep 1
```

---

### 3. 🔁 Misurare su un singolo socket (es. core 0)

```bash
perf stat -C 0 -e power/energy-pkg/ sleep 1
```

- `-C 0` → specifica la CPU/core

---

## 🛠️ Versione avanzata: polling via `perf_event_open` (programmatico)

Per implementazioni custom in C, Rust o Python via binding, il syscall è:

```c
int fd = perf_event_open(&attr, pid, cpu, group_fd, flags);
```

Dove:

- `attr.type = PERF_TYPE_POWER`  
- `attr.config = evento RAPL desiderato`

📚 Documentazione: `man 2 perf_event_open`

---

## ⚠️ Note e Consigli

- Il valore restituito è in unità arbitrarie → da convertire con lo scale factor (es. `0.000015 = 15 µJ per tick`).  
- `perf` non gestisce overflow di contatori in modo trasparente → meglio usare frequenze elevate e differenze successive.  
- I domini non sono organizzati per socket: se hai più socket, devi aprire un evento per ciascun socket manualmente.

---

## ✅ Riferimenti

- Paper: *Dissecting the software-based measurement of CPU energy consumption: a comparative analysis*  
- Linux Kernel Docs: `/sys/devices/power/events/`  
- `perf_event_open` syscall: man pages  

---

Posso anche prepararti uno **script bash** pronto per loggare i valori ogni secondo su file `.csv` se ti serve per testing. Fammi sapere! 💪
