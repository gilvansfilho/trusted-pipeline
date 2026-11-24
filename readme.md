# Trusted Pipeline

Esse é um laboratório que busca demonstrar como implantar no Red Hat Openshift Container Platform uma solução de pipeline segura (trusted software supply chain) usando as seguintes soluções de middleware:

- Red Hat Build of Keycloak
- Red Hat Trusted Profile Analyzer
- Red Hat Trusted Artifact Signer
- Red Hat Openshift Pipelines + Tekton Chains

## Preparação do Ambiente

Antes de iniciar defina as seguintes variáveis de ambiente:

- CLUSTER_DOMAIN
- RHBK_OPERATOR_CHANNEL

Exemplo:

```
export CLUSTER_DOMAIN="apps.cluster-52k9v.dyn.redhatworkshops.io"
export RHBK_OPERATOR_CHANNEL="stable-v26.4"
```

Ou atualize e use o arquivo `.env`:

```
set -a
source .env
set +a
```

## Instalação do Red Hat Build of Keycloak

### Instalar o Operator

```
envsubst < rhbk\operator/1-namespace.yaml | oc apply -f -
envsubst < rhbk\operator/2-operator-group.yaml | oc apply -f -
envsubst < rhbk\operator/3-subscription.yaml | oc apply -f -
```

### Instalar o RHBK e seus pre-reqs

```
# Deploy de um banco de dados postgresql (Pré-Requisito do RHBK)
envsubst < rhbk\instance\1-tssc-rhbk-database.yaml | oc apply -f -

# Aguarde o banco de dados ficar pronto
oc wait --for=jsonpath='{.status.phase}'=Running pod -l app=tssc-rhbk-database

# Deploy do RHBK
envsubst < rhbk\instance\2-tssc-rhbk-instance.yaml | oc apply -f -
```

### Criar e preparar o realm do RHTPA

Source: https://github.com/guacsec/trustify/blob/main/docs/book/modules/admin/pages/infrastructure.adoc#_keycloak_config_script (adapted for this lab)

**ATENÇÃO** Atualize a variável `RHBK_KCADM_PATH` com o valor correspondente em sua máquina / estação de trabalho

```
RHBK_KEYCLOAK_ADMIN_USERNAME=$(oc get secret keycloak-initial-admin -n tssc-rhbk -o jsonpath='{.data.username}' | base64 --decode)
RHBK_KEYCLOAK_ADMIN_PASSWORD=$(oc get secret keycloak-initial-admin -n tssc-rhbk -o jsonpath='{.data.password}' | base64 --decode)
RHBK_TPA_REALM_NAME=tssc-tpa
RHBK_HOSTNAME=$(echo https://tssc-rhbk.$CLUSTER_DOMAIN)
RHBK_KCADM_PATH=/path/to/kcadm.sh
#RHBK_KCADM_PATH=/home/gfilho/Documents/lab/rhbk-26.4.10/bin/kcadm.sh
TPA_NAMESPACE=tssc-tpa

./keycloak-setup.sh \
  --kcadm-path $RHBK_KCADM_PATH \
  --keycloak-url $RHBK_HOSTNAME \
  --keycloak-admin $RHBK_KEYCLOAK_ADMIN_USERNAME \
  --keycloak-admin-password $RHBK_KEYCLOAK_ADMIN_PASSWORD \
  --base-hostname $CLUSTER_DOMAIN \
  --namespace $TPA_NAMESPACE \
  --realm $RHBK_TPA_REALM_NAME \
  --init-data ./tpa_realm_setup
```

## Instalação do Red Hat Trusted Profile Analyzer Operator

### Instalar o Operator
```
envsubst < rhtpa\operator/1-namespace.yaml | oc apply -f -
envsubst < rhtpa\operator/2-operator-group.yaml | oc apply -f -
envsubst < rhtpa\operator/3-subscription.yaml | oc apply -f -
```

### Instalar o RHTPA e seus pre-reqs
```
# Deploy de um banco de dados postgresql (Pré-Requisito do RHTPA)
envsubst < rhtpa\instance/0-tssc-tpa-database.yaml | oc apply -f -

# Atualize com o client secret do client tssc-tpa-cli do realm tssc-tpa
envsubst < rhtpa\instance/1-tssc-tpa-oidc-cli-credentials.yaml | oc apply -f -

# Criação de um bucket no ODF
envsubst < rhtpa\instance/2-tssc-tpa-storage.yaml | oc apply -f -

# Define o HOSTNAME do Bucket S3 com o dns interno do odf 
# Se você tiver problemas com certificado auto-assinado, tente utilizar o DNS externo com o comando abaixo ou configurar a cadeia de confiança do RHTPA
# noobaa status -n openshift-storage |& grep 'Address\|DNS'
TPA_BUCKET_HOST=$(oc get configmap tssc-tpa-bucket -n trusted-profile-analyzer -o jsonpath='{.data.BUCKET_HOST'})
TPA_BUCKET_NAME=$(oc get configmap tssc-tpa-bucket -n trusted-profile-analyzer -o jsonpath='{.data.BUCKET_NAME'})
TPA_BUCKET_PORT=$(oc get configmap tssc-tpa-bucket -n trusted-profile-analyzer -o jsonpath='{.data.BUCKET_PORT'})
TPA_BUCKET_ACCESS_KEY=$(oc get secret tssc-tpa-bucket -n trusted-profile-analyzer -o go-template='{{index .data "AWS_ACCESS_KEY_ID"}}')
TPA_BUCKET_SECRET=$(oc get secret tssc-tpa-bucket -n trusted-profile-analyzer -o go-template='{{index .data "AWS_SECRET_ACCESS_KEY"}}')

# Cria a secret com as credenciais de acesso ao bucket s3
envsubst < rhtpa\instance/3-tssc-tpa-storage-credentials.yaml | oc apply -f -

# Cria as secrets com as credenciais de banco de dado
envsubst < rhtpa\instance/4-tssc-tpa-database-user-credentials.yaml | oc apply -f -
envsubst < rhtpa\instance/5-tssc-tpa-database-admin-credentials.yaml | oc apply -f -

# Cria a instância do TPA
envsubst < rhtpa\instance/7-tssc-tpa-instance.yaml | oc apply -f -
```

Valide se o client `tssc-tpa-cli` está com permissão para realizar o upload de SBOMs:

```
TPA_SERVER_URL="https://$(oc get route -l app.kubernetes.io/component=server -n trusted-profile-analyzer -o json | jq -r '.items[0].spec.host')"
RHBK_SERVER_URL="https://$(oc get route -n tssc-rhbk -o yaml -l app=keycloak -o json | jq -r '.items[0].spec.host')/realms/tssc-tpa/.well-known/openid-configuration" 

TOKEN_EP=$(curl "$RHBK_SERVER_URL" | jq -r .token_endpoint)
TOKEN=$(curl -s -X POST "$TOKEN_EP" \
  -d grant_type=client_credentials \
  -d client_id="tssc-tpa-cli" \
  -d client_secret="ATUALIZAR-SECRET" | jq -r .access_token)

curl -X POST ${TPA_SERVER_URL}/api/v3/sbom?id=test-sbom  -H "Authorization: Bearer ${TOKEN}" -H "Content-Type: application/json" --data-binary @./rhtpa/artifacts/test-sbom.json
```

> Não prossiga caso ocorram erros para enviar o arquivo SBOM de exemplo