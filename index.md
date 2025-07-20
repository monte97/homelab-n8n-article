---
title: "Deploy Self-Hosted di n8n in Homelab"
date: 2025-07-20T15:00:00+02:00
description: Automazione personale con n8n deploy, configurazione e integrazione in un ambiente casalingo
menu:
sidebar:
name:  Homelab n8n
identifier:  Homelab n8n
weight: 20
tags: ["n8n", "Automazione", "Homelab", "DevOps", "Self-Hosted"]
categories: ["Tecnologie", "Automazione"]
---


##  🔍 Contesto e Motivazioni 🔍


n8n è un workflow automation tool open-source, pensato per collegare tra loro servizi differenti tramite nodi configurabili. A differenza di alternative cloud-based come Zapier o Make, n8n permette il **self-hosting** completo, offrendo pieno controllo su dati, estensibilità e privacy.  
Maggiori informazioni sono disponibili nel [sito ufficiale](https://n8n.io/) o nella relativa [documentazione](https://docs.n8n.io/).

L’obiettivo di questo progetto è realizzare un’istanza di n8n **self-hosted** e **automatizzata** all’interno di un ambiente casalingo (homelab), usando container **LXC**, provisioning con **OpenTofu** (fork open-source di Terraform) e configurazione idempotente tramite **Ansible**.

Il repository pubblico è disponibile qui:  
👉 [https://github.com/monte97/homelab-n8n](https://github.com/monte97/homelab-n8n)

---

## 🏗️ Stack Infrastrutturale 🏗️

### Scelte architetturali

L’infrastruttura è definita attraverso tre livelli:

- **Provisioning**: l'infrastuttura su cui verrà installato il servizio è gestita tramite [OpenTofu](https://opentofu.org/)
- **Configurazione**: l'impostazione e la configurazione avviene meniante [Ansible](https://www.ansible.com/)
- **Deploy**: l'applicazione `n8n` viene infine distribuite tramite `docker-compose`, oanch'esso orchestrato da Ansible.

Questo approccio a livelli permette una chiara separazione delle responsabilità e un ambiente completamente riproducibile.

La scelta di LXC (Linux Containers) come nostra tecnologia di containerizzazione è stata deliberata, motivata dalla sua leggerezza e dalla maggiore trasparenza rispetto a una macchina virtuale (VM) tradizionale. A differenza delle VM, che emulano un intero stack hardware per ogni sistema operativo guest, LXC opera direttamente sul kernel del sistema host, rendendolo molto più efficiente.

#### Focus: Container LXC e confronto con Docker

I Linux Containers (LXC) rappresentano un approccio fondamentalmente diverso alla containerizzazione rispetto a Docker. Mentre Docker ha rivoluzionato il deployment delle applicazioni con il paradigma dell'application container, LXC mantiene la filosofia del **system container**: 
> un ambiente che simula un sistema operativo completo, condividendo il kernel dell'host.

I container LXC, infatti, sono progettati per comportarsi come sistemi Linux completi e indipendenti. Al loro interno troviamo:

- Un sistema di init (systemd o SysV) che gestisce il ciclo di vita dei servizi
- Capacità di eseguire servizi multipli simultaneamente
- Gestione utenti tradizionale con login e sessioni
- Ambiente persistente e stateful per natura

Docker, al contrario, adotta una filosofia molto diversa, focalizzandosi sull'esecuzione di un unico carico di lavoro. Questo si nota in particolar modo quando prendiamo in considerazione le sue caratteristiche principali:

- Un processo principale per container (idealmente)
- Architettura stateless e immutable
- Deployment attraverso layers filesystem
- Paradigma "cattle, not pets" - i container sono sostituibili

Vale la pena evidenziare come entrambe le tecnologie adottino i medesimi meccanismi per garantire isoalmento e accesso controllato alle risorse: [Namespace e CGroup.](https://montelli.dev/posts/docker-internals/)

### Topologia logica

L'istanza n8n è isolata in un container LXC con rete in modalità bridge. Questa configurazione collega il container alla rete fisica dell'host tramite un bridge virtuale, assegnandogli un indirizzo IP unico sulla rete locale. Sebbene il container sia pienamente accessibile dalla LAN, non è direttamente esposto su Internet, migliorando la sicurezza.

Tutte le configurazioni, dalla definizione del container all'impostazione dell'applicazione, sono versionate su Git e applicate in modo idempotente tramite Ansible. Ciò significa che i playbook possono essere eseguiti più volte ottenendo sempre lo stesso risultato. Questa pratica assicura che l'intero servizio possa essere ricreato in modo affidabile e completo su qualunque nodo compatibile, con un intervento manuale minimo.

> Todo: inserire immagine per rappresentare la configurazione di rete
---

## 🛠️ Provisioning con OpenTofu 🛠️

### Introduzione ad OpenTofu

OpenTofu è uno strumento open-source per la gestione dell'infrastruttura attraverso codice dichiarativo. Nato come fork della versione open-source di Terraform dopo il cambio di licenza di quest'ultimo, OpenTofu mantiene la stessa sintassi e filosofia, garantendo continuità e trasparenza nella gestione infrastrutturale.

Tradizionalmente, il provisioning dell'infrastruttura avviene attraverso interfacce web, script manuali o procedure documentate che devono essere eseguite passo dopo passo. Questo approccio presenta diversi problemi:

- **Inconsistenza**: Ogni deployment può differire leggermente dal precedente
- **Mancanza di tracciabilità**: Difficile sapere chi ha fatto cosa e quando
- **Scalabilità limitata**: Creare 10 server richiede 10 volte il tempo di crearne uno
- **Disaster recovery complesso**: Ricreare un ambiente richiede tempo e conoscenza tacita

OpenTofu rivoluziona questo processo permettendo di descrivere lo stato desiderato dell'infrastruttura attraverso file di configurazione. Invece di specificare come creare le risorse, si definisce cosa si vuole ottenere:

```hcl
# Esempio: definisco COSA voglio, non COME crearlo
resource "lxc_container" "n8n_prod" {
  name     = "n8n-production"
  image    = "ubuntu/22.04"
  memory   = "2048MB"
  cpu      = 2
  
  network {
    name = "lxc-bridge"
    ip   = "10.0.0.100"
  }
}
```

### Codifica Infrastruttura

Analizziamo le parti salienti del nostro [`main.tf](https://github.com/monte97/homelab-n8n/blob/master/terraform/main.tf)` per comprendere come OpenTofu traduce le nostre intenzioni in infrastruttura concreta.

#### Configurazione provider

```hcl
terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
      version = "~> 2.9"
    }
  }
  required_version = ">= 1.0"
}
```

Questa sezione definisce i **requisiti fondamentali** del nostro progetto. Specifichiamo che utilizzeremo il provider Proxmox nella versione 2.9.x (il simbolo `~>` indica compatibilità semantica) e richiediamo OpenTofu/Terraform versione 1.0 o superiore. Questa pratica garantisce **riproducibilità** e **stabilità** negli ambienti di team.


#### Autenticazione Sicura

```hcl
provider "proxmox" {
  pm_api_url = var.proxmox_api_url
  pm_api_token_id = var.proxmox_api_token_id
  pm_api_token_secret = var.proxmox_api_token_secret
  pm_tls_insecure = var.proxmox_tls_insecure
}
```

La configurazione del provider utilizza **variabili** anziché credenziali hard-coded. Questo pattern permette di mantenere i segreti separati dal codice, supportando diversi ambienti (dev, staging, prod) con la stessa configurazione base ma credenziali differenti.

#### Definizione Dichiarativa della Risorsa

Il cuore della configurazione è la definizione del container LXC:

```hcl
resource "proxmox_lxc" "n8n_container" {
  target_node = var.proxmox_node
  hostname = var.vm_name
  description = "n8n Workflow Automation Container"
  
  # Configurazione risorse computazionali
  cores = var.vm_cores
  memory = var.vm_memory
  swap = var.vm_swap
}
```

Questa sezione dimostra la **natura dichiarativa** di OpenTofu: non stiamo scrivendo uno script che dice "crea un container, poi assegna memoria, poi configura CPU", ma stiamo definendo lo **stato finale desiderato**. OpenTofu si occuperà di orchestrare le chiamate API necessarie per raggiungere questo stato.

#### Gestione Storage Multi-Layer

```hcl
# Root filesystem
rootfs {
  storage = var.vm_storage
  size = var.vm_disk_size
}

# Storage dedicato per i dati applicativi
mountpoint {
  key = "0"
  slot = 0
  storage = var.vm_storage
  mp = "/opt/n8n"
  size = var.vm_data_disk_size
}
```

La configurazione storage mostra un pattern avanzato: separazione tra **filesystem di sistema** e **dati applicativi**. Questo approccio facilita backup selettivi, migrazione dati e ridimensionamento indipendente degli storage.

#### Networking Deterministico

```hcl
network {
  name = "eth0"
  bridge = var.vm_network_bridge
  ip = var.vm_ip_address
  gw = var.vm_gateway
  type = "veth"
}
```

La configurazione di rete elimina l'assegnazione casuale di indirizzi IP. Ogni risorsa ha un **indirizzo prevedibile**, essenziale per automazioni, monitoraggio e integrazione con sistemi esterni.

#### Provisioning Post-Creazione

```hcl
provisioner "remote-exec" {
  inline = [
    "apt-get update",
    "apt-get install -y curl wget gnupg python3",
    "systemctl enable ssh"
  ]
  connection {
    type = "ssh"
    host = split("/", var.vm_ip_address)[0]
    private_key = file(var.ssh_private_key)
  }
}
```

I **provisioner** permettono di eseguire configurazioni post-creazione. In questo caso, prepariamo il sistema con i pacchetti base e abilitiamo SSH. Questo bridge tra infrastruttura e configurazione applicativa è fondamentale per un deployment completo.

#### Lifecycle Management Intelligente

```hcl
lifecycle {
  ignore_changes = [
    ostemplate,  # Evita ricreazione del container per cambi template
  ]
}
```

Le regole di lifecycle prevengono **ricreazioni indesiderate**. Se il template del container viene aggiornato nel sistema Proxmox, OpenTofu non tenterà di ricreare il container esistente, preservando dati e configurazioni applicative.

---

## 📦 Automazione con Ansible 📦
 
### Introduzione ad Ansible

Ansible è una piattaforma di automation che gestisce la configurazione, il deployment e l'orchestrazione di sistemi attraverso un approccio dichiarativo e agentless. Sviluppato da Red Hat, rappresenta uno degli strumenti più diffusi nell'ecosistema DevOps per la gestione di infrastrutture complesse.

Una volta che l'infrastruttura è stata provisionata, rimane il compito più complesso: configurarla. Tradizionalmente, questo processo richiede:

- Login manuale sui server per installare software e modificare configurazioni
- Script bash personalizzati che spesso diventano ingestibili nel tempo
- Procedure documentate che devono essere seguite passo dopo passo
- Configurazioni che derivano nel tempo senza controllo delle modifiche
- Mancanza di idempotenza: ripetere la stessa operazione può produrre risultati diversi

Ansible rivoluziona questo processo con due caratteristiche fondamentali:
- **Dichiarativo**: Invece di scrivere script che descrivono come fare qualcosa, si dichiara quale stato finale si vuole raggiungere. Ansible si occupa di determinare le azioni necessarie per raggiungerlo.
- **Agentless**: A differenza di altri strumenti di configuration management, Ansible non richiede l'installazione di agent sui sistemi target. Utilizza SSH per Linux e WinRM per Windows, sfruttando protocolli già presenti nei sistemi.

```yaml
# Esempio: dichiaro LO STATO DESIDERATO
- name: Ensure Docker is installed and running
  package:
    name: docker.io
    state: present
  
- name: Ensure Docker service is enabled
  service:
    name: docker
    state: started
    enabled: yes
```
La forza di Ansible sta nella sua capacità di unificare la gestione di tutti questi livelli utilizzando un unico linguaggio e una metodologia coerente. Non importa se si sta configurando il sistema host o l'interno di un container LXC: gli stessi pattern, la stessa sintassi, la stessa filosofia operativa.

Questa uniformità è cruciale in un ambiente production dove la complessità deve essere gestita attraverso strumenti che riducano, non aumentino, il carico cognitivo degli operatori. Ansible trasforma la gestione della configurazione da attività manuale e frammentata a processo automatizzato, versionato e completamente riproducibile.

### Analisi del Playbook di Configurazione

Esaminiamo le sezioni più significative del nostro playbook Ansible per comprendere come trasformare un container vuoto in un'applicazione production-ready.

#### Struttura e Variabili Centralizzate

```yaml
vars:
  n8n_data_dir: "/opt/n8n_data"
  n8n_port: 5678
  n8n_domain: "n8n.K8S2.homelab"
  n8n_timezone: "Europe/Rome"
  n8n_docker_image: "docker.n8n.io/n8nio/n8n"
```

La **centralizzazione delle variabili** rappresenta una best practice fondamentale. Tutte le configurazioni specifiche dell'ambiente sono definite in un unico punto, permettendo di adattare lo stesso playbook a diversi contesti (development, staging, production) semplicemente modificando questi valori. Questo pattern elimina la necessità di cercare e sostituire valori hardcoded sparsi nel codice.

#### Gestione Condizionale del Sistema Operativo

```yaml
- name: Install required system packages
  ansible.builtin.package:
    name:
      - ca-certificates
      - curl
      - gnupg
      - python3-pip
    state: present
  when: ansible_os_family == "Debian"
```

Ansible utilizza **facts automatici** per rilevare le caratteristiche del sistema target. La condizione `when: ansible_os_family == "Debian"` rende il playbook **multi-platform compatible**. Questo approccio permette di gestire con lo stesso codice diverse distribuzioni Linux, adattando automaticamente i comandi alle specificità di ciascuna.

#### Installazione Docker con Controllo Dettagliato

```yaml
- name: Add Docker GPG key
  ansible.builtin.apt_key:
    url: "https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg"
    state: present

- name: Add Docker APT repository
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/{{ ansible_distribution | lower }} {{ ansible_distribution_release }} stable"
    state: present
```

Anziché installare Docker dai repository di default (spesso obsoleti), il playbook configura i **repository ufficiali Docker**. L'uso delle variabili `{{ ansible_distribution }}` e `{{ ansible_distribution_release }}` garantisce che venga utilizzato il repository corretto per la specifica versione del sistema operativo, eliminando errori di compatibilità.

#### Gestione Privilegi e Sicurezza

```yaml
- name: Add user to docker group
  ansible.builtin.user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Install compatible Docker Python packages
  ansible.builtin.pip:
    name:
      - docker==6.1.3
      - docker-compose==1.29.2
    state: present
```

La gestione dei privilegi segue il **principio del minimo privilegio**. L'utente viene aggiunto al gruppo `docker` per evitare di dover utilizzare `sudo` per ogni comando Docker. L'installazione di versioni specifiche dei pacchetti Python (`docker==6.1.3`) garantisce **compatibilità e riproducibilità**, evitando problemi di breaking changes in versioni future.

#### Cleanup Idempotente

```yaml
- name: Remove old n8n containers and data (cleanup)
  ansible.builtin.shell: |
    docker compose down || true
    docker container rm -f n8n || true
    docker volume rm n8n_data || true
  args:
    chdir: "{{ n8n_data_dir }}"
  ignore_errors: yes
```

Questa sezione implementa un **cleanup robusto** prima della configurazione. L'uso di `|| true` e `ignore_errors: yes` rende l'operazione **idempotente**: può essere eseguita multiple volte senza generare errori, anche se alcuni elementi da rimuovere non esistono. Questo pattern è essenziale per playbook che devono essere ri-eseguibili.

#### Generazione Dinamica della Configurazione

```yaml
- name: Create Docker Compose file for n8n
  ansible.builtin.copy:
    content: |
      version: '3.8'
      services:
        n8n:
          image: {{ n8n_docker_image }}:{{ n8n_docker_tag }}
          environment:
            - N8N_HOST={{ n8n_domain }}
            - WEBHOOK_URL=http://{{ n8n_domain }}:{{ n8n_port }}
            - GENERIC_TIMEZONE={{ n8n_timezone }}
          volumes:
            - n8n_data:/home/node/.n8n
    dest: "{{ n8n_data_dir }}/docker-compose.yml"
  notify: Restart n8n container
```

Il playbook **genera dinamicamente** il file Docker Compose utilizzando le variabili definite in precedenza. Questo approccio elimina la necessità di mantenere template separati e garantisce che ogni deployment sia configurato correttamente per il proprio ambiente specifico.

#### Sistema di Handler per Reazioni Automatiche

```yaml
handlers:
  - name: Restart n8n container
    ansible.builtin.command:
      cmd: docker compose restart
      chdir: "{{ n8n_data_dir }}"
    listen: "Restart n8n container"
```

Gli **handler** implementano un sistema di reazioni automatiche: quando il task di creazione del Docker Compose file viene modificato (`notify: Restart n8n container`), Ansible esegue automaticamente il restart del container. Questo meccanismo garantisce che le modifiche di configurazione vengano applicate immediatamente senza intervento manuale.

#### Deployment Idempotente

```yaml
- name: Start n8n container with Docker Compose
  ansible.builtin.command:
    cmd: docker compose up -d
    chdir: "{{ n8n_data_dir }}"
  register: docker_compose_result
  changed_when: "'Creating' in docker_compose_result.stdout or 'Starting' in docker_compose_result.stdout"
```

Il task finale implementa **deployment intelligente**: Ansible registra l'output del comando e considera il task "cambiato" solo se effettivamente vengono creati o avviati nuovi container. Questo permette di distinguere tra esecuzioni che modificano lo stato del sistema e quelle che lo trovano già nell'stato desiderato.

Questa uniformità è cruciale in un ambiente production dove la complessità deve essere gestita attraverso strumenti che riducano, non aumentino, il carico cognitivo degli operatori. Ansible trasforma la gestione della configurazione da attività manuale e frammentata a processo automatizzato, versionato e completamente riproducibile.


---

## 🤖 n8n: deploy e configurazione 🤖

Il deployment di n8n rappresenta l'ultimo livello del nostro stack, dove la semplicità operativa incontra la potenza dell'automazione. Ho scelto di utilizzare Docker Compose per orchestrare il container applicativo, mantenendo coerenza con l'approccio dichiarativo dell'intera infrastruttura.

### Filosofia di Deployment

La configurazione di n8n segue i principi di production-readiness e operabilità, evitando configurazioni complesse quando non necessarie. Il focus è su:

- **Configurazione essenziale** attraverso variabili d'ambiente minime
- **Persistenza dei dati** attraverso volumi gestiti
- **Affidabilità** con restart automatico
- **Semplicità** operativa senza over-engineering

```yaml
version: '3.8'
services:
    n8n:
        image: {{ n8n_docker_image }}:{{ n8n_docker_tag }}
        container_name: n8n
        restart: unless-stopped
        ports:
        - "{{ n8n_port }}:5678"
        environment:
        - N8N_SECURE_COOKIE=false
        - N8N_HOST={{ n8n_domain }}
        - N8N_PORT=5678
        - N8N_PROTOCOL=http
        - WEBHOOK_URL=http://{{ n8n_domain }}:{{ n8n_port }}
        - GENERIC_TIMEZONE={{ n8n_timezone }}
        - TZ={{ n8n_timezone }}
        - N8N_LOG_LEVEL=info
        - N8N_DIAGNOSTICS_ENABLED=false
        
        volumes:
        - n8n_data:/home/node/.n8n
        
        # Rimuovere il comando personalizzato - lasciare che il container usi il suo entrypoint di default
        
        # Rimuovere health check per ora
        # healthcheck:
        #   test: ["CMD-SHELL", "curl -f http://localhost:5678/healthz || exit 1"]
        #   interval: 30s
        #   timeout: 10s
        #   retries: 5
        #   start_period: 30s
        
    volumes:
    n8n_data:
        external: false
```

### Scelte di configurazione

- **Configurazione Minimale**: Ho optato per un approccio "minimal viable configuration" che include solo le variabili d'ambiente strettamente necessarie. Questo riduce la complessità operativa e minimizza i punti di failure, sfruttando i default sensati dell'immagine n8n ufficiale.

- **Named Volumes per la Persistenza**
L'utilizzo di un named volume Docker (n8n_data) garantisce la persistenza di tutti i dati critici: database SQLite interno, workflow, credenziali e configurazioni. Docker gestisce automaticamente la location fisica, garantendo portabilità e semplificando backup e migrazione.

- **Restart Policy**: La policy unless-stopped assicura che il container si riavvii automaticamente in caso di crash o riboot del sistema, mantenendo alta disponibilità senza interventi manuali.

- **Variabili Template**: Le variabili Ansible (`{{ n8n_domain }}`, `{{ n8n_port }}`, etc.) permettono di personalizzare la configurazione per diversi ambienti (development, staging, production) utilizzando lo stesso template base.

---
<!-- 
## ⚡ RUN ⚡

---

## 📚 Risorse Utili 📚

--- -->