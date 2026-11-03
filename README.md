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
  image: ghcr.io/dmarc-analyzer-net/dmarc-analyzer:latest
  imagePullPolicy: Always
  database:
    database: dmarc-analyser
  envFrom:
    - secretRef:
        name: dmarc-analyzer
```

FLAIS håndterer kjøring av containeren og provisjonering av databasen `dmarc-analyser`. Det skal derfor ikke opprettes en separat PostgreSQL `Deployment`, `Service` eller `PersistentVolumeClaim` i dette repositoriet.

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

## Validering

Render manifestene lokalt før deployment:

```shell
kubectl kustomize kustomize/overlays/beta/novari-no
kubectl kustomize kustomize/overlays/api/novari-no
```

Begge kommandoene skal produsere én FLAIS `Application` og én `OnePasswordItem` i namespacet `novari-no`.
