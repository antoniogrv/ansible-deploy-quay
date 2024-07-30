# Self-hosted Quay senza accesso ad Internet

Questa repository descrive come installare e configurare il container registry [Quay](https://github.com/quay/quay) su una macchina virtuale senza l'accesso ad Internet. 

- [Installazione automatizzata](#installazione-automatizzata)
- [Installazione manuale](#installazione-manuale)
    - [Configurare HTTPS](#configurare-https)
    - [Problemi frequenti](#problemi-frequenti)
- [Esempio d'uso](#esempio-duso)
- [Preparazione di una release](#preparazione-di-una-release)

> [!WARNING]
> La macchina virtuale dovrebbe avere *almeno* 2 vCPU, 4 GB di RAM e 20+ GB di spazio libero su disco. Inoltre, le istruzioni presuppongono che il nodo monti una distribuzione vergine di [Ubuntu 22.04 Live Server](https://releases.ubuntu.com/jammy/ubuntu-22.04.4-live-server-amd64.iso).

La presente installazione ha l'obiettivo di distribuire Quay in "modalità di sviluppo" (i.e. non production-ready). Ciò significa che la configurazione che segue *non* si conforma ai requisiti tipici di un ambiente di produzione. Inoltre, si sottolinea che questo specifico branch di questa repository utilizza versioni particolari delle dipendenze di Quay. Utilizzare un branch diverso, se disponibile, in base alle proprie esigenze.

Per procedere, è possibile utilizzare una installazione automatizzata tramite playbook Ansible, oppure effettuare un'installazione manuale.

| Dipendenza        | Versione                          	| 
|----------------	|---------------------------------	    |
| `Quay`      	    | `master` 	                            |
| `docker-ce`      	| `20.10.24~3-0~ubuntu-jammy_amd64` 	|
| `docker-ce-cli`  	| `20.10.24~3-0~ubuntu-jammy_amd64` 	|
| `containerd.io`  	| `1.7.19-1_amd64`                  	|
| `docker-compose` 	| `2.29.0-linux-x86_64`             	|

## Installazione automatizzata

A partire da un'installazione vergine di Ubuntu 22.04 Live Server...

1. Accedere in SSH al nodo target. Creare un account `admin` nel gruppo `sudo` e impostare la sua password. 
```bash
sudo useradd -d -m admin
sudo passwd admin
sudo adduser admin sudo
```

Le successive istruzioni devono essere eseguite sulla propria macchina locale.

3. Modificare il file `/home/admin/.ssh/authorized_keys`, aggiungendo la propria chiave pubblica.

4. [Installare Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip). Oltre ai metodi indicati sulla documentazione, è possibile usare [Miniconda](https://docs.anaconda.com/miniconda/). 
L'obiettivo è ottenere la CLI `ansible-playbook` nel proprio `PATH`.
```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm -rf ~/miniconda3/miniconda.sh
~/miniconda3/bin/conda init bash

conda install -c conda-forge ansible
```

5. Clonare questa repository ed accedervi.
```bash
git clone git@github.com:antoniogrv/quay-offline-playbook.git
cd quay-offline-playbook
```

6. [Scaricare la dipendenze della propria release di riferimento](https://github.com/antoniogrv/quay-dev-playbook/releases), spostare i `.tar` nella root directory della repository appena clonata, ed eseguire il seguente script.
```bash
mv quay.tar deps/src
mv quay-local.tar quay-build.tar redis.tar postgres.tar clair.tar deps/images
tar -xvf tls.tar -C deps
tar -xvf docker.tar -C deps
rm tls.tar docker.tar
```

7. Modificare il file `inventory.yaml`, sostituendo a `x.y.z.k` l'indirizzo IP della macchina virtuale target.

8. Eseguire `ansible-playbook -i inventory.yaml --ask-become setup-quay-registry.yaml`. All'esecuzione, immettere la password dell'utente `admin`.

9. Il registry Quay sarà raggiungibile all'indirizzo IP indicato al punto 6. 

10. *Opzionale*. L'esecuzione del playbook restitiuisce localmente in output un file `outputs/rootCA.pem` con cui, volendo, è possibile mockare l'autenticazione della connessione HTTPS col registry. 

## Installazione manuale

1. Installare Docker sulla macchina virtuale target

    **a.** Scaricare i seguenti binari sulla macchina locale:
    - [docker-ce_20.10.24~3-0~ubuntu-jammy_amd64.deb](https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/docker-ce_20.10.24~3-0~ubuntu-jammy_amd64.deb)
    - [docker-ce-cli_20.10.24~3-0~ubuntu-jammy_amd64.deb](https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/docker-ce-cli_20.10.24~3-0~ubuntu-jammy_amd64.deb)
    - [containerd.io_1.7.19-1_amd64.deb](https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/containerd.io_1.7.19-1_amd64.deb)
    - [docker-compose-linux-x86_64](https://github.com/docker/compose/releases/download/v2.29.0/docker-compose-linux-x86_64)

    **b.** Trasferire i file sulla macchina target ed eseguire `sudo dpkg -i *.deb`

    **c.** Per installare Docker Compose, eseguire i seguenti comandi sulla macchina virtuale target:
    
    ```bash
    sudo mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```

2. Buildare le immagini e trasferire il codice sorgente

    **a.** Sulla macchina locale, eseguire `git clone https://github.com/quay/quay.git` ed accedere alla directory `quay`

    > **Attenzione.** Se si è su macchina locale Windows, accertarsi di eseguire i successivi comandi su terminale WSL, altrimenti le immagine create conterranno delle malformazioni che impediranno la corretta esecuzione dei container

    **a.** Quay e i suoi servizi accessori sono distribuiti mediante container Docker. L'esecuzione dei servizi è delegata ad un [Makefile](https://github.com/quay/quay/blob/master/Makefile), che a sua volta richiama un [docker-compose.yaml](https://github.com/quay/quay/blob/master/docker-compose.yaml)

    **b.** Sulla propria macchina locale, eseguire `make local-dev-up`. La directory di Quay verrà popolata con nuovi file e directory; inoltre, verranno costruite alcune immagini per container.

    **c.** Alla fine del processo, tramite `sudo docker images` dovrebbero risultare cinque immagini: PostgreSQL, Redis, Clair, `quay-build` e `quay-local`. Si sottolinea che Clair potrebbe non essere necessario, in base alle proprie esigenze applicative

    **d.** Comprimere in un file `.tar` l'intera directory di Quay (su Windows, è possibile usare 7zip)

    **e.** Trasferire l'archivio TAR di Quay sulla macchina virtuale target; successivamente, estrarne il contenuto (e.g. `mkdir quay; tar -xvf quay.tar -C quay`)

3. Salvare e caricare le immagini

    **a.** Sulla macchina locale, eseguire - per ognuna delle cinque immagini elencate in `sudo docker images` - il comando `sudo docker save [nome immagine]:[tag] -o [nome file]`, es. `sudo docker save <image id di quay-build> -o quay-build`

    **b.** Trasferire i file delle cinque immagini così salvate dalla macchina locale alla macchina virtuale target

    **c.** Dalla macchina virtuale target, per ognuna delle cinque immagini, eseguire `sudo docker load -i [nome file]`. Dopo ogni caricamento, verificare le immagini con `sudo docker images`: dovrebbero risultare senza nome né tag. Di conseguenza, prima di caricare la successiva immagine, sarà necessario taggare opportunamente l'immagine avendo cura di prelevare prima l'identificato della stessa. Le immagini dovranno essere così taggate:

    > **Attenzione.**Le versioni delle immagini indicate nei seguenti tag potrebbero non corrispondere a quelle effettivamente presenti sul `docker-compose.yaml` relativo alla release di Quay selezionata dal branch Git.

    -  `sudo docker tag <image id di quay-local> localhost/quay-local:latest`
    -  `sudo docker tag <image id di quay-build> localhost/quay-build:latest`
    -  `sudo docker tag <image id di clair> quay.io/projectquay/clair:4.7.2`
    -  `sudo docker tag <image id di redis> redis:latest`
    -  `sudo docker tag <image id di postgres> postgres:12.1`

4. Eseguire i servizi di Quay

    **a.** Sulla macchina virtuale target, eseguire i seguenti comandi:

    ```bash
    sudo DOCKER_USER="$(id -u):$(id -g)" docker-compose up -d local-dev-frontend
    sudo docker-compose up -d redis quay-db
    sudo DOCKER_USER="$(id -u):0" docker-compose up -d quay
    ```

    **b.** Sarà possibile accedere a Quay via `localhost:8080`

### Configurare HTTPS

Per configurare una connessione HTTPS col registry, fare riferimento alle seguenti istruzioni.

> *Nota*: Usando la fork di Quay predisposta da questa repository, i punti (3) e (4) non sono necessari. Inoltre, il punto (5) è necessario solo se in precedenza è stato eseguito il comando `sudo DOCKER_USER="$(id -u):0" docker-compose up -d quay`.

1. Sulla macchina virtuale target, seguire le procedure descritte ai punti *Creating a certificate authority* e  *Signing a certificate* di [questa documentazione](https://docs.projectquay.io/manage_quay.html#using-ssl-to-protect-quay)
2. A questo punto, copiare i file `ssl.cert` e `ssl.key` in `<quay directory>/local-dev/stack`
3. Modificare il file `<quay directory>/local-dev/stack/config.yaml`, cambiando le seguenti key:
    - `SERVER_HOSTNAME: <hostname scelto in precedenza al punto (1)>`
    - `PREFERRED_URL_SCHEME: https`
4. Aggiungere al file `docker-compose.yaml` il port mapping `443:8443`
5. Eseguire `sudo docker rm -vf quay-quay` e infine `sudo DOCKER_USER="$(id -u):0" docker-compose up -d quay`

Sarà quindi possibile usare il file `rootCA.pem` come certificato della CA per comunicare in HTTPS col registry. Per interagire con Quay dalla macchina locale, trasferire il file `rootCA.pem` sul proprio computer. A questo punto, sarà necessario aggiungere il certificato fra i *trusted root certificates* del sistema.

In particolare:
- Da Linux/WSL, consultare [questa guida](https://www.hs-schmalkalden.de/en/university/faculties/faculty-of-electrical-engineering/studium/use-of-it/install-ca-certificates-on-linux-systems) per istruzioni circa la propria distribuzione.
- Da Windows/Chrome, seguire [questa guida](https://docs.vmware.com/en/VMware-Adapter-for-SAP-Landscape-Management/2.1.0/Installation-and-Administration-Guide-for-VLA-Administrators/GUID-D60F08AD-6E54-4959-A272-458D08B8B038.html) per importare il certificato. 

Sarà adesso possibile interagire col registry all'indirizzo `https://<ip>` o `https://<hostname>` dopo aver aggiunto l'hostname indicato sopra come `SERVER_HOSTNAME` nel proprio `/etc/hosts`; fare riferimento a [questa guida](https://docs.rackspace.com/docs/modify-your-hosts-file) per la procedura.

### Problemi frequenti

1. Se la renderizzazione del frontend presenta problemi, verificare il contenuto della directory `static` e allinearlo rispetto alla build realizzata in locale con `make local-dev-up`. In particolare, il container `quay-local-dev-up` non risulta pienamente stabile.
2. I container di Quay montano la root della repository del codice sorgente come volume. Potrebbe essere quindi necessario effettuare il pruning dei volumi in caso di anomalia rispetto al comportamento atteso, come l'assenza dei manifesti (e di conseguenza l'impossibilità di effettuare il pull) dopo aver caricato un'immagine sul registry.
3. Le porte 8080 e 8443 potrebbero essere bloccate da un firewall. In tal caso, aggiungere i port mapping `80:8080` e `443:8443` nel manifesto del Docker Compose.

## Esempio d'uso

E' possibile usare Podman, Docker o altri tool compatibili con lo standard OCI.

Dalla GUI del registry, creare un account ("*organizzazione*") `test`. Successivamente, creare una repository `test_repository`. Dirigersi adesso nelle impostazioni della repository e creare un *robot account* con permessi di scrittura, prelevando poi il comando di autenticazione alla voce "Podman Login" e immettendolo sul terminale WSL. Adesso, sarà possibile caricare nuove immagini sul registry, taggandole preventivamente con `<hostname>/test/test_repository:test_tag`.

## Preparazione di una release

Per sfruttare l'installazione automatizzata di Quay così come descritto in questa repository, durante la procedura sarà richiesto di scaricare ed estrarre una release `[deps].tar` che contiene le dipendenze da trasferire mediante playbook sul sistema (che, ricordiamo, non prevede accesso ad Internet). 

Una release può essere preparatata accludendo le seguenti dipendenze, nella struttura così definita:

```
deps/
├─ docker/
│  ├─ containerd.io.deb
│  ├─ docker-ce-cli.deb
│  ├─ docker-ce.deb
│  ├─ docker-compose
├─ images/
│  ├─ clair.tar
│  ├─ postgres.tar
│  ├─ redis.tar
│  ├─ quay-local.tar
│  ├─ quay-build.tar
├─ src/
│  ├─ quay.tar
├─ tls/
│  ├─ openssl.cnf
```

E' possibile variare qualsiasi delle dipendenze, in termini di versionamento e contenuti, fermo restando che la struttura della directory `deps` non può variare.

Per ciò che concerne il contenuto di `deps/images`, è importante evidenziare che le immagini salvate mediante `docker save` devono utilizzare un'identificativo pienamente qualificato dell'immagine, avendo cura di includere il tag. Ad esempio, un utilizzo corretto è `sudo docker save localhost/quay-local:latest -o quay-local.tar`. Salvare l'immagine senza indicare il tag causerà una malformazione nel relativo `docker load` eseguito da Ansible.   

Infine, si sottolinea che il file `deps/src/quay.tar` dev'essere generato così come descritto al punto 2 del paragrafo *Installazione manuale*.
