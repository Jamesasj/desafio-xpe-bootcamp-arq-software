# 🛠️ Provisionamento de Ambiente no Google Cloud Platform (GCP)
Este guia cria um ambiente completo no GCP com:

- Múltiplas VMs em zonas distintas com escalonamento automático

- Balanceador de carga HTTP

- Banco de dados gerenciado (Cloud SQL)

- Segurança via firewall, IAM e monitoramento


![image](https://github.com/user-attachments/assets/45f479dd-218d-4dd3-9b9a-9791636bced8)



### 1️⃣ Criação e Configuração das VMs
1.1 Criar modelo de instância
```bash
gcloud compute instance-templates create web-template \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --tags=http-server \
  --boot-disk-size=10GB
```

1.2 Criar grupo de instâncias gerenciado (regional)

```bash
gcloud compute instance-groups managed create web-group \
  --base-instance-name=web-vm \
  --template=web-template \
  --size=3 \
  --region=us-central1 \
  --zones=us-central1-a,us-central1-b
```

1.3 Criar regra de firewall para liberar acesso HTTP apenas para seu IP

```bash
MY_IP=$(curl -s ifconfig.me)
gcloud compute firewall-rules create allow-http-from-my-ip \
  --allow=tcp:80 \
  --source-ranges=$MY_IP/32 \
  --target-tags=http-server \
  --description="Permitir HTTP apenas do IP atual"
```

1.4 Criar health check

```bash
gcloud compute health-checks create http http-basic-check \
  --port 80 \
  --request-path="/" \
  --check-interval=30s \
  --timeout=10s \
  --unhealthy-threshold=3 \
  --healthy-threshold=2
```

1.5 Criar backend service

``` bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

1.6 Adicionar grupo de instâncias ao backend

```bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=web-group \
  --instance-group-region=us-central1 \
  --global
```

1.7 Criar mapa de URL

```bash
gcloud compute url-maps create web-map-http \
  --default-service=web-backend-service
```

1.8 Criar proxy HTTP

```bash
gcloud compute target-http-proxies create http-lb-proxy \
  --url-map=web-map-http
```

1.9 Criar endereço IP global

```bash
gcloud compute addresses create http-lb-ipv4 \
  --ip-version=IPV4 \
  --global
```

1.10 Criar regra de encaminhamento

```bash
gcloud compute forwarding-rules create http-content-rule \
  --address=http-lb-ipv4 \
  --global \
  --target-http-proxy=http-lb-proxy \
  --ports=80
```
1.11 Configurar escalonamento automático

```bash
gcloud compute instance-groups managed set-autoscaling web-group \
  --max-num-replicas=6 \
  --min-num-replicas=3 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=60 \
  --region=us-central1
```

### 2️⃣ Banco de Dados (Cloud SQL)
2.1 Criar instância do Cloud SQL

```bash
gcloud sql instances create web-db \
  --database-version=POSTGRES_14 \
  --cpu=2 \
  --memory=4GB \
  --region=us-central1 \
  --root-password=senha_segura
```

2.2 Habilitar backups automáticos e replicação

```bash
gcloud sql instances patch web-db \
  --backup-start-time=03:00 \
  --enable-bin-log \
  --replica-type=READ_REPLICA
  ```

⚠️ Para multi-região, seria necessário criar réplicas posteriormente com gcloud sql instances create --master-instance-name.

2.3 Criar banco e usuário
```bash
gcloud sql databases create appdb --instance=web-db
gcloud sql users create appuser --instance=web-db --password=senha123
```

3️⃣ Segurança e Monitoramento
3.1 Conceder permissão IAM para acesso ao banco

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role=roles/cloudsql.client
Substitua PROJECT_ID e PROJECT_NUMBER pelos valores reais.
```

3.2 Habilitar conexão com banco via Cloud SQL Auth Proxy (ex: na VM)
Instale e configure o Cloud SQL Auth Proxy, ou use a conexão nativa com IP privado.

3.3 Ativar Cloud Monitoring

```bash
gcloud services enable monitoring.googleapis.com logging.googleapis.com
Acesse o Monitoring via Console: Monitoring → Metrics Explorer
```

✅ Conclusão
Com esse ambiente provisionado, você terá:

Infraestrutura escalável com balanceamento de carga

Banco de dados gerenciado, replicado e com backups

Monitoramento e segurança implementados via IAM, firewall e logs
