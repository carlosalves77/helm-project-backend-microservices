# backend-app

Chart Helm para deploy do **Products Service** (API Spring Boot) e seu banco de dados **PostgreSQL** em um cluster Kubernetes.

## Visão geral

Este chart provisiona:

- **`products-service`** — `Deployment` + `Service` (ClusterIP) da aplicação Spring Boot, exposta na porta `4100`. Um `initContainer` aguarda o Postgres ficar disponível antes de subir a aplicação, e os probes de *readiness*/*liveness* usam o endpoint `/actuator/health`.
- **`product-service-database`** — `StatefulSet` + `Service` headless (`clusterIP: None`) rodando `postgres:16`, com `PersistentVolumeClaim` (`volumeClaimTemplate`) de 5Gi para persistência dos dados.
- **`postgres-pvc`** — `PersistentVolumeClaim` avulso de 5Gi (armazenamento adicional).
- **`products-service-secret`** — `Secret` (`Opaque`) com as credenciais do Postgres (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`), preenchido a partir de `values.yaml`.

Todos os recursos são criados no namespace `default`.

## Requisitos

- Kubernetes >= 1.19
- Helm >= 3
- Um `StorageClass` padrão disponível no cluster (para os PVCs)

## Configuração (`values.yaml`)

O `Secret` do banco depende dos seguintes valores, que **devem** ser definidos antes da instalação (o `values.yaml` do chart está vazio por padrão):

```yaml
postgres:
  user: <usuario>
  password: <senha>
  database: <nome_do_banco>
```

## Instalação

```bash
helm install backend-app . \
  --set postgres.user=<usuario> \
  --set postgres.password=<senha> \
  --set postgres.database=<nome_do_banco>
```

Ou usando um arquivo de valores próprio:

```bash
helm install backend-app . -f my-values.yaml
```

## Atualização

```bash
helm upgrade backend-app .
```

## Desinstalação

```bash
helm uninstall backend-app
```

> ⚠️ O `uninstall` não remove os `PersistentVolumeClaim`s. Caso queira apagar os dados do Postgres, remova-os manualmente:
> ```bash
> kubectl delete pvc postgres-pvc postgres-storage-product-service-database-0
> ```

## Estrutura do chart

```
backend-app/
├── Chart.yaml
├── values.yaml
├── .helmignore
└── templates/
    ├── product-service-deployment.yaml       # Deployment da aplicação
    ├── product-service-svc.yaml              # Service da aplicação (porta 4100)
    ├── product-service-postgres-statefulset.yaml  # StatefulSet do Postgres
    ├── product-service-postgres-svc.yaml     # Service headless do Postgres
    ├── product-service-postgres-pvc.yaml     # PVC adicional
    └── secrets.yaml                          # Secret com credenciais do banco
```

## Conectividade interna

- Aplicação → Banco: `jdbc:postgresql://product-service-database:5432/product_service_database`
- Acesso à aplicação dentro do cluster: `products-service:4100`

## Imagem da aplicação

- `carlosalves77/products-service:v.0.0.1`

## Observações

- Os nomes dos recursos e a imagem da aplicação estão atualmente fixos (*hardcoded*) nos templates, não parametrizados via `values.yaml` — exceto as credenciais do Postgres.
- `appVersion` do chart: `1.16.0`.
