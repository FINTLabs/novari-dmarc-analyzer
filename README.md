# DMARC Analyzer

Dette repositoriet inneholder deployment-konfigurasjon for DMARC Analyzer i Novari. Applikasjonen deployes med FLAIS og har Kustomize-overlays for `novari-no` i både beta- og api.

## FLAIS-applikasjonen

Basemanifestet ligger i [`kustomize/base/flais.yaml`](kustomize/base/flais.yaml):

```yaml
apiVersion: fintlabs.no/v1alpha1
kind: Application
metadata:
  name: dmarc-analyzer
  namespace: novari-no
spec:
  orgId: novari.no
  image: ghcr.io/dmarc-analyzer-net/dmarc-analyzer@sha256:c232a277c8c59bc564d4cfaba3fbad75b1bec95275cdb34e176619f278952064
  imagePullPolicy: Always
  database:
    database: flais
  env:
    - name: ConnectionStrings__Default
      value: "Host=$(DB_HOST);Port=$(DB_PORT);Database=$(DB_NAME);Username=$(fint.database.username);Password=$(fint.database.password);SSL Mode=Require"
    - name: Logging__LogLevel__Microsoft.EntityFrameworkCore.Database.Command
      value: Warning
  envFrom:
    - secretRef:
        name: novari-dmarc-analyzer
```

FLAIS håndterer kjøring av containeren og provisjonering av en applikasjonsbruker og et schema i den eksisterende databasen `flais`. Det skal derfor ikke opprettes en separat PostgreSQL `Deployment`, `Service` eller `PersistentVolumeClaim` i dette repositoriet.

Applikasjonens `ConnectionStrings__Default` bygges fra databaseverdiene som FLAIS og 1Password gjør tilgjengelig:

- `DB_HOST`, `DB_PORT` og `DB_NAME` fra 1Password
- `fint.database.username` og `fint.database.password` fra den FLAIS-genererte database-secreten

Databasemigrering og OIDC client secret konfigureres med nøklene `Database__MigrateOnStartup` og `Auth__Oidc__ClientSecret` i Kubernetes-secreten `novari-dmarc-analyzer`. Begge importeres direkte gjennom `envFrom`.

## Hemmeligheter fra 1Password

[`kustomize/base/onePassword.yaml`](kustomize/base/onePassword.yaml) oppretter en `OnePasswordItem`. 1Password-operatoren gjør elementet tilgjengelig som en Kubernetes Secret med navnet `novari-dmarc-analyzer`. Hele secreten leses inn som miljøvariabler gjennom `spec.envFrom` i FLAIS-applikasjonen.

1Password-elementet må inneholde følgende felter med nøyaktig disse navnene:

```text
Auth__Oidc__Enabled
Auth__Oidc__Authority
Auth__Oidc__ClientId
Auth__Oidc__ClientSecret
Auth__Oidc__DisplayName
Auth__Oidc__AutoProvision
Database__MigrateOnStartup
DB_HOST
DB_PORT
DB_NAME
MtaSts__PolicyHost
Security__CredentialEncryptionKey
```

`Security__CredentialEncryptionKey` brukes til å kryptere lagrede mailbox-legitimasjoner. Dersom nøkkelen endres eller går tapt, kan eksisterende legitimasjoner ikke dekrypteres og må registreres på nytt.

Hemmeligheter skal ikke legges direkte i manifestene eller committes til repositoriet.

## DMARC Collector

[`kustomize/base/dmarc-collector.yaml`](kustomize/base/dmarc-collector.yaml) inneholder en `CronJob` som hvert femte minutt leser uleste DMARC-rapporter fra Microsoft Graph og sender vedleggene til DMARC Analyzer. `concurrencyPolicy: Forbid` hindrer overlappende kjøringer. Kubernetes beholder den siste vellykkede og den siste feilede Job-en.

Collector-scriptet monteres fra en `ConfigMap`. Derfor er collectoren en ordinær Kubernetes `CronJob`, ikke en FLAIS `Application`: FLAIS `Job` støtter ikke den nødvendige command- og ConfigMap-volume-konfigurasjonen. Collector og FLAIS-applikasjonen opprettes likevel sammen av samme Kustomize-overlay. Første planlagte kjøring skjer ved neste femminuttersgrense.

[`kustomize/base/onePassword-collector.yaml`](kustomize/base/onePassword-collector.yaml) synkroniserer 1Password-elementet til Kubernetes-secreten `novari-dmarc-collector`. Elementet må inneholde:

```text
AZURE_TENANT_ID
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
DMARC_MAILBOX
DMARC_ANALYZER_API_KEY
DMARC_ANALYZER_URL
```

`DMARC_ANALYZER_API_KEY` må være en machine credential med rettighet til report ingestion. Siden collectoren kjører i samme namespace, kan `DMARC_ANALYZER_URL` bruke den interne adressen `http://dmarc-analyzer:8080`.

Loggnivå styres med `LOG_LEVEL` i CronJob-manifestet. Bruk `DEBUG` under testing og sett det tilbake til `INFO` når feilsøkingen er ferdig. Debug-logging viser kontrollflyt og antall meldinger/vedlegg, men logger ikke access tokens eller secret-verdier.

## MTA-STS policyvert

API-overlegget oppretter en separat FLAIS `Application` med navnet `dmarc-analyzer-mta-sts`. Den bruker samme digest-pinnede DMARC Analyzer-image som hovedapplikasjonen, men kjører med:

```text
APP_MODE=mta-sts
```

Denne modusen har bare MTA-STS-policyer, `/mta-sts/ask` og health-endepunktene. Den starter ikke konsoll, API, autentisering eller worker. Applikasjonen har én replika og små ressursgrenser. Den offentlige FLAIS-ingressen er begrenset til:

```text
https://mta-sts.novari.no/.well-known/mta-sts.txt
```

API-overlegget definerer dette som:

```yaml
routes:
  - host: mta-sts.novari.no
    path: /.well-known/mta-sts.txt
```

Health-endepunktene brukes direkte av Kubernetes-probene og trenger ikke være offentlig eksponert.

Policyverten gjenbruker hovedapplikasjonens secrets gjennom `envFrom`: `novari-dmarc-analyzer` gir `DB_HOST`, `DB_PORT` og `DB_NAME`, mens den FLAIS-genererte `dmarc-analyzer-db` gir `fint.database.username` og `fint.database.password`. Den oppretter ikke en egen FLAIS-databasebruker.

Hovedapplikasjonen leser følgende verdi fra 1Password-elementet `novari-dmarc-analyzer` gjennom `envFrom`:

```text
MtaSts__PolicyHost=<felles CNAME-mål>
```

Ikke legg `APP_MODE=mta-sts` i det delte 1Password-elementet. Det ville også starte hovedapplikasjonen i MTA-STS-modus og hindre den i å kjøre databasemigreringene. `APP_MODE=mta-sts` settes bare eksplisitt på `dmarc-analyzer-mta-sts`.

Dette er det felles CNAME-målet konsollet viser i publiseringsinstruksjonene. Hvis verdien settes til `sts.novari.no`, blir DNS-oppsettet for `novari.no` eksempelvis:

```text
sts.novari.no             -> API-ingressens DNS-navn
mta-sts.novari.no CNAME   sts.novari.no
_mta-sts.novari.no TXT    v=STSv1; id=<id fra konsollet>
```

NetIQ må ha et gyldig sertifikat for `mta-sts.novari.no`, videresende den opprinnelige `Host`-headeren og sende trafikken videre til Traefik. Ingressen må ha én route for hvert offentlig `mta-sts.<domene>`-navn. Nye domener legges til som flere routes på `dmarc-analyzer-mta-sts` i API-overlegget; de trenger ikke egne pods eller FLAIS-applikasjoner.

MTA-STS-applikasjonen migrerer aldri databasen. Hovedapplikasjonen må derfor ha opprettet schemaet før readiness-proben på `/health/ready` blir grønn.

Verifiser hele den offentlige kjeden etter DNS-endringen:

```shell
curl -i https://mta-sts.novari.no/.well-known/mta-sts.txt
```

Svaret skal være HTTP 200 med `Content-Type: text/plain` og policyen som er konfigurert i DMARC Analyzer.

## Miljøer

Prosjektet har to overlays:

| Miljø | Overlay                              | Ingress                    | Analyzer-element                                     | Collector-element                                     |
|--------|--------------------------------------|----------------------------|------------------------------------------------------|-------------------------------------------------------|
| Beta   | `kustomize/overlays/beta/novari-no` | `dmarc-beta.novari.no`     | `vaults/aks-beta-vault/items/novari-dmarc-analyzer` | `vaults/aks-beta-vault/items/novari-dmarc-collector` |
| API    | `kustomize/overlays/api/novari-no`  | `dmarc-analyzer.novari.no` | `vaults/aks-api-vault/items/novari-dmarc-analyzer`  | `vaults/aks-api-vault/items/novari-dmarc-collector`  |

Overlayene bruker samme basemanifest, men definerer hver sin ingress og peker på hvert sitt 1Password-vault. Basemanifestet inneholder ingen miljøspesifikk ingress.

Begge overlayene merker ressursene med de obligatoriske FLAIS-labelene, inkludert `app.kubernetes.io/part-of: it-operations`, `fintlabs.no/org-id: novari.no` og `fintlabs.no/team: flais`.

## Validering

Render manifestene lokalt før deployment:

```shell
kubectl kustomize kustomize/overlays/beta/novari-no
kubectl kustomize kustomize/overlays/api/novari-no
```

Beta skal produsere én FLAIS `Application`, én collector-`CronJob`, én collector-`ConfigMap` og to `OnePasswordItem`-ressurser. API skal i tillegg produsere FLAIS-applikasjonen `dmarc-analyzer-mta-sts`.

Etter deployment kan collectoren kontrolleres med:

```shell
kubectl get cronjob dmarc-collector -n novari-no
kubectl get onepassworditem novari-dmarc-collector -n novari-no
kubectl get secret novari-dmarc-collector -n novari-no
```

Trigger en testkjøring uten å vente på neste femminuttersgrense:

```shell
kubectl create job --from=cronjob/dmarc-collector dmarc-collector-manual -n novari-no
kubectl logs job/dmarc-collector-manual -n novari-no --follow
```

## Oppdatere container-image

Container-imaget er låst til en OCI-digest slik at en eksisterende deployment ikke endres dersom `latest` flyttes til en ny versjon. Finn digesten som `latest` peker på med:

```shell
docker buildx imagetools inspect ghcr.io/dmarc-analyzer-net/dmarc-analyzer:latest
```

Bruk verdien fra `Digest` i både `kustomize/base/flais.yaml` og den lokale `compose.yml`:

```text
ghcr.io/dmarc-analyzer-net/dmarc-analyzer@sha256:<digest>
```

Valider og test den nye versjonen før den oppdaterte digesten deployes.
