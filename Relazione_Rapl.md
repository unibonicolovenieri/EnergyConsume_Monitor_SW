# Relazione sul Paper di RAPL: Analisi e Sintesi.

## III. Analisi Comparativa dei Meccanismi Basati su RAPL

### A. Funzionamento dell’interfaccia RAPL

**RAPL** (*Running Average Power Limit*) è una tecnologia di gestione energetica introdotta da **Intel** con l’architettura **Sandy Bridge** nel 2011. Permette di **misurare e limitare** il consumo energetico di varie parti del computer, chiamate **domini**.  
In questo studio si analizza solo l’interfaccia di misura, non quella di limitazione.

Anche **AMD**, a partire dalla prima architettura **Zen** (2017), ha adottato una soluzione simile. L’uso è sostanzialmente identico, ma cambiano gli **indirizzi dei registri** e il **numero di domini disponibili**.

#### Osservazioni sulle gerarchie dei domini:

- I domini `core` e `uncore` sono **sottoinsiemi** del dominio `package`.
- Il dominio `platform` (o `psys`) **varia in base al produttore** del dispositivo. Test su due laptop (Lenovo e Alienware) hanno mostrato che `psys` corrisponde al **consumo totale del laptop**.
- Il dominio `dram` è associato a un **singolo socket**.
- AMD (Zen 1 - Zen 4) supporta solo `package` e `core`.

Ogni dominio ha un **contatore energetico** che si aggiorna ogni ~976 µs. La **differenza** tra due letture consecutive del contatore rappresenta l’energia consumata. Leggere più frequentemente del tasso di aggiornamento **non ha senso**: il valore rimane invariato fino al prossimo aggiornamento.

---

### B. Criteri di Confronto

I meccanismi permettono di ottenere i valori dei contatori RAPL. La scelta del meccanismo comporta dei compromessi. I criteri di valutazione sono:

- **Difficoltà tecnica** per l’implementazione
- **Conoscenze richieste**, es. dettagli della microarchitettura
- **Salvaguardie** contro errori (robustezza)
- **Privilegi richiesti** per l’esecuzione
- **Resilienza**, ovvero adattabilità a nuovi processori/domìni

---

### C. Registri Specifici del Modello (MSR)

Il meccanismo **MSR** è il più basso livello. I registri si leggono tramite `/dev/cpu/N/msr`, dove `N` è il numero del core. È necessario conoscere gli **offset** specifici del modello del processore.

#### Passaggi:
1. Leggere `MSR_RAPL_POWER_UNIT` → restituisce l’unità di misura.
2. Leggere `MSR_d_ENERGY_STATUS` per ciascun dominio `d`.
3. Convertire in Joule (moltiplicando per il valore letto al punto 1).

⚠️ **Nessuna protezione** contro errori:
- Letture errate → risultati errati.
- Non causa danni (solo lettura), ma può portare a misure completamente sbagliate.

📛 Attenzione agli **overflow**: dopo circa **60 secondi di carico pesante**, i contatori si azzerano. Bisogna effettuare polling frequente per rilevarli.

🔐 Accesso limitato: serve `CAP_SYS_RAWIO` o `CAP_SYS_ADMIN`.  
Il modulo kernel `msr` non offre un controllo fine sui registri accessibili.

📚 È richiesta **conoscenza tecnica avanzata**, specialmente sugli offset e su eventuali “quirk” di microarchitettura (es. unità fissa in alcuni Intel).

---

### D. Power Capping Framework (powercap)

Il framework **Powercap** è un’interfaccia kernel di alto livello per RAPL, disponibile tramite `sysfs`:

/sys/devices/virtual/powercap/intel-rapl


Questa gerarchia riflette la struttura dei domini RAPL. Ogni sotto-cartella corrisponde a un dominio.

🧠 Il dominio `dram` è annidato dentro `package`, probabilmente per rappresentare la memoria direttamente connessa al socket.

### Vantaggi:

- ✅ Dati già convertiti in testo leggibile (es. Joule)
- ✅ Non servono conoscenze tecniche avanzate
- ✅ Nessuna capacità richiesta, solo **permessi di lettura**
- ⚠️ Anche qui servono **correzioni per overflow**
- ⚙️ Buona **resilienza**: la struttura `sysfs` si adatta automaticamente ai domini disponibili (se non è hardcoded)

---

### E. perf-events (userspace)

**perf-events** è un sottosistema Linux per il **monitoraggio basato su eventi**. Supporta:
- **Eventi di conteggio** (es. i contatori energetici)
- **Eventi di campionamento** (con buffer, overflow gestito automaticamente)

I contatori RAPL sono **eventi di conteggio**, quindi:
- Bisogna fare **polling periodico**
- Gli overflow sono **rari** perché già gestiti internamente
- ⚠️ Overflow teorico possibile (variabili a 64 bit)

#### Come si usa:
1. Leggere gli eventi da `/sys/devices/power/events/`
2. Aprirli con `perf_event_open()`
3. Fare polling del file descriptor

⚠️ Non è presente una **gerarchia per socket** → bisogna aprire **un evento per socket**.

🛠️ Non richiede esperienza avanzata.  
🔐 Serve `CAP_PERFMON` (o `CAP_SYS_ADMIN`, o settare `perf_event_paranoid = 0`)

📦 **Buona resilienza**, facile da mantenere  
❌ Più difficile da usare in script (es. Bash)

---

### F. perf-events tramite eBPF

**eBPF** permette di iniettare codice nel kernel (Linux ≥ 3.15) e agganciarlo a eventi kernel come `SF_CPU_CLOCK`.

🧪 Gli autori propongono un nuovo meccanismo:
- Si chiama `perf_event_open()`
- I file descriptor vengono passati a un programma eBPF
- Il codice eBPF legge i contatori e scrive in un buffer
- Un thread in user-space fa polling del buffer

🧠 **Alta difficoltà tecnica**, specialmente nel passaggio dati kernel ↔ user space.  
📛 Errori possibili su dimensioni buffer, compatibilità e aggiornamenti

🔐 Serve `CAP_BPF` (o `CAP_SYS_ADMIN`)

⚠️ Poco flessibile: alcune versioni del kernel non supportano `loop` in eBPF → servono `match-case` hardcoded per gestire nuovi domini


## IV. Implementazione di uno Strumento Minimo

### A. Architettura

Abbiamo implementato uno **strumento minimo per la misurazione** del consumo energetico, utilizzando i quattro meccanismi descritti nella sezione precedente.

L’architettura è rappresentata nella **Figura 4** del paper.

Lo strumento include una **interfaccia a riga di comando (CLI)** che permette all’utente di specificare:
- i **domini RAPL** da monitorare
- il **meccanismo** da utilizzare
- la **frequenza di acquisizione**
- il **file di output**

Per i benchmark, abbiamo scelto di scrivere sempre le misurazioni in un file **CSV**, così da simulare una situazione realistica.  
👉 Scartare i valori subito dopo la lettura non avrebbe avuto senso, e avremmo perso il costo effettivo della scrittura.

Per ridurre il carico I/O alle alte frequenze di campionamento, abbiamo scelto di **scrivere su file una volta al secondo**.

#### Vantaggi rispetto agli strumenti esistenti:

- **Accesso ai dati “grezzi”** (raw) del consumo energetico, non a stime come nel caso di CodeCarbon [20].
- Possibilità di **scegliere tra tutti i meccanismi disponibili**, utile per fare test comparativi.
- **Maggiore robustezza**: se un meccanismo è buggato, si può cambiarlo.
- Rilevati comportamenti incoerenti su un server AMD: `powercap` riportava solo due domini (`package` e `core`), mentre `perf-events` ne elencava molti (ma inutilizzabili).  
  ⚠️ Il dominio `core` restituiva valori sospettosamente bassi (~0.001 J/s).  
  Questo bug è stato risolto in **Linux 6.6**.
- Lo strumento gestisce correttamente:
  - problemi di overflow
  - frequenze di misura elevate

Maggiori dettagli sono disponibili nel **[repository pubblico GitHub](https://github.com/TheElectronWill/cpu-energy-consumption-comparative-analysis)**.

---

### B. Correzione dell’Overflow dei Contatori

Come detto nella sezione III-C, i contatori energetici RAPL possono andare in **overflow**.

🔍 La frequenza dell’overflow dipende da:
- il **dominio RAPL**
- il **meccanismo di misura**, perché ciascuno usa **variabili di dimensioni diverse**

#### Condizioni per una misura corretta:
1. L’intervallo tra due letture deve essere **minore del tempo di overflow**
2. Se si verifica un overflow, bisogna correggere il valore con questa formula:

```plaintext
Δm = 
    mcurrent - mprev + C   (se mcurrent < mprev)
    mcurrent - mprev       (altrimenti)

dove C è una costante di correzione che varia in base al meccanismo usato.
```

### C. Accuratezza della Frequenza di Acquisizione

La **frequenza di acquisizione** è un parametro critico.  
Frequenze elevate permettono una **profilazione più dettagliata** del comportamento dell'applicazione (es. stima dell'energia di una singola funzione [44]).

Per questo motivo, è essenziale:
- ✅ Permettere all'utente di **scegliere la frequenza**
- ✅ Garantire che lo strumento **la rispetti con precisione**

Durante lo sviluppo, si è osservato che l'uso di:

- `std::thread::sleep` (in Rust)
- `nanosleep` (in C)

non era **sufficientemente preciso**.  
Era **impossibile raggiungere i 1000 Hz**, e la frequenza misurata era soggetta a **forti variazioni**.

💡 **Soluzione adottata**: utilizzo di **`timerfd`**, un timer più accurato disponibile in Linux.

> Anche se non garantisce una precisione assoluta (su kernel non real-time), `timerfd` è **nettamente migliore dello sleep classico**.

Inoltre, per minimizzare l’impatto I/O:

- Il **loop di polling** esegue solo l’acquisizione dei dati.
- La **scrittura su file** è delegata a un **secondo thread**.

👉 Questo **riduce la latenza** causata dall’output su disco e migliora la stabilità della frequenza di campionamento.

---

### Tabella: Costanti di Correzione Overflow (C)

| **Meccanismo**         | **Costante C**                        |
|------------------------|----------------------------------------|
| MSR                    | `u32::MAX` → `0xFFFFFFFF`             |
| perf-events            | `u64::MAX` → `0xFFFFFFFFFFFFFFFF`     |
| perf-events + eBPF     | `u64::MAX` → `0xFFFFFFFFFFFFFFFF`     |
| powercap               | Valore letto da `max_energy_uj` (in sysfs) |

---

### D. Ottimizzazioni e Risultati

Per valutare l'impatto delle ottimizzazioni, sono state implementate **tre versioni** dello strumento:

- 🟥 **fully optimized**: utilizza `timerfd` + thread separato per la scrittura su file
- 🔵 **badsleep**: usa `sleep` classico + thread separato per scrittura
- 🟢 **badsleep-st**: `sleep` classico **senza** thread separato (tutto in un unico thread)

Ogni variante è stata testata a **1000 Hz** per almeno **5 minuti**.  
È stato misurato il **numero effettivo di campioni/secondo**, escludendo il primo e ultimo secondo.

📊 **Figura 5** (del paper) mostra:

- ✅ Solo la versione **ottimizzata** mantiene stabilmente i **1000 Hz**
- ❌ Le versioni con `sleep` classico **perdono tra il 10% e il 18%** della frequenza target

---

📌 **Osservazione interessante**:  
> Usare un **thread separato per la scrittura** **non porta benefici** se **non si usa anche `timerfd`**.
