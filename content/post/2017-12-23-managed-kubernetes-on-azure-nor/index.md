---
title:  "Managed Kubernetes på Microsoft Azure (Norwegian)"
date:   2017-12-25 00:00:00 +0000
image: minecraft-k8s.png
categories:
    - technology
tags: 
    - kubernetes
    - azure
excerpt_separator: <!--more-->
---

_Update 29. Dec: There is an [English version of this post here.][33]_

Kubernetes (K8s) er i ferd med å bli de-facto standard for deployments av kontainer-baserte applikasjoner. Microsoft har nå preview av deres managed Kubernetes tjeneste (Azure Kubernetes Service, AKS) som gjør det enkelt å opprette et Kubernetes cluster og rulle ut tjenester uten å måtte ha kompetanse og tid til den daglige driften av selve Kubernetes-clusteret, som per i dag kan være relativt komplisert og tidkrevende.

I denne posten setter vi opp et Kubernetes cluster fra scratch ved bruk av Azure CLI.

<!--more-->

Table of contents

  - [Bakgrunn](#Bakgrunn)
    - [Docker containers](#Dockercontainers)
    - [Container orchestration](#Containerorchestration)
  - [Kom i gang med Azure Kubernetes - AKS](#OppretteAKS)
    - [Forbehold](#Forbehold)
    - [Forberedelser](#Forberedelser)
    - [Azure innlogging](#Azureinnlogging)
    - [Aktiver ContainerService](#AktiverContainerService)
    - [Opprett en resource group](#OpprettResourceGroup)
    - [Opprette Kubernetes cluster](#OpprettK8sCluster)
    - [Installer kubectl](#InstallerKubectl)
    - [Inspiser cluster](#InspiserCluster)
    - [Starte noen nginx containere](#StarteNginx)
    - [Gjøre nginx tilgjengelig med en tjeneste](#NginxService)
    - [Skalere cluster](#ScaleCluster)
    - [Slette cluster](#DeleteCluster)
  - [Bonusmateriale](#Bonusmateriale)
    - [Rulle ut tjenester med Helm pakker](#HelmIntro)
      - [MineCraft server med Helm](#HelmMinecraft)
    - [Kubernetes Dashboard](#KubernetesDashboard)
  - [Konklusjon](#Konklusjon)
  
#### Microsoft Azure

> [Hvis du ikke har Azure fra før kan du prøve tjenester for $200 i 30 dager.][24] VM typen **Standard_B2s** er Burstable, har 2vCPU, 4GB RAM, 8GB temp storage og koster ~$38 / mnd. For $200 kan du ha et cluster på 3-4 B2s noder plus trafikkostnad, lastbalanserere og andre nødvendige tjenester.

> _Vi har ingen tilknytning til Microsoft bortsett fra at de sponser vår startup [DataDynamics][26] med cloud-tjenester i 24 mnd i deres [BizSpark program][25]._

<a id="Bakgrunn"></a>
## Bakgrunn

<a id="Dockercontainers"></a>
###  Docker containers

_Vi tar ikke for oss Docker containers i dybden i denne posten, men her er en kort oppsummering for de som ikke er kjent med teknologien._

Docker er en måte å pakketere programvare slik at det kan kjøres på samtlige populære platformer uten å måtte bruke mye tid på dependencies, oppsett og konfigurasjon.

I tillegg bruker en Docker container operativsystemet på vertsmaskinen når den kjører. Dette gjør at en kan kjøre mange flere containere på samme vertsmaskin sammenlignet med virtuelle maskiner.

Her er en ufullstendig og grov sammenligning mellom en Docker container og en virtuell maskin:

|                     | Virtuel maskin          | Docker container              |
| ------------------- | --------------          | --------------------          |
| Image størrelse     | fra 200MB til mange GB  | fra 10MB til 3-400MB          |
| Oppstartstid        | 60 sekunder +           | 1-10 sekunder                 |
| Minnebruk           | 256MB-512MB-1GB +       | 2MB +                         |
| Sikkerhet           | God isolasjon mellom VM | Dårligere isolasjon mellom containere |
| Bygge image         | Minutter                | Sekunder                      |

> **PS** Tallene for virtuelle maskiner er tatt fra hukommelsen. Jeg forsøkte å starte en MySQL virtuell appliance på min laptop men VMware Player nekter å kjøre pga inkompatibilitet med Windows Hyper-V. VMware Workstation nekter å kjøre pga utgått lisens og Oracle VirtualBox gir en nasty bluescreen gang på gang. Hooray!

> **Protip** De minste og raskeste Docker imagene er bygget på Alpine Linux. For webserveren Nginx er det Alpine-baserte imaget 15MB mot det Debian-baserte imaget på 108MB. PostgreSQL:Alpine er 38MB mot 287MB. Siste versjon av MySQL er 343MB men vil i versjon 8 støtte Alpine Linux også.

Noen av fordelene med Docker containers er altså:
  - Kompatibilitet på tvers av platformer, Linux, Windows og MacOS.
  - 10-100x mindre størrelse. Raskere å laste ned, raskere å bygge, raskere å laste opp.
  - Minnebruk kun for applikasjon og ikke eget OS. 
    - Fordel under utvikling, kan kjøre 10-20-30 Docker containere samtidig på en laptop.
    - Fordel i produksjon, kan redusere hardware utgifter betraktelig.
  - Oppstart på få sekunder. Gjør dynamisk skalering av applikasjoner mye enklere.



[Last ned Docker for Windows her.][27]

Og start en MySQL database fra Windows CMD eller Powershell:

```
docker run --name mysql -p 3306:3306 -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql
```

Stop containeren med:

```
docker kill mysql
```

En kan søke etter ferdige Docker images på [Docker Hub][28]. Det er også mulig å lage private Docker repositories for egen programvare som ikke skal være tilgjengelig for omverden.

<a id="Containerorchestration"></a>
#### Container orchestration

Etter som Docker containers har blitt den foretrukne måten å pakke og distribuere programvare på Linux platformen de siste par årene har det vokst frem et behov for systemer som kan samkjøre drift og utrulling av disse containerene. Ikke ulikt det økosystemet av produkter VMware har bygget opp rundt utvikling og drift av virtuelle maskiner.

Container orchestration systemene har som oppgave å sørge for:
 - Lastbalansering.
 - Service discovery.
 - Health checks.
 - Automatisk skalering og restarting av vertsmaskiner og containere.
 - Oppgraderinger uten nedetid (rolling deploy).

Frem til nylig har økosystemet rundt container orchestration vært fragmentert og de mest populære alternativene har vært:
 - [Kubernetes][4] (Opprinnelig fra Google, nå styrt av CNCF, Cloud Native Computing Foundation)
 - [Swarm][2] (Fra produsenten bak Docker)
 - [Mesos][3] (Fra Apache Software Foundation)
 - [Fleet][1] (Fra CoreOS)

Men det siste året har det vært en konvergens mot Kubernetes som foretrukket løsning.

 - 7 februar
    * [CoreOS annonserer at de fjerner Fleet fra Container Linux og anbefaler Kubernetes][5]
 - 27 juli
    * [Microsoft slutter seg til CNCF][9]
 - 9 august
    * [Amazon Web Services slutter seg til CNCF][7]
 - 29 august
    * [VMware og Pivotal slutter seg til CNCF][10]
 - 17 september
    * [Oracle slutter seg til CNCF][8]
 - 17 oktober
    * [Docker annonserer native støtte for Kubernetes i tillegg til sitt eget Swarm produkt][6]
 - 24 oktober
    * [Microsoft Azure annonserer managed Kubernetes med tjenesten AKS][13]
 - 29 november
    * [Amazon Web Services annonserer managed Kubernetes med tjenesten EKS][12]
 
De to siste nyhetene er spesielt viktige. Å drifte sin egen Kubernetes-installasjon krever tid og kompetanse. ([Les hvordan Stripe brukte 5 måneder på å bli fortrolig med å drifte sitt eget Kubernetes cluster, bare for batch jobs.][15])

Frem til nå har valget vært mellom å drifte sitt eget Kubernetes cluster eller bruke Google Container Engine som har [brukt Kubernetes siden 2014][14]. Mange av oss føler et visst ubehag ved å låse oss til én tilbyder. Men dette er nå anderledes når en kan utvikle infrastruktur på Kubernetes, og velge tilnærmet fritt **\*** mellom de 3 store cloud-tilbyderene i tillegg til å drifte selv om ønskelig.

**\*** Kubernetes utvikles raskt, og funksjonalitet blir ofte ikke tilgjengelig på de ulike platformene samtidig.

<a id="OppretteAKS"></a>
## Opprette Azure Kubernetes Cluster

<a id="Forbehold"></a>
### Forbehold

> Denne gjennomgangen tar utgangspunkt i dokumentasjonen på [Microsoft.com][16]. Å sette opp et Azure Kubernetes cluster fungerte ikke i starten av desember, men per dags dato, 23. desember, ser det ut til å fungere relativt bra. Men, oppgradering av cluster fra Kubernetes 1.7 til 1.8 fungerer for eksempel IKKE.
> 
> AKS er i Preview og Azure jobber kontinuerlig med å gjøre AKS stabilt og støtte så mange Kubernetes-funksjoner som mulig. Amazon Web Services har tilsvarende en lukket invite-only Preview per dags dato mens de også jobber med stabilitet og funksjonalitet.
> 
> Både Azure og AWS uttrykker forventning om at deres Kubernetes tjenester skal være klare for produksjonsmiljø ila 2018.

<a id="Forberedelser"></a>
### Forberedelser

Du behøver Azure-CLI (versjon 2.0.21 eller nyere) for å utføre kommandoene:
 * [Last ned Azure-CLI her][17]
 * [Informasjon om Azure-CLI på MacOS og Linux finner du her][18]

Alle kommandoer gjøres i Windows PowerShell.

<a id="Azureinnlogging"></a>
### Azure innlogging

Logg på Azure:

```
az login
```
Du får en link som du åpner i din browser samt en autentiseringskode. Skriv koden på nettsiden og `az login` lagrer påloggingsinformasjonen slik at du ikke behøver å autentisere igjen på samme maskin.

> **PS** Pålogingsinformasjonen lagres i `C:\Users\Brukernavn\.azure\`. Du må selv passe på at ingen kopierer disse filene. Da får de full tilgang til din Azure konto.

<a id="AktiverContainerService"></a>
### Aktiver ContainerService

Siden AKS er i Preview/Beta må du eksplisitt aktivere det for å få tilgang til `aks` kommandoene.

```
az provider register -n Microsoft.ContainerService
az provider show -n Microsoft.ContainerService
```

<a id="OpprettResourceGroup"></a>
### Opprett en resource group

Her oppretter vi en resource group med navn "min_aks_rg" i Azure region West Europe. 

```
az group create --name min_aks_rg --location westeurope
```

> **Protip** 
> For å se en liste over tilgjengelige Azure regioner, bruk kommandoen `az account list-locations --output table`. **PS** Det kan hende AKS ikke er tilgjengelig i alle regioner enda.

<a id="OpprettK8sCluster"></a>
### Opprette Kubernetes cluster

```
az aks create --resource-group min_aks_rg --name mitt_cluster --node-count 3 --generate-ssh-keys --node-vm-size Standard_B2s --node-osdisk-size 256 --kubernetes-version 1.8.2
```

  * `--node-count` 
    - Antall vertsmaskiner tilgjengelig for å kjøre containers
  * `--generate-ssh-keys`
    - Oppretter og outputter en SSH key som kan brukes for å SSHe direkte til vertsmaskinene.
  * `--node-vm-size`
    - Hvilken type Azure VM clusteret skal bestå av. For å se tilgjengelige størrelser bruk `az vm list-sizes -l westeurope --output table` og [Microsofts nettsider.][19]
  * `--node-osdisk-size`
    - Disk størrelse på vertsmaskiner i GB. **PS** Conteinere kan bli stoppet og flyttet til en annen host ved behov eller hvis en vertsmaskin forsvinner. Alle data lagret lokalt i conteineren blir da borte. Hvis en skal lagre ting permanent må en bruke PersistentVolumes og ikke lokal disk på vertsmaskin.
  * `--kubernetes-version`
    - Hvilken Kubernetes versjon som skal installeres. Azure installerer IKKE den siste versjonen som standard, og per dags dato fungerer ikke `az aks upgrade` tilstrekkelig. Siste tilgjengelige versjon per dags dato er 1.8.2. Det er en fordel å bruke siste versjon da det skjer store forbedringer i Kubernetes fra versjon til versjon. Dokumentasjon er også mye bedre for nyere versjoner.
  
Lagre teksten som kommandoen spytter ut i en fil på en trygg plass. Den inneholder nøkler som kan brukes for å kople til clusteret med SSH. Selv om det i teorien ikke skal være nødvendig.

<a id="InstallerKubectl"></a>
### Installer kubectl

`kubectl` er klienten som gjør alle operasjoner mot ditt Kubernetes cluster. Azure CLI kan installere `kubectl` for deg:

```
az aks install-cli
```

Etter `kubectl` er installert behøver vi å få påloggingsinformasjon slik at `kubectl` kan kommunisere med Kubernetes clusteret.

```
az aks get-credentials --resource-group min_aks_rg --name mitt_cluster
```
Påloggingsinformasjonen lagres i `C:\Users\Brukernavn\.kube\config`. Hold disse filene hemmelig også.

> **Protip** Når en har flere ulike Kubernetes clusters kan en bytte hvilken `kubectl` skal snakke til med `kubectl config get-contexts` og `kubectl config set-context mitt_cluster`.

<a id="InspiserCluster"></a>
### Inspiser cluster

For å se at clusteret og `kubectl` virker begynner vi med noen kommandoer.

Se alle vertsmaskiner og status:
```
> kubectl get nodes
NAME                       STATUS    AGE       VERSION
aks-nodepool1-16970026-0   Ready     15m       v1.8.2
aks-nodepool1-16970026-1   Ready     15m       v1.8.2
aks-nodepool1-16970026-2   Ready     15m       v1.8.2
```

Se alle tjenester, pods, deployments:

```
> kubectl get all --all-namespaces

NAMESPACE     NAME                                          READY     STATUS    RESTARTS   AGE
kube-system   po/kubernetes-dashboard-6fc8cf9586-frpkn      1/1       Running   0          3d

NAMESPACE     NAME                          CLUSTER-IP     EXTERNAL-IP     PORT(S)           AGE
kube-system   svc/kubernetes-dashboard      10.0.161.132   <none>          80/TCP            3d

NAMESPACE     NAME                             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-system   deploy/kubernetes-dashboard      1         1         1            1           3d

NAMESPACE     NAME                                    DESIRED   CURRENT   READY     AGE
kube-system   rs/kubernetes-dashboard-6fc8cf9586      1         1         1         3d
```

Jeg har bare tatt et lite utdrag fra denne kommandoen. Du behøver ikke å forstå hva alle ressursene i `kube-system` namespacet gjør. Det er hensikten at du skal slippe det når Microsoft står for management av selve clusteret.

> **Namespaces**
> I Kubernetes er det noe som heter Namespaces. Ressurser i ett namespace har ikke automatisk tilgang til ressurser i et annet namespace. Tjenestene som Kubernetes selv benytter installeres i namespacet `kube-system`. Kommandoen `kubectl` viser deg vanligvis bare ressurser i `default` namespace med mindre du spesifiserer `--all-namespaces` eller `--namespace=xx`.

<a id="StarteNginx"></a>
### Starte noen nginx containere

> En instans av en kjørende container kalles i Kubernetes for en **Pod**.

> `nginx` er en rask og fleksibel webserver.

Nå som clusteret er oppe å kjøre kan vi begynne å rulle ut tjenster og deployments på det.

Vi begynner med å lage en Deployment bestående av 3 containere som alle kjører `nginx:mainline-alpine` imaget fra [Docker hub][31].

**nginx-dep.yaml** ser slik ut:
```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:mainline-alpine
        ports:
        - containerPort: 80
```

Last denne inn på clusteret med `kubectl create`:

```
kubectl create -f https://raw.githubusercontent.com/StianOvrevage/gh-pages-blog/master/static/attachments/2017-12-23-managed-kubernetes-on-azure/nginx-dep.yaml

```

Denne kommandoen oppretter ressursene beskrevet i filen. `kubectl` kan lese filer enten lokalt fra din maskin eller fra en URL.

> Etter du har gjort endringer i en ressurs-definisjon (`.yaml` fil) kan du oppdatere ressursene i clusteret med `kubectl replace -f ressurs.yaml`

Vi kan verifisere at Deployment er klar:

```
> kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           10m
```

Vi kan også hente de faktiske Pods som er startet:

```
> kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-569477d6d8-dqwx5   1/1       Running   0          10m
nginx-deployment-569477d6d8-xwzpw   1/1       Running   0          10m
nginx-deployment-569477d6d8-z5tfk   1/1       Running   0          10m
```

> **Logger** Vi kan se logger fra én pod med `kubectl logs nginx-deployment-569477d6d8-xwzpw`. Men siden vi i dette tilfellet ikke vet hvilken Pod som ender opp med å få innkommende forespørsler kan vi se logger fra alle Pods som har `app=nginx` label: `kubectl logs -lapp=nginx`. At vi her bruker `app=nginx` har vi selv bestemt i `nginx-dep.yaml` når vi satt `spec.template.metadata.labels: app: nginx`.

<a id="NginxService"></a>
### Gjøre nginx tilgjengelig med en tjeneste

For å kommunisere med våre nye Pods behøver vi å opprette en tjeneste (**Service**). En tjeneste består av en eller flere Pods som velges basert på ulike kriterier, blant annet hvilke labels de har og om Podene det gjelder er Running og Ready.

Nå lager vi en tjeneste som ruter trafikk til alle Pods som har label `app: nginx` og som lytter på port 80. I tillegg gjør vi tjenesten tilgjengelig via en LoadBalancer:

**nginx-svc.yaml** ser slik ut:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
```

Vi ber Kubernetes om å opprette tjeneten vår med `kubectl create` som vanlig:

```
kubectl create -f https://raw.githubusercontent.com/StianOvrevage/gh-pages-blog/master/static/attachments/2017-12-23-managed-kubernetes-on-azure/nginx-svc.yaml
```

Deretter kan vi se hvilken IP-adresse tjenesten vår har fått av Azure:

```
> kubectl get svc -w
NAME         CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
nginx        10.0.24.11   13.95.173.255   80:31522/TCP   15m
```

> **PS** Det kan ta et par minutter for Azure å tildele tjenesten vår en Public IP, i mellomtiden vil det stå `<pending>` under EXTERNAL-IP.

En enkel **Welcome to nginx** webside skal nå være tilgjengelig på http://13.95.173.255 (_husk å bytt ut med din egen External-IP_).

**Vi har nå en lastbalansert `nginx` tjeneste med 3 servere klar til å ta imot trafikk.**

For ordens skyld kan vi slette tjeneste og deployment etterpå:
```
kubectl delete svc nginx
kubectl delete deploy nginx-deployment
```

<a id="ScaleCluster"></a>
### Skalere cluster

Hvis en ønsker å endre antall vertsmaskiner/noder som kjører Pods kan en gjøre det via Azure-CLI:
```
az aks scale --name mitt_cluster --resource-group min_aks_rg --node-count 5
```
> For øyeblikket blir alle noder opprettet med samme størrelse som når clusteret ble opprettet. AKS vil antageligvis få støtte for [**node-pools**][20] i løpet av neste år. Da kan en opprette grupper av noder med forskjellig størrelse og operativsystem, både Linux og Windows.


<a id="DeleteCluster"></a>
### Slette cluster

En kan slette hele clusteret slik:

```
az aks delete --name mitt_cluster --resource-group min_aks_rg --yes
```

<a id="Bonusmateriale"></a>
## Bonusmateriale

Her er litt bonusmateriale dersom du ønsker å gå enda litt videre med Kubernetes.

<a id="HelmIntro"></a>
### Rulle ut tjenester med Helm

[Helm][21] er en pakke-behandler og et bibliotek av programvare som er klart for å rulles ut i et Kubernetes-cluster.

Start med å laste ned [Helm-klienten][22]. Den henter påloggingsinformasjon osv fra samme sted som `kubectl` automatisk.

Installer Helm-serveren (Tiller) på Kubernetes clusteret og oppdater pakke-biblioteket:

```
helm init
helm repo update
```

Se tilgjengelige pakker (**Charts**) med: `helm search`.

<a id="HelmMinecraft"></a>
#### Rulle ut MineCraft med Helm

La oss rulle ut en MineCraft installasjon på clusteret vårt, fordi vi kan :-)

```
helm install --name stian-sin --set minecraftServer.eula=true stable/minecraft
```
> `--set` overstyrer en eller flere av standardverdiene som er satt i pakken. MineCraft pakken er laget slik at den ikke starter uten å ha sagt seg enig i brukervilkårene i variabelen `minecraftServer.eula`. Alle variablene som kan overstyres i MineCraft pakken er [dokumentert her][23].

Så venter vi litt på at Azure skal tildele en Public IP:

```
> kubectl get svc -w
stian-sin-minecraft   10.0.237.0   13.95.172.192   25565:30356/TCP   3m
```

Og vipps kan vi kople til Minecraft på `13.95.172.192:25565`.

![Kubernetes in MineCraft on Kubernetes](minecraft-k8s.png)

<a id="KubernetesDashboard"></a>
### Kubernetes Dashboard

Kubernetes har også et grafisk web-grensesnitt som gjør det litt lettere å se hvilke ressurser som er i clusteret, se logger og åpne remote-shell inne i en kjørende Pod, blant annet.

```
> kubectl proxy
Starting to serve on 127.0.0.1:8001
```

`kubectl` krypterer og tunnelerer trafikken inn til Kubernetes' API servere. Dashboardet er tilgjengelig på [http://127.0.0.1:8001/ui/][32].

![Kubernetes Dashboard](k8s-dash.png)


<a id="Konklusjon"></a>
## Konklusjon

Jeg håper du har fått mersmak for Kubernetes. Lærekurven kan være litt bratt i begynnelsen men det tar ikke så veldig lang tid før du er produktiv.

Se på de [offisielle guidene på Kubernetes.io][29] for å lære mer om hvordan du definerer forskjellige typer ressurser og tjenester for å kjøre på Kubernetes. **PS: Det gjøres store endringer fra versjon til versjon så sørg for å bruke dokumentasjonen for riktig versjon!**

Kubernetes har også et veldig aktivt Slack-miljø på [kubernetes.slack.com][30]. Der er det også en kanal for norske Kubernetes brukere; **#norw-users**.



[1]: https://github.com/coreos/fleet
[2]: https://docs.docker.com/engine/swarm/
[3]: http://mesos.apache.org/
[4]: https://kubernetes.io/
[5]: https://coreos.com/blog/migrating-from-fleet-to-kubernetes.html
[6]: https://www.theregister.co.uk/2017/10/17/docker_ee_kubernetes_support/
[7]: https://techcrunch.com/2017/08/09/aws-joins-the-cloud-native-computing-foundation/
[8]: https://techcrunch.com/2017/09/13/oracle-joins-the-cloud-native-computing-foundation-as-a-platinum-member/
[9]: https://azure.microsoft.com/en-us/blog/announcing-cncf/
[10]: https://www.geekwire.com/2017/now-vmware-pivotal-cncf-becoming-hub-enterprise-tech/
[11]: https://cloudplatform.googleblog.com/2014/11/unleashing-containers-and-kubernetes-with-google-compute-engine.html
[12]: https://aws.amazon.com/blogs/aws/amazon-elastic-container-service-for-kubernetes/
[13]: https://azure.microsoft.com/en-us/blog/introducing-azure-container-service-aks-managed-kubernetes-and-azure-container-registry-geo-replication/
[14]: https://cloudplatform.googleblog.com/2014/11/unleashing-containers-and-kubernetes-with-google-compute-engine.html
[15]: https://stripe.com/blog/operating-kubernetes
[16]: https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough
[17]: https://aka.ms/InstallAzureCliWindows
[18]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
[19]: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes
[20]: https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools
[21]: https://helm.sh/
[22]: https://github.com/kubernetes/helm/releases
[23]: https://github.com/kubernetes/charts/blob/master/stable/minecraft/values.yaml
[24]: https://azure.microsoft.com/en-us/free/
[25]: https://bizspark.microsoft.com/
[26]: http://www.datadynamics.no/
[27]: https://store.docker.com/editions/community/docker-ce-desktop-windows
[28]: https://hub.docker.com/
[29]: https://v1-8.docs.kubernetes.io/docs/tutorials/
[30]: http://slack.k8s.io/
[31]: https://hub.docker.com/r/_/nginx/
[32]: http://127.0.0.1:8001/ui/
[33]: /2017/12/29/managed-kubernetes-on-azure-eng.html
