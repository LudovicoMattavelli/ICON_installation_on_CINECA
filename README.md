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
