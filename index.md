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

## Contesto e Motivazioni

n8n è un potente strumento open source per l'automazione dei workflow, che permette di collegare una vasta gamma di servizi tramite nodi configurabili. A differenza di alternative SaaS (Software as a Service) come Zapier o Make, **n8n** può essere installato in modalità **self-hosted**, garantendo il pieno controllo su dati, privacy ed estensibilità.

L'obiettivo di questo progetto è illustrare come realizzare un'istanza di n8n completamente **automatizzata** all'interno di un ambiente casalingo (homelab). Verranno utilizzati:

  * **container LXC** per l'isolamento dell'ambiente
  * **OpenTofu** (un fork open source di Terraform) per il provisioning dell'infrastruttura
  * **Ansible** per la configurazione idempotente del servizio

L'intero codice del progetto è disponibile nel repository pubblico:  
👉 [https://github.com/monte97/homelab-n8n](https://github.com/monte97/homelab-n8n)

-----

## Stack Infrastrutturale

### Le scelte architetturali

L'infrastruttura è stata concepita su tre livelli distinti per una chiara separazione delle responsabilità e per garantire un ambiente completamente riproducibile:

1.  **Provisioning**: L'infrastruttura che ospiterà il servizio è gestita tramite [OpenTofu](https://opentofu.org/).
2.  **Configurazione**: L'impostazione del sistema operativo e l'installazione dei prerequisiti avvengono con [Ansible](https://www.ansible.com/).
3.  **Deploy**: L'applicazione **n8n** viene distribuita tramite `docker-compose`, anch'esso gestito da Ansible.

L'adozione di **LXC (Linux Containers)** come tecnologia di containerizzazione è una scelta ponderata. Motivata dalla sua leggerezza e dalla maggiore trasparenza rispetto a una macchina virtuale (VM) tradizionale, LXC opera direttamente sul kernel del sistema host, offrendo un'efficienza superiore.


### Focus: Container LXC vs. Docker

I Linux Containers (LXC) e Docker rappresentano due approcci alla containerizzazione con filosofie diverse. Mentre Docker ha rivoluzionato il deployment delle applicazioni con il paradigma dell'**application container**, LXC si basa sul concetto di **system container**: un ambiente che simula un sistema operativo completo pur condividendo il kernel dell'host.

| **Caratteristica** | **LXC (System Container)** | **Docker (Application Container)** |
| :---------------------------- | :------------------------------------------------------------- | :------------------------------------------------------------- |
| **Obiettivo** | Eseguire un sistema operativo completo, isolato e autonomo.    | Eseguire una singola applicazione o un singolo processo.        |
| **Struttura** | Include un sistema di init (es. `systemd`), più servizi, utenti e processi. | Generalmente ha un unico processo principale e un'architettura stateless. |
| **Persistenza** | Progettato per essere persistente e "stateful".                  | Progettato per essere immutabile e "stateless" (i dati persistenti sono gestiti tramite volumi esterni). |
| **Filosofia** | Si comporta come una VM leggera.                              | Paradigma "cattle, not pets".                               |
| **Meccanismi di isolamento** | Utilizza `Namespace` e `CGroup`.                                 | Utilizza `Namespace` e `CGroup`.                                 |


### Topologia logica

L'istanza di **n8n** è isolata in un container LXC con una rete in modalità **bridge**. Questa configurazione assegna al container un indirizzo IP unico sulla rete locale, rendendolo accessibile dalla LAN senza esporlo direttamente su Internet, migliorando la sicurezza.

Tutte le configurazioni, dalla definizione del container all'impostazione dell'applicazione, sono **versionate su Git** e applicate in modo **idempotente** tramite Ansible. Questo assicura che l'intero servizio possa essere ricreato in modo affidabile e completo su qualsiasi nodo compatibile, con un intervento manuale minimo.

-----

## Provisioning con OpenTofu

### Che cos'è OpenTofu?

OpenTofu è uno strumento open source per la gestione dell'infrastruttura attraverso codice dichiarativo. Nato come fork della versione open source di Terraform, OpenTofu mantiene la stessa sintassi e filosofia, offrendo continuità e trasparenza.

Tradizionalmente, il provisioning avviene tramite interfacce web, script manuali o procedure documentate, un approccio che può portare a:

  * **Inconsistenza**: ogni deployment può differire leggermente dal precedente.
  * **Mancanza di tracciabilità**: è difficile sapere chi ha fatto cosa e quando.
  * **Scalabilità limitata**: creare molti server richiede un tempo proporzionalmente maggiore.
  * **Disaster recovery complesso**: ricreare un ambiente da zero è spesso un processo lungo e manuale.

OpenTofu risolve questi problemi permettendo di descrivere lo stato desiderato dell'infrastruttura in file di configurazione, come nell'esempio seguente.

```hcl
# Esempio: si definisce COSA si vuole, non COME crearlo
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

### Analisi del codice (file `main.tf`)

Di seguito sono analizzate le parti salienti del file `main.tf` per comprendere come OpenTofu traduce le intenzioni in infrastruttura concreta.

#### Configurazione del provider

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

Questa sezione definisce i requisiti fondamentali del progetto. Si specifica l'uso del provider Proxmox (`version = "~> 2.9"`) e una versione minima di OpenTofu/Terraform (`>= 1.0`). Questa pratica garantisce **riproducibilità** e **stabilità** nell'ambiente di deployment.

#### Autenticazione sicura

```hcl
provider "proxmox" {
  pm_api_url = var.proxmox_api_url
  pm_api_token_id = var.proxmox_api_token_id
  pm_api_token_secret = var.proxmox_api_token_secret
  pm_tls_insecure = var.proxmox_tls_insecure
}
```

Il provider utilizza **variabili** anziché credenziali hard-coded. Questo approccio, una **best practice** in ambito Infrastructure as Code (IaC), mantiene i segreti separati dal codice e supporta diversi ambienti (dev, staging, prod) con la stessa configurazione base, ma credenziali differenti.

#### Definizione dichiarativa della risorsa

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

Questa sezione dimostra la **natura dichiarativa** di OpenTofu. Non si sta scrivendo uno script che dice "crea un container, poi assegna memoria...", ma si sta definendo lo **stato finale desiderato**. OpenTofu orchestra le chiamate API necessarie per raggiungere tale stato.

#### Gestione dello storage

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

La configurazione dello storage mostra un pattern avanzato: la separazione tra **filesystem di sistema** e **dati applicativi**. Questo facilita backup selettivi, migrazione dei dati e ridimensionamento indipendente dello storage.

#### Networking deterministico

```hcl
network {
  name = "eth0"
  bridge = var.vm_network_bridge
  ip = var.vm_ip_address
  gw = var.vm_gateway
  type = "veth"
}
```

La configurazione di rete elimina l'assegnazione casuale degli indirizzi IP. Ogni risorsa ha un **indirizzo prevedibile**, essenziale per l'automazione, il monitoraggio e l'integrazione con altri sistemi.

#### Provisioning post-creazione

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

I **provisioner** permettono di eseguire configurazioni post-creazione. In questo caso, il sistema viene preparato con i pacchetti base e viene abilitato SSH. Questo crea un ponte fondamentale tra la creazione dell'infrastruttura e la sua configurazione applicativa.

#### Lifecycle management

```hcl
lifecycle {
  ignore_changes = [
    ostemplate,  # Evita ricreazione del container per cambi template
  ]
}
```

Le regole di lifecycle prevengono **ricreazioni indesiderate**. Ad esempio, se il template del container viene aggiornato su Proxmox, OpenTofu non tenterà di ricreare il container esistente, preservando i dati e le configurazioni.

-----

## Automazione con Ansible

### Che cos'è Ansible?

Ansible è una piattaforma di automazione "agentless" per la configurazione, il deployment e l'orchestrazione dei sistemi. Sviluppata da Red Hat, è uno degli strumenti più diffusi in ambito DevOps per la gestione di infrastrutture complesse.

Dopo aver effettuato il provisioning dell'infrastruttura, il passo successivo è la configurazione, un processo che tradizionalmente richiedeva:

  * login manuale sui server per installare software.
  * script bash personalizzati, spesso difficili da gestire.
  * procedure manuali da seguire passo dopo passo.
  * mancanza di idempotenza: ripetere la stessa operazione poteva produrre risultati diversi.

Ansible risolve questi problemi grazie a due caratteristiche fondamentali:

  * **Dichiarativo**: invece di scrivere script che descrivono come fare qualcosa, si dichiara quale stato finale si vuole raggiungere. Ansible determina le azioni necessarie per ottenerlo.
  * **Agentless**: non richiede l'installazione di alcun software aggiuntivo sui sistemi target, utilizzando protocolli standard come SSH (per Linux) e WinRM (per Windows).

La sua forza sta nella capacità di unificare la gestione di tutti i livelli con un unico linguaggio e una metodologia coerente.

```yaml
# Esempio: si dichiara LO STATO DESIDERATO
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

### Analisi del Playbook di configurazione

Di seguito sono analizzate le sezioni più significative del playbook Ansible per comprendere come trasformare un container vuoto in un'applicazione pronta per la produzione.

#### Struttura e variabili centralizzate

```yaml
vars:
  n8n_data_dir: "/opt/n8n_data"
  n8n_port: 5678
  n8n_domain: "n8n.K8S2.homelab"
  n8n_timezone: "Europe/Rome"
  n8n_docker_image: "docker.n8n.io/n8nio/n8n"
```

La **centralizzazione delle variabili** è una best practice fondamentale. Tutte le configurazioni specifiche dell'ambiente sono definite in un unico punto, rendendo il playbook adattabile a diversi contesti (development, staging, production) semplicemente modificando questi valori.

#### Gestione condizionale del sistema operativo

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

Ansible utilizza i **facts automatici** per rilevare le caratteristiche del sistema target. La condizione `when: ansible_os_family == "Debian"` rende il playbook compatibile con diverse distribuzioni Linux, adattando automaticamente i comandi.

#### Installazione di Docker

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

Invece di installare Docker dai repository di default, spesso obsoleti, il playbook configura i repository ufficiali. L'uso delle variabili `{{ ansible_distribution }}` e `{{ ansible_distribution_release }}` garantisce che venga utilizzato il repository corretto per la specifica versione del sistema operativo.

#### Gestione dei privilegi

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

La gestione dei privilegi segue il **principio del minimo privilegio**. L'utente viene aggiunto al gruppo `docker` per evitare l'uso di `sudo` per ogni comando. L'installazione di versioni specifiche dei pacchetti Python (`docker==6.1.3`) garantisce **compatibilità e riproducibilità**.

#### Cleanup idempotente

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

Questa sezione implementa un **cleanup robusto** prima della configurazione. L'uso di `|| true` e `ignore_errors: yes` rende l'operazione **idempotente**: può essere eseguita più volte senza generare errori, anche se gli elementi da rimuovere non esistono.

#### Generazione dinamica della configurazione

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

Il playbook **genera dinamicamente** il file Docker Compose utilizzando le variabili definite in precedenza. Questo elimina la necessità di mantenere template separati e assicura che ogni deployment sia configurato correttamente per il proprio ambiente.

#### Handler per reazioni automatiche

```yaml
handlers:
  - name: Restart n8n container
    ansible.builtin.command:
      cmd: docker compose restart
      chdir: "{{ n8n_data_dir }}"
    listen: "Restart n8n container"
```

Gli **handler** sono un sistema di reazioni automatiche: quando il task di creazione del Docker Compose file viene modificato (`notify: Restart n8n container`), Ansible esegue automaticamente il riavvio del container. Questo garantisce che le modifiche di configurazione vengano applicate immediatamente.

#### Deployment idempotente

```yaml
- name: Start n8n container with Docker Compose
  ansible.builtin.command:
    cmd: docker compose up -d
    chdir: "{{ n8n_data_dir }}"
  register: docker_compose_result
  changed_when: "'Creating' in docker_compose_result.stdout or 'Starting' in docker_compose_result.stdout"
```

Il task finale implementa un **deployment intelligente**: Ansible considera il task "modificato" solo se effettivamente vengono creati o avviati nuovi container. Questo distingue tra esecuzioni che cambiano lo stato del sistema e quelle che lo trovano già nello stato desiderato.

-----

## n8n: deploy e configurazione

Il deployment di n8n rappresenta l'ultimo livello dello stack, dove la semplicità operativa incontra la potenza dell'automazione. L'utilizzo di Docker Compose per orchestrare il container applicativo mantiene la coerenza con l'approccio dichiarativo dell'intera infrastruttura.

### Filosofia di deployment

La configurazione di n8n segue i principi di "production-readiness" e operabilità, focalizzandosi su:

  * **Configurazione essenziale** tramite variabili d'ambiente.
  * **Persistenza dei dati** gestita da volumi.
  * **Affidabilità** con riavvio automatico.
  * **Semplicità** operativa, evitando l'over-engineering.

<!-- end list -->

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
        
volumes:
    n8n_data:
        external: false
```

### Le scelte di configurazione

  * **Configurazione minimale**: l'approccio "minimal viable configuration" include solo le variabili d'ambiente strettamente necessarie, riducendo la complessità operativa.
  * **Named Volumes**: l'uso di un volume Docker (`n8n_data`) garantisce la persistenza dei dati critici (database SQLite interno, workflow, credenziali e configurazioni), semplificando backup e migrazione.
  * **Restart Policy**: la policy `unless-stopped` assicura il riavvio automatico del container in caso di crash o riavvio del sistema, garantendo alta disponibilità.
  * **Variabili template**: l'uso di variabili Ansible (`{{ n8n_domain }}`, `{{ n8n_port }}`) permette di personalizzare la configurazione per diversi ambienti usando lo stesso template di base.

-----

## 📚 Risorse Utili

  * **n8n Official Documentation**: La risorsa principale per esplorare le funzionalità avanzate di n8n.
      * [n8n docs](https://docs.n8n.io/)
      * [n8n Community Forum](https://community.n8n.io/)
  * **OpenTofu Official Documentation**: Per comprendere a fondo la sintassi HCL e le capacità di provisioning.
      * [OpenTofu Docs](https://opentofu.org/docs/)
      * [Proxmox Provider for OpenTofu](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)
  * **Ansible Official Documentation**: Una guida completa per padroneggiare la configuration management.
      * [Ansible Documentation](https://docs.ansible.com/)
      * [Ansible Best Practices](https://docs.ansible.com/ansible/latest/playbooks_best_practices.html)
  * **Linux Containers (LXC)**: Per approfondire i container di sistema e le loro differenze con Docker.
      * [LXC Website](https://linuxcontainers.org/)
      * [LXD Documentation](https://linuxcontainers.org/lxd/docs/)
  * **Docker Documentation**: Indispensabile per comprendere i concetti base e avanzati dei container applicativi e Docker Compose.
      * [Docker Docs](https://docs.docker.com/)
      * [Docker Compose Overview](https://docs.docker.com/compose/)

-----