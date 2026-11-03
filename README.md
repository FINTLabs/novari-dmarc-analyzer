# DMARC Analyzer

Dette repositoriet inneholder deployment-konfigurasjon for DMARC Analyzer i Novari-plattformen. Applikasjonen deployes med FLAIS og har Kustomize-overlays for `novari-no` i både beta- og API-miljøet.

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
    database: dmarc-analyzer
  env:
    - name: ConnectionStrings__Default
      value: "${fint.database.url};Username=${fint.database.username};Password=${fint.database.password}"
  envFrom:
    - secretRef:
        name: dmarc-analyzer
```

FLAIS håndterer kjøring av containeren og provisjonering av databasen `dmarc-analyzer`. Det skal derfor ikke opprettes en separat PostgreSQL `Deployment`, `Service` eller `PersistentVolumeClaim` i dette repositoriet.

Applikasjonens `ConnectionStrings__Default` bygges fra databaseverdiene som FLAIS gjør tilgjengelig:

- `${fint.database.url}`
- `${fint.database.username}`
- `${fint.database.password}`

## Hemmeligheter fra 1Password

[`kustomize/base/onePassword.yaml`](kustomize/base/onePassword.yaml) oppretter en `OnePasswordItem`. 1Password-operatoren gjør elementet tilgjengelig som en Kubernetes Secret med navnet `dmarc-analyzer`. Hele secreten leses inn som miljøvariabler gjennom `spec.envFrom` i FLAIS-applikasjonen.

1Password-elementet må inneholde følgende felter med nøyaktig disse navnene:

```text
Auth__Oidc__Enabled
Auth__Oidc__Authority
Auth__Oidc__ClientId
Auth__Oidc__ClientSecret
Auth__Oidc__DisplayName
Auth__Oidc__AutoProvision
Security__CredentialEncryptionKey
```

`Security__CredentialEncryptionKey` brukes til å kryptere lagrede mailbox-legitimasjoner. Dersom nøkkelen endres eller går tapt, kan eksisterende legitimasjoner ikke dekrypteres og må registreres på nytt.

Hemmeligheter skal ikke legges direkte i manifestene eller committes til repositoriet.

## Miljøer

Prosjektet har to overlays:

| Miljø | Overlay                             | 1Password-element                                   |
|-------|-------------------------------------|-----------------------------------------------------|
| Beta  | `kustomize/overlays/beta/novari-no` | `vaults/aks-beta-vault/items/novari-dmarc-analyzer` |
| API   | `kustomize/overlays/api/novari-no`  | `vaults/aks-api-vault/items/novari-dmarc-analyzer`  |

Overlayene bruker samme basemanifest, men peker på hvert sitt 1Password-vault.

Begge overlayene merker ressursene med `fintlabs.no/org-id: novari.no` og `fintlabs.no/team: flais`. FLAIS bruker blant annet organisasjonslabelen når den bygger den underliggende deploymenten.

## Validering

Render manifestene lokalt før deployment:

```shell
kubectl kustomize kustomize/overlays/beta/novari-no
kubectl kustomize kustomize/overlays/api/novari-no
```

Begge kommandoene skal produsere én FLAIS `Application` og én `OnePasswordItem` i namespacet `novari-no`.

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
