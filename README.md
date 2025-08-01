# Guida Tecnica per l'Installazione del Modello ICON su Cluster HPC CINECA G100

## Indice

1. [Prerequisiti e Preparazione Ambiente](#1-prerequisiti-e-preparazione-ambiente)
2. [Gestione Dipendenze e Librerie](#2-gestione-dipendenze-e-librerie)
3. [Setup Repository e Codice Sorgente](#3-setup-repository-e-codice-sorgente)
4. [Configurazione Ambiente di Build](#4-configurazione-ambiente-di-build)
5. [Compilazione ICONTOOLS](#5-compilazione-icontools)
6. [Compilazione ICON Principale](#6-compilazione-icon-principale)
7. [Configurazione Post-Installazione](#7-configurazione-post-installazione)
8. [Troubleshooting e Manutenzione](#8-troubleshooting-e-manutenzione)
9. [Casi Speciali e Configurazioni Avanzate](#9-casi-speciali-e-configurazioni-avanzate)
10. [Appendici Tecniche](#10-appendici-tecniche)

---

## 1. Prerequisiti e Preparazione Ambiente

### 1.1 Requisiti Hardware e Software

Il modello ICON (ICOsahedral Nonhydrostatic) è un sistema di modellazione meteorologica e climatica che richiede risorse computazionali significative. Per l'installazione su cluster HPC CINECA G100 sono necessari i seguenti requisiti:

**Requisiti Hardware:**
- Architettura x86_64 con supporto AVX2
- Minimo 8 GB RAM per processo di compilazione
- Spazio disco: almeno 10 GB per codice sorgente e build
- Accesso a filesystem paralleli (GPFS/Lustre)

**Requisiti Software:**
- Sistema operativo Linux (CentOS/RHEL 7+ o equivalente)
- Compilatori Fortran 2008/2015 compatibili
- Librerie MPI (OpenMPI o Intel MPI)
- Build tools (make, cmake, autotools)
- Accesso a moduli software tramite Environment Modules

**Compilatori Supportati (aggiornato ad Aprile 2024):**
- GNU gcc: v11.2.0, v12.1.0, v12.3.0
- Intel ifort: v2021.5.0, v2021.6.0
- Cray ftn: v12.0.3, v16.0.1
- NAG nagfor: v7.1.7114, v7.1.7125
- NEC nfort: v5.0.1, v5.1.0
- NVIDIA nvfortran: v21.3, v23.3

### 1.2 Accesso al Cluster CINECA G100

Il cluster G100 utilizza un sistema di autenticazione basato su chiavi SSH e richiede configurazioni specifiche per l'accesso ai repository GitLab.

**Configurazione Accesso Base:**
```bash
# Connessione al cluster
$vssh username@login.g100.cineca.it

# Verifica ambiente
$ hostname

$ which module

$ echo $MODULEPATH
```

**Setup Ambiente Utente:**
```bash
# Creazione directory di lavoro
$ mkdir -p $WORK/username/my_icon
cd $WORK/username/my_icon

# Setup variabili ambiente base:
# "Definizione delle directory principali usate per organizzare i file di lavoro (ICON_WORK_DIR) e per la compilazione (ICON_BUILD_DIR). Si basano sulla variabile d’ambiente globale $WORK, comune nei sistemi HPC, e permettono di strutturare il workflow in modo modulare e pulito."
$ export ICON_WORK_DIR=$WORK/username/my_icon
$ export ICON_BUILD_DIR=$ICON_WORK_DIR/builds
```

### 1.3 Setup SSH e Chiavi di Accesso

From: [Icon for new developers](https://www.cosmo-model.org/content/support/icon/icon_for_nwp_developers.pdf)
Per accedere ai repository ICON su [GitLab DKRZ](https://gitlab.dkrz.de/), you have to register a SSH key in your
GitLab profile, per configurare correttamente le chiavi SSH.
The procedure is described in the [GitLab README](https://docs.gitlab.com/user/ssh/) and the key commands
are summarized in the following:

**Generazione Chiave SSH ED25519:**
```bash
# See if you have an existing SSH key par: Go to your home directory, go in to the .ssh/ subdirectory. If it doesn't exist you need to generate an SSH key par.
$ cd
$ .ssh/
 -bash: .ssh/: No such file or directory

# Generazione chiave con passphrase per sicurezza
# Generate a ED25519 ssh key. Follow the README. For security reasons your ssh keys should be secured with a passphrase. E.g.:
$ ssh-keygen -t ed25519 -f "$HOME/.ssh/id_ed25519_gitlab.dkrz.de"
Enter passphrase (empty for no passphrase): 

# Configurazione ssh-agent per gestione passphrase
$ eval "$(ssh-agent -s)"

#You can setup and configure ssh-agent on your machine in order to unlock your ssh passphrases just once per computer session
$ ssh-add ~/.ssh/id_ed25519_gitlab.dkrz.de
```

**Configurazione SSH Config:**
```bash
#It is possible to use different ssh keys for different services. This becomes handy for services with short expire limits. As a starting point, connect your ssh key and the gitlab.dkrz.de in your ~/.ssh/config as follows.
# Aggiunta configurazione a ~/.ssh/config
$ cat >> ~/.ssh/config << EOF

$ #GitLab DKRZ
$ Host gitlab.dkrz.de
$    PreferredAuthentications publickey
$    IdentityFile ~/.ssh/id_ed25519_gitlab.dkrz.de
$ EOF
```

**Registrazione Chiave su GitLab:**
```bash
# After creating the ssh key, you have to copy-paste the content of the new *.pub file to the SSH settings in GitLab.
# Visualizzazione chiave pubblica
$ cat "$HOME/.ssh/id_ed25519_gitlab.dkrz.de.pub"

# Copio la chiave precedente e la inserisco su Git: "Vai su GitLab → https://gitlab.dkrz.de/-/user_settings/ssh_keys; Incolla nella textarea chiamata Key: premi Add Key."
![Immagine_aggiunta_key](Icon_installation_on_CINECA_Immagine_1_0.png)

# Test connessione (se non ho fatto il passaggio precedente mi dirà "accesso negato")
$ssh -T git@gitlab.dkrz.de
Welcome to GitLab, @number_user!
```

### 1.4 Configurazione Profili Moduli Software

Il sistema G100 utilizza profili moduli per organizzare il software disponibile. Per ICON è necessario attivare profili specifici.

**Attivazione Profili Necessari:**
```bash
# Lista profili disponibili
$ modmap --all

# Caricamento profilo advanced (richiesto per ecCodes)
$ module load profile/advanced

# In alcuni casi può essere necessario profile/archive
$ module load profile/archive

# Verifica moduli disponibili
$ module avail
```

---

## 2. Gestione Dipendenze e Librerie (DA VERIFICARE)
From "ICON indofrm.docx", cap:"Compilazion Icon su Cineca" by eminguz
[ICON inform](https://docs.google.com/document/d/1EUSS5lHaqZmkv0PiUGr1d75ei0EuMAR_/edit)
### 2.1 Introduzione a SPACK (Build System)
**NOTA IMPORTANTE: SPACK è OPZIONALE per l'installazione di ICON su G100.**
La maggior parte degli utenti dovrebbe utilizzare i moduli precompilati del cluster (sezione 2.2) che sono
già ottimizzati e testati. SPACK è utile in casi specifici. 
**Quando usare SPACK:**
- Versioni specifiche di librerie non disponibili nei moduli di sistema
- Configurazioni personalizzate delle librerie
- Sviluppo di funzionalità che richiedono librerie sperimentali
- Installazione su sistemi senza moduli precompilati
**Quando NON usare SPACK (casi più comuni):**
- Prima installazione di ICON
- Uso di configurazioni standard
- Cluster con moduli ottimizzati disponibili (come G100)

SPACK è un "build system", un sistema di gestione pacchetti progettato per ambienti HPC che consente l'installazione e gestione di librerie scientifiche con controllo granulare delle dipendenze.

I pacchetti vengono scaricati dai repository remoti e installati in una dir specificata dall'utente.
In "opt" ci sono i pacchetti installati; se ripeto l'installazione, Spack riconosce i pacchetti già installati e compilati e, se la configurazione richiesta non è cambiata, non li ri-compila.
SPACK gestisce sia i singoli pacchetti che i moduli, permette di insallare e mantenere build diversi dello stesso pacchetto, permette di creare environment dedicati contenenti i pacchetti richiesti.

Sono presenti delle opzioni, le "variances", che possono essere abilitate volta per volta per personalizzare la configurazione dei pacchetti. Le sintassi '*','%','^' ... servono a specificare queste opzioni (e.g: @ specifica la versione, %specifica il compilatore, + aggiunge opzione,...). Per ulteriori specifiche, seguire la documentazione disponibile su [Da aggiungere documentazione](). Per i comandi vedi [ICON inform](https://docs.google.com/document/d/1EUSS5lHaqZmkv0PiUGr1d75ei0EuMAR_/edit)

**Caratteristiche Principali di SPACK:**
- Gestione automatica delle dipendenze
- Supporto per multiple versioni dello stesso pacchetto
- Controllo fine delle opzioni di compilazione
- Creazione di ambienti isolati (evnironment)
- Riutilizzo di build esistenti

**Installazione SPACK (solo se necessario):**
Ci sono 2 opzioni: installare la versione più aggiornata, installandolo in locale da github oppure caricare spack del sistema, che sarà meno aggiornato ma è già preconfigurato.

Opzione Installazione SPACK Locale:
```bash
# Clone repository SPACK
$git clone -c feature.manyFiles=true https://github.com/spack/spack.git
$cd spack

# Setup ambiente SPACK
$source share/spack/setup-env.sh

# Verifica installazione
$spack --version
1.1.0.dev0 (b821076ad78e9468d001393df7e5b36b70604fb6)

# Verifica compilatori disponibili
$spack compilers
No compilers available. Run `spack compiler find` to autodetect compilers
# è giusto così perchè li devo prima caricare

```
Opzione Utilizzo SPACK di Sistema (NON TESTATO):
```bash
# Caricamento SPACK di sistema
$module load spack

# Verifica compilatori disponibili
$spack compilers
```

**Comandi SPACK Fondamentali(NON TESTATO):**
```bash
# Ricerca pacchetti
spack list hdf5
spack info hdf5

# Gestione ambienti
spack env create icon-env
spack env activate icon-env
```

### 2.2 Configurazione Moduli Software su G100

I moduli sono gruppi di istruzioni che modificano l’ambiente shell per rendere fruibile un pacchetto software. Assegnano o integrano un insieme di variabili d'ambiente, tra cui $PATH, $LD_LIBRARY_PATH, etc.
Dal 2021 i moduli sono raggruppati in “profili”: di default ne sono disponibili solo alcuni, per gli altri bisogna prima caricare il relativo profilo. Di default viene caricato solo il profilo “base”; per ICON serve (almeno) anche il profilo “advanced”: module load profile/advanced
Per visualizzare tutti i profili disponibili: modmap -all. Vedi anche: modmap --help

module avail package: elenco delle versioni disponibili per il modulo package. Il comando restituisce anche il path assoluto del “module file”.
module list: elenco dei moduli attualmente caricati; <aL>=auto-loaded, <L>=loaded
module load, module switch: carica o sostituisce un modulo
module clear: cancella le assegnazioni di tutti i moduli caricati


Il sistema G100 fornisce moduli precompilati ottimizzati per l'architettura del cluster. La corretta selezione e compatibilità dei moduli è fondamentale per il successo della compilazione: i diversi moduli usati devono essere stati creati con lo stesso compilator

**Principi di Compatibilità Moduli:**
- Tutti i moduli usati devono essere stati compilati con lo stesso compilatore
- Non è possibile mescolare moduli scalari e paralleli (MPI)
- Le versioni delle librerie devono essere coerenti tra loro

L'ultimo principio è importante perché alcune librerie sono state compilate anche in locale da SIMC (con il compilatore intel) e possono essere chiamate senza usare moduli. Si trovano in /g100_work/smr_prod/srcintel, documentazione in [Documentazione_SIMC_Compilazione_grib_api_etc](https://docs.google.com/document/d/1L9oTAeGiYE_Yqp4A5XAYwLj7XjaPWZ0PrytsPG_VDQw)
Devo prima caricare i moduli richiesti e esportare le variabili che puntino ai compilatori caricati.


Per verificare i module available usa il comando (e.g. xxxx==intel) 
$ module avail 2>&1 | grep xxxx
**Set di Moduli Usati (2025/08/01):**
```bash
# Setup compilatori Intel OneAPI
$ module load intel/oneapi-2021--binary
$ module load intelmpi/oneapi-2021--binary

# Librerie I/O (al momento non MPI)
$ module load hdf5/1.10.7--oneapi--2021.2.0-ifort
$ module load netcdf-c/4.8.1--intel--2021.4.0
$ module load netcdf-fortran/4.5.3--intel--2021.4.0

# Compressione e codifica
$ module load szip/2.1.1--oneapi--2021.2.0-ifort
$ module load eccodes/2.21.0--intelmpi--oneapi-2021--binary
ERROR: Unable to locate a modulefile for 'eccodes/2.21.0--intelmpi--oneapi-2021--binary'
# Giusto perchè non ho MPI al momento
$ module load eccodes/2.21.0--intel--2021.4.0
# Librerie matematiche
$ module load mkl/oneapi-2021--binary

```



**Set di Moduli Alternativi (GNU Compiler)(NON TESTATO):**
```bash
# Setup compilatori GNU
module load gcc/10.2.0
module load openmpi/4.1.1--gcc--10.2.0

# Librerie I/O
module load hdf5/1.10.7--gcc--10.2.0
module load netcdf/4.7.4--gcc--10.2.0
module load netcdff/4.5.3--gcc--10.2.0

# Codifica GRIB
module load eccodes/2.21.0--gcc--10.2.0
```

### 2.3 Librerie Richieste
From: [ICON Tutorial 2024]()
Spiegazione dei moduli appena caricati.
ICON richiede diverse librerie specializzate per l'I/O scientifico e la gestione di formati dati meteorologici.

**NetCDF (Network Common Data Form):**
NetCDF è essenziale per l'I/O di dati scientifici multidimensionali e la definizione della topologia di griglia. Serve infatti per salvare i dati di output o leggere quelli di imput

Caratteristiche:
- Formato autodescrittivo con metadati completi
- Supporto per array multidimensionali
- Compatibilità con HDF5 per file di grandi dimensioni
- Interfacce C e Fortran

Inoltre, è importante che NetCDF lavori in parallelo. Se ogni processo scrivesse su un file separato o attendesse il proprio turno per scrivere (lettura/scrittura) seriale, sarebbe inefficiente e lento.

Il supporto MPI-IO permette a più processi contemporaneamente di leggere/scrivere sullo stesso file NetCDF in modo coordinato. Questo si chiama I/O parallelo.

(COME SCARICARLA? CHIEDO PRIMA CONFERMA A DAVIDE)

**HDF5 (Hierarchical Data Format 5)(DA SISTEMARE):**
Formato di file gerarchico per dati scientifici di grandi dimensioni.

Configurazione specifica:
- Deve supportare I/O parallelo per prestazioni ottimali
- Compatibilità con NetCDF-4
- Supporto per compressione dati

**ecCodes (ECMWF Coding/Decoding):**
Libreria ECMWF per la codifica/decodifica di formati GRIB1/2. Quindi serve a leggere/scrivere i file GRIB.

Se ecCodes supporta MPI-IO, più processi possono accedere contemporaneamente allo stesso file GRIB in modo coordinato e efficiente. Questo è importante quando i file GRIB sono grandi e devono essere letti/scritti da molti processi contemporaneamente: per esempio, nella fase di post-processing parallelo o durante l’output simultaneo da più processi.

Se il workflow di ICON usa GRIB principalmente in modo seriale o solo da pochi processi, ecCodes senza MPI potrebbe bastare.
Se invece è previsto un I/O parallelo intensivo su GRIB, conviene la versione MPI-aware. 
Per la maggior parte degli usi (come preprocessing GRIB per ICON), la versione seriale è sufficiente.

All'interno di icon è già presente ecCodes ma senza MIP. Per verificarlo:
```bash
# Cerco i moduli di ecCodes
$ grep -i eccodes
eccodes/2.21.0--gcc--10.2.0(default)         ncview/2.1.8--oneapi--2021.2.0-ifort                                     
eccodes/2.21.0--intel--2021.4.0              ninja/1.11.1 

# Carico il modulo della versione intel e verifico se è stato compilato con supporto MPI
$ module load eccodes/2.21.0--intel--2021.4.0
$ ldd $(which grib_ls) | grep mpi

# Se non trovo riferimenti a libmpi o libmpiifort, è seriale. (è non li trovo)

```

Se vuoi scaricare la versione MPI (NON CREDO CHE QUESTO QUA SOTTO BASTI!)
Installazione e configurazione:
```bash
# Download da ECMWF
wget https://confluence.ecmwf.int/download/attachments/45757960/eccodes-2.21.0-Source.tar.gz

# Configurazione build
./configure --prefix=/path/to/install \
            --disable-jpeg \
            --enable-shared=no \
            --enable-fortran

# Compilazione
make -j8
make install
```
