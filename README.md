# WordPress na k3s z ArgoCD i Longhorn

GitOps deployment WordPressa z MySQL na k3s, zarządzany przez ArgoCD z Kustomize.

## Stos technologiczny

- **k3s** — lekka dystrybucja Kubernetes
- **ArgoCD** — GitOps, automatyczny sync klastra z repozytorium
- **Kustomize** — zarządzanie manifestami
- **Longhorn** — distributed block storage (RWO)

## Struktura repo

```
├── argocd-app.yaml          # Rejestracja aplikacji w ArgoCD
├── kustomization.yaml       # Punkt wejścia Kustomize
├── namespace.yaml           # Namespace wordpress
├── mysql/
│   ├── deployment.yaml
│   ├── pvc.yaml             # 5Gi RWO, storageClass: longhorn
│   ├── secret.env.example   # Szablon zmiennych środowiskowych
│   └── service.yaml         # ClusterIP :3306
└── wordpress/
    ├── deployment.yaml
    ├── pvc.yaml             # 5Gi RWO, storageClass: longhorn
    └── service.yaml         # LoadBalancer :80
```

## Wymagania wstępne

- k3s z działającym Longhorn
- ArgoCD zainstalowany w namespace `argocd`
- `kubectl` skonfigurowany z dostępem do klastra

## Deployment

### 1. Stwórz Secret ręcznie

Secret nie jest przechowywany w repo (repo jest publiczne). Przed pierwszym deploymentem utwórz go bezpośrednio w klastrze:

```bash
kubectl create namespace wordpress

kubectl create secret generic mysql-secret -n wordpress \
  --from-literal=MYSQL_ROOT_PASSWORD='twoje-haslo' \
  --from-literal=MYSQL_DATABASE=wordpress \
  --from-literal=MYSQL_USER=wp \
  --from-literal=MYSQL_PASSWORD='twoje-haslo-wp'
```

Wymagane klucze znajdziesz w `mysql/secret.env.example`.

### 2. Zarejestruj aplikację w ArgoCD

```bash
kubectl apply -f argocd-app.yaml
```

ArgoCD automatycznie pobierze repo i zastosuje wszystkie manifesty. Od tej chwili każdy `git push` wyzwala sync.

### 3. Sprawdź status

```bash
kubectl get application wordpress -n argocd
kubectl get pods -n wordpress
kubectl get svc wordpress -n wordpress   # → EXTERNAL-IP
```

WordPress będzie dostępny pod `http://EXTERNAL-IP`.

## Zarządzanie secretami

Sekrety **nie trafiają do repo**. Podejście świadome:

| Zaleta | Wada |
|--------|------|
| Zero ryzyka wycieku w publicznym repo | Secret nie przeżywa usunięcia namespace |
| Brak zależności od dodatkowych narzędzi | Wymaga ręcznego odtworzenia po resecie klastra |

Na produkcję zalecane jest [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) — zaszyfrowany YAML bezpieczny do commitowania.

## Uwagi operacyjne

**Longhorn RWO** — WordPress działa na jednej replice. Skalowanie poziome wymaga RWX (ReadWriteMany), co z kolei wymaga `nfs-common` na węzłach:
```bash
sudo apt-get install -y nfs-common   # na każdym węźle
```

**ArgoCD `prune: true`** — zasoby usunięte z git są usuwane z klastra. Nie usuwaj PVC z repo przypadkowo — Longhorn skasuje dane.

**Odtworzenie po resecie klastra:**
```bash
# Odtwórz secret (jedyna ręczna czynność)
kubectl create secret generic mysql-secret -n wordpress \
  --from-literal=MYSQL_ROOT_PASSWORD='...' \
  --from-literal=MYSQL_DATABASE=wordpress \
  --from-literal=MYSQL_USER=wp \
  --from-literal=MYSQL_PASSWORD='...'

kubectl apply -f argocd-app.yaml
```
