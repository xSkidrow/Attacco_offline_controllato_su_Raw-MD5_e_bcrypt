<p align="center">
  <img src="./assets/banner-password-offline-lab.png"
       alt="Banner del progetto"
       width="100%">
</p>

# Analisi sperimentale della sicurezza delle password in uno scenario di attacco offline

## Presentazione del progetto

Questo repository contiene l’analisi integrale sviluppata per l’elaborato del corso di **Principi e Metodi di Crittografia**. Il progetto studia, in un ambiente controllato, quanto la funzione di hash utilizzata per proteggere una password influenzi la velocità con cui un attaccante può verificarne le possibili candidate dopo aver ottenuto una copia degli hash.

L’esperimento confronta **Raw-MD5**, funzione veloce e priva di salt, con **bcrypt**, configurato con salt casuale e costo 12. La stessa password sintetica viene trasformata con entrambi i metodi e sottoposta a un attacco offline mediante **John the Ripper**, utilizzando la medesima wordlist di 20.000 candidate.

La relazione accademica costituisce una sintesi dei passaggi e dei risultati principali. Il Jupyter Notebook presente in questo repository documenta invece l’intera analisi sperimentale, includendo configurazione dell’ambiente, generazione controllata dei dati, esecuzione degli attacchi, validazione dei risultati, misurazioni, tabelle, grafici e prove supplementari su CPU e GPU.

## Obiettivo dell’analisi

Lo scopo non è valutare la robustezza della singola password, ma isolare l’effetto della funzione di hash mantenendo invariate tutte le altre condizioni sperimentali. Per questo motivo Raw-MD5 e bcrypt vengono confrontati usando la stessa password, la stessa wordlist, la stessa posizione della soluzione e il medesimo strumento di verifica.

L’esperimento consente di osservare concretamente la differenza tra una funzione progettata per essere molto veloce e una funzione specificamente concepita per la protezione delle password. Il salt non rende impossibile indovinare una password presente nella wordlist, ma impedisce di riutilizzare lo stesso calcolo tra hash protetti da salt differenti e rende inutilizzabili le rainbow table universali. Il costo di bcrypt, invece, aumenta deliberatamente il lavoro necessario per ogni singolo tentativo.

## Scenario sperimentale

| Elemento                    | Configurazione                                       |
| --------------------------- | ---------------------------------------------------- |
| Scenario                    | Attacco offline controllato                          |
| Password                    | Sintetica, lunga 17 caratteri                        |
| Composizione                | Maiuscole, minuscole, cifre e simboli                |
| Wordlist                    | 20.000 candidate uniche                              |
| Posizione della soluzione   | Ultima riga                                          |
| Primo formato               | Raw-MD5 senza salt                                   |
| Secondo formato             | bcrypt con salt casuale e costo 12                   |
| Strumento                   | John the Ripper Jumbo                                |
| Ambiente                    | Python 3.10 e Jupyter Notebook                       |
| Accelerazione supplementare | NVIDIA GeForce RTX 5070 Ti Laptop GPU tramite OpenCL |

La collocazione della password all’ultima posizione obbliga John the Ripper a esaminare l’intera wordlist prima di trovare la soluzione. In questo modo entrambi i formati vengono sottoposti allo stesso numero di candidate e il confronto non dipende dalla posizione casuale della password.

La prova utilizza un **attacco a dizionario basato su wordlist** e non una ricerca esaustiva di tutte le combinazioni possibili. La wordlist è stata generata esclusivamente per il laboratorio e contiene soltanto dati sintetici.

## Generazione degli hash

La password viene codificata in UTF-8 e trasformata nei due formati previsti dall’esperimento:

```python
# Converte la password sintetica in byte.
password_bytes = PASSWORD.encode("utf-8")

# Raw-MD5 produce un'impronta deterministica senza salt.
md5_hash = hashlib.md5(password_bytes).hexdigest()

# bcrypt genera un salt casuale e incorpora il costo 12.
bcrypt_salt = bcrypt.gensalt(rounds=12)
bcrypt_hash = bcrypt.hashpw(
    password_bytes,
    bcrypt_salt,
).decode("utf-8")
```

Raw-MD5 produce sempre la stessa impronta quando riceve la medesima password. bcrypt genera invece stringhe differenti grazie all’impiego di salt casuali, anche quando password e costo rimangono invariati.

Due stringhe bcrypt generate dalla stessa password, con salt casuali distinti e costo 12, sono infatti risultate diverse. Per facilitarne il confronto senza riportarle integralmente, nel notebook sono mostrate le rispettive impronte SHA-256 abbreviate:

```text
2df99491de099800
baa72fb02ca5a306
```

Entrambe le stringhe superano comunque la verifica con la password originale. La differenza osservata dipende quindi dal salt e non da una modifica della password.

## Esecuzione con John the Ripper

John the Ripper viene avviato separatamente sui due file di hash, mantenendo invariata la wordlist. File `.pot` distinti impediscono che il risultato di una prova precedente venga riutilizzato nelle esecuzioni successive.

La validazione non si basa soltanto sul codice di uscita del processo. Il notebook controlla anche l’output di `--show`, la presenza della password recuperata e il numero di hash ancora da individuare.

Esempio generale dei comandi utilizzati:

```powershell
.\JtR\run\john.exe --format=Raw-MD5 --wordlist=wordlists/wordlist_controllata.txt hashes/hash_md5.txt

.\JtR\run\john.exe --format=bcrypt --wordlist=wordlists/wordlist_controllata.txt hashes/hash_bcrypt.txt
```

Nel notebook i percorsi, i file `.pot`, la pulizia delle sessioni e la raccolta dei risultati sono gestiti automaticamente da Python.

## Risultati principali

Entrambe le password sono state recuperate perché la candidata corretta era presente nella wordlist. La differenza sostanziale riguarda il numero di tentativi verificabili nell’unità di tempo.

| Formato | Salt |           Costo | Candidate effettive al secondo | Esito      |
| ------- | ---: | --------------: | -----------------------------: | ---------- |
| Raw-MD5 |   No | Non applicabile |                       82.440,2 | Recuperata |
| bcrypt  |   Sì |              12 |                          324,6 | Recuperata |

Nella prova end-to-end Raw-MD5 ha verificato le candidate con una velocità effettiva circa **254 volte superiore** a bcrypt. Questo risultato non significa che bcrypt renda impossibile l’indovinamento di una password debole o già presente in un dizionario. Mostra invece che il costo computazionale imposto a ogni tentativo modifica in modo decisivo la quantità di candidate analizzabili nello stesso intervallo di tempo.

I valori ottenuti descrivono la specifica configurazione hardware e software utilizzata nel laboratorio. Non devono essere interpretati come prestazioni universali, perché tempi e throughput possono cambiare in funzione del processore, del numero di thread, della versione di John the Ripper, dei backend disponibili e delle condizioni della singola esecuzione.

## Studio del costo di bcrypt

Il notebook comprende una sezione dedicata al comportamento del parametro di costo di bcrypt. Il valore di costo controlla il numero di iterazioni della funzione e segue una progressione esponenziale: un incremento unitario raddoppia approssimativamente il lavoro richiesto.

Lo studio permette di confrontare i costi 8, 10, 12 e 14 e di osservare la crescita del tempo necessario per calcolare una singola stringa bcrypt. Questa parte dell’analisi è distinta dall’attacco principale, che utilizza esclusivamente bcrypt con costo 12.

Il costo deve essere scelto cercando un equilibrio tra sicurezza e usabilità. Un valore più elevato rallenta gli attacchi offline, ma aumenta anche il tempo e le risorse richiesti al sistema legittimo durante autenticazione e registrazione.

## Prova supplementare sulla GPU

Il repository documenta anche una prova separata sulla **NVIDIA GeForce RTX 5070 Ti Laptop GPU**, eseguita tramite il backend OpenCL di John the Ripper. Questa sezione serve a osservare la capacità di parallelizzazione dell’architettura GPU sui formati adatti a un numero molto elevato di operazioni indipendenti.

Il benchmark Raw-MD5 mostra l’elevato throughput raggiungibile dalla GPU in un carico sufficientemente esteso. L’attacco con una wordlist di sole 20.000 candidate è però troppo breve per sfruttare pienamente tale capacità: inizializzazione di OpenCL, preparazione del kernel, allocazione dei buffer, trasferimento dei dati e sincronizzazione finale possono incidere più del calcolo vero e proprio.

Per questo motivo i risultati del benchmark e quelli dell’attacco end-to-end vengono mantenuti separati. Un throughput teorico molto elevato non implica automaticamente un vantaggio proporzionale in ogni prova reale, soprattutto quando il carico è ridotto.

Anche il benchmark bcrypt viene interpretato separatamente dall’attacco principale quando utilizza un costo diverso. I risultati ottenuti con un determinato costo non vengono trasferiti direttamente a bcrypt con costo 12.

## Struttura del repository

```text
Password_Offline_Lab/
│
├── README.md
├── .gitignore
│
├── assets/
│   └── banner-password-offline-lab.png
│
├── hashes/
│   ├── hash_md5.txt
│   └── hash_bcrypt.txt
│
├── wordlists/
│   └── wordlist_controllata.txt
│
├── figures/
│   └── grafici generati dal notebook
│
└── notebooks/
    └── Notebook Password_Offline_Lab.ipynb
```

Il notebook rappresenta il nucleo del progetto. Durante la configurazione locale vengono create anche le cartelle `JtR`, `outputs`, `results`, `data` e `report`. Questi contenuti non sono pubblicati nella repository perché comprendono il programma esterno, file temporanei, sessioni di John the Ripper e risultati rigenerabili.

## Dashboard di esecuzione

All’inizio del notebook è presente una dashboard centralizzata che permette di scegliere quali sezioni eseguire. I flag controllano lo studio del costo di bcrypt, i benchmark CPU, la prova di scalabilità, il benchmark GPU e gli attacchi GPU opzionali.

Le celle successive leggono queste impostazioni senza ridefinirle. In questo modo l’intera esecuzione può essere configurata da un unico punto, evitando modifiche sparse nel notebook.

## Configurazione completa dell’ambiente

Le istruzioni seguenti sono riferite a **Windows 10 o Windows 11 con PowerShell**. L’esperimento principale viene eseguito su CPU e non richiede una GPU specifica. Le prove GPU sono supplementari e possono essere disattivate dalla dashboard del notebook.

### 1. Prerequisiti

Prima di iniziare occorrono:

- **Git**, necessario per clonare la repository;
- **Python 3.10 a 64 bit**, con il comando `python` disponibile nel terminale;
- **John the Ripper Jumbo per Windows**, necessario per i formati Raw-MD5 e bcrypt;
- **driver GPU aggiornati e supporto OpenCL**, richiesti soltanto per le prove GPU.

Verificare Git e Python con:

```powershell
git --version
python --version
```

La versione usata per lo sviluppo è Python 3.10.11. Versioni successive possono funzionare, ma Python 3.10 è la configurazione di riferimento sulla quale è stato eseguito il laboratorio.

### 2. Clonazione della repository

Aprire PowerShell nella cartella in cui si desidera conservare il progetto ed eseguire:

```powershell
git clone https://github.com/xSkidrow/Attacco_offline_controllato_su_Raw-MD5_e_bcrypt.git
Set-Location .\Attacco_offline_controllato_su_Raw-MD5_e_bcrypt
```

Tutti i comandi successivi devono essere eseguiti dalla cartella principale della repository.

### 3. Creazione dell’ambiente virtuale Python

Creare e attivare un ambiente isolato:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Quando l’ambiente è attivo, PowerShell mostra `(.venv)` all’inizio della riga. Se l’esecuzione dello script di attivazione viene bloccata, autorizzarla soltanto per la sessione corrente e riprovare:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

### 4. Installazione delle dipendenze Python

Aggiornare `pip` e installare Jupyter insieme alle librerie impiegate dal notebook:

```powershell
python -m pip install --upgrade pip
python -m pip install jupyter ipykernel bcrypt pandas numpy matplotlib psutil
```

Registrare quindi l’ambiente come kernel selezionabile in Jupyter:

```powershell
python -m ipykernel install --user --name password-offline-lab --display-name "Python (Password Offline Lab)"
```

### 5. Installazione di John the Ripper Jumbo

Scaricare una build **Jumbo** di John the Ripper per Windows ed estrarla nella cartella locale `JtR`, mantenendo la directory `run` fornita dal programma. Il percorso atteso è:

```text
Password_Offline_Lab/
└── JtR/
    └── run/
        └── john.exe
```

Se la clonazione ha creato una cartella con il nome della repository, in questo schema `Password_Offline_Lab` corrisponde semplicemente alla sua cartella principale. `JtR` è esclusa da Git perché John the Ripper è un programma esterno e deve essere installato localmente.

Verificare che l’eseguibile funzioni:

```powershell
.\JtR\run\john.exe --list=build-info
```

Verificare poi la disponibilità dei formati impiegati nel laboratorio:

```powershell
.\JtR\run\john.exe --list=formats | Select-String -Pattern "Raw-MD5|bcrypt"
```

Se `john.exe` si trova in una posizione differente, aggiornare nella dashboard iniziale del notebook la variabile che contiene il percorso dell’eseguibile.

### 6. Preparazione delle cartelle locali

Creare le directory usate per output, risultati e materiali locali:

```powershell
@("outputs", "results", "data", "report") | ForEach-Object {
    New-Item -ItemType Directory -Force -Path $_ | Out-Null
}
```

Queste cartelle possono rimanere vuote prima della prima esecuzione. I relativi file vengono prodotti o aggiornati dal notebook.

### 7. Controllo opzionale di OpenCL e della GPU

Questo passaggio serve soltanto per il benchmark e per gli attacchi GPU opzionali. Con i driver della scheda video già installati, verificare i dispositivi OpenCL riconosciuti da John the Ripper:

```powershell
.\JtR\run\john.exe --list=opencl-devices
```

Annotare l’indice del dispositivo da utilizzare e riportarlo nella dashboard del notebook. L’indice non è universale: su un altro computer la GPU potrebbe essere, per esempio, il dispositivo `0` anziché il dispositivo `1`.

Per verificare separatamente il backend Raw-MD5 OpenCL si può eseguire:

```powershell
.\JtR\run\john.exe --format=raw-MD5-opencl --test=10 --device=INDICE_GPU
```

Sostituire `INDICE_GPU` con il numero mostrato dal comando precedente. Se nessun dispositivo compatibile viene rilevato, lasciare disattivati i flag GPU: l’analisi principale su CPU rimane pienamente eseguibile.

### 8. Avvio del notebook

Con l’ambiente virtuale ancora attivo, aprire direttamente il notebook:

```powershell
jupyter notebook "notebooks/Notebook Password_Offline_Lab.ipynb"
```

In alternativa, per usare JupyterLab:

```powershell
jupyter lab "notebooks/Notebook Password_Offline_Lab.ipynb"
```

Selezionare il kernel **Python (Password Offline Lab)**. Prima di eseguire tutte le celle, controllare nella dashboard iniziale:

1. il percorso di `john.exe`;
2. i flag delle prove da eseguire;
3. l’indice OpenCL della GPU, se le sezioni GPU sono abilitate;
4. i percorsi delle cartelle di input e output.

Eseguire quindi le celle dall’alto verso il basso. I benchmark possono richiedere alcuni minuti; lo studio dei costi bcrypt più elevati e le prove GPU dipendono dalle prestazioni dell’hardware.

### 9. Verifica rapida della configurazione

Prima della prova completa è possibile controllare i componenti principali con:

```powershell
python -c "import bcrypt, pandas, numpy, matplotlib, psutil; print('Dipendenze Python: OK')"
.\JtR\run\john.exe --list=build-info
.\JtR\run\john.exe --list=formats | Select-String -Pattern "Raw-MD5|bcrypt"
```

Se questi comandi terminano senza errori, l’ambiente CPU è pronto. Il riconoscimento OpenCL è necessario soltanto quando vengono abilitate le sezioni GPU.

### Problemi comuni

| Problema | Soluzione |
| --- | --- |
| `python` o `git` non viene riconosciuto | Installare il programma mancante, riaprire PowerShell e controllare che sia presente nel `PATH`. |
| `Activate.ps1` è bloccato | Usare `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`, che modifica la regola soltanto per il terminale corrente. |
| `john.exe` non viene trovato | Controllare che esista `JtR\run\john.exe` oppure correggere il percorso nella dashboard. |
| Raw-MD5 o bcrypt non compaiono tra i formati | Utilizzare una build **Jumbo** completa di John the Ripper. |
| Nessun dispositivo OpenCL viene rilevato | Aggiornare i driver della GPU o disattivare le prove GPU. |
| Jupyter usa un interprete differente | Selezionare il kernel **Python (Password Offline Lab)** e riavviare il kernel. |
| Risultati numerici diversi | È normale: throughput e tempi dipendono da CPU, GPU, driver, versione del software e carico del sistema. |

## Riproducibilità e limiti

Il progetto è stato costruito per rendere esplicite le condizioni sperimentali. La password, gli hash, la wordlist e gli altri file di laboratorio sono generati e utilizzati localmente. Le misurazioni vengono registrate insieme alla configurazione della sessione, mentre tabelle e grafici derivano direttamente dai risultati raccolti dal notebook.

L’esperimento principale su CPU può essere riprodotto anche su sistemi dotati di una GPU diversa o privi di accelerazione OpenCL. Le prove supplementari su GPU richiedono invece un dispositivo compatibile con OpenCL, correttamente riconosciuto da John the Ripper Jumbo. L’indice del dispositivo deve essere adattato alla configurazione del sistema utilizzato oppure selezionato automaticamente dal notebook.

La procedura sperimentale rimane riproducibile su hardware differente, ma non è possibile attendersi gli stessi tempi e valori di throughput. Le prestazioni dipendono infatti dal processore, dal modello di GPU, dai driver, dal backend OpenCL, dalla versione del software e dalle condizioni della singola esecuzione.

La prova non rappresenta un attacco contro sistemi reali e non utilizza credenziali appartenenti a terzi. I risultati non devono essere generalizzati senza considerare l’hardware, il software, la dimensione della wordlist e i parametri degli algoritmi.

L’analisi non dimostra che una password lunga e casuale possa essere individuata mediante ricerca esaustiva in tempi realistici. Dimostra invece che, quando la password candidata è già presente nella wordlist, la funzione impiegata per proteggerla determina il costo di ogni verifica e, di conseguenza, la sostenibilità complessiva dell’attacco offline.

## Considerazioni conclusive

Il confronto mette in evidenza il limite fondamentale delle funzioni di hash veloci quando vengono utilizzate direttamente per memorizzare password. Raw-MD5 consente di calcolare un numero molto elevato di tentativi e non offre una protezione individuale tramite salt.

bcrypt affronta il problema introducendo salt casuale e key stretching. Il salt impedisce la precomputazione generalizzata e il riutilizzo diretto del lavoro tra hash differenti, mentre il costo rallenta deliberatamente ogni tentativo. La sicurezza complessiva dipende comunque anche dalla qualità della password, dalla corretta configurazione del sistema, dalla protezione del database e dall’impiego di ulteriori misure, come l’autenticazione multifattore.

## Uso responsabile

Il repository è stato realizzato esclusivamente per finalità didattiche e sperimentali. Tutte le password, le wordlist e gli hash impiegati sono sintetici e generati localmente. Gli strumenti e le procedure documentati devono essere utilizzati soltanto su dati propri o in ambienti per i quali si dispone di un’autorizzazione esplicita.
