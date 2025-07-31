
# Projeto AWS - Infraestrutura Web com Monitoramento

## ✅ Visão Geral

Este projeto consiste em criar uma infraestrutura básica na AWS, composta por uma VPC personalizada, instâncias EC2, servidor Nginx e um sistema de monitoramento com alertas via Webhook (Discord).

<img width="1440" height="900" alt="Captura de Tela 2025-07-30 às 23 31 54" src="https://github.com/user-attachments/assets/dc77ef0d-9daa-46af-97e4-639c09e6315e" />

---

## 📌 Etapa 1 – Configuração do Ambiente

### Objetivo:
Criar uma infraestrutura de rede para suportar um servidor web com monitoramento.

### Ações realizadas:

1. **Criação de uma VPC personalizada**
   - VPC com CIDR `10.0.0.0/16`
   - Nome fictício: `PB-VPC`

2. **Criação de Sub-redes**
   - 2 Sub-redes públicas: `PB-Public-1`, `PB-Public-2`
   - 2 Sub-redes privadas: `PB-Private-1`, `PB-Private-2`

3. **Internet Gateway**
   - Anexado à VPC para permitir acesso externo

4. **Route Tables**
   - Tabela de rotas pública associada às sub-redes públicas
   - Tabela de rotas privada associada às sub-redes privadas

5. **Instâncias EC2**
   - Criadas nas sub-redes públicas
   - Sistema operacional: **Ubuntu 22.04 LTS**
   - Nome fictício da instância pública: `Bastion Host`
   - Nome fictício da instância pública: `EC2-Privada`
   - Nome fictício da instância NAT: `NAT-Instance` (amazon linux)

---

## 🌐 Etapa 2 – Configuração do Servidor Web

### Objetivo:
Instalar e configurar um servidor web na EC2. (privada)

### Ações realizadas:

1. **Instalação do Nginx**
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install nginx -y
   ```

2. **Criação de página HTML**
   - Arquivo localizado em `/var/www/html/index.html` (dentro da ec2-privada)

```
<!DOCTYPE html>
<html>
<head>
    <title>Projeto Estágio - Servidor Nginx</title>
</head>
<body>
    <h1>Projeto de Estágio - Servidor Web Nginx na AWS</h1>
    <p>Esta página está hospedada em uma instância EC2 privada.</p>
    <p>Autor: Murilo Muller</p>
    <p>Data: 25/07/2025</p>
</body>
</html>
```

3. **Verificação**
   - Página acessível via navegador no IP público da EC2 (funciona em rede interna apenas)

---

## 📡 Etapa 3 – Script de Monitoramento + Webhook

### Objetivo:
Monitorar o status do site e enviar alerta se o servidor cair.

### Script em Bash (/usr/local/bin/monitoramento.sh):

```bash
#!/bin/bash

URL="http://10.0.3.91"
WEBHOOK_URL="https://discord.com/api/webhooks/1398397961997910056/3T727zPZLSw9qfoZJxaRh9JsGBwTQ_fqo6cFhpFHukgJ6T0yx0LzUjWE3GjpVrJu4CGt"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL")
DATA=$(date +"%d/%m/%Y %H:%M")

STATUS_FILE="/tmp/ultimo_status_site.txt"

if [ ! -f "$STATUS_FILE" ]; then
    echo "$STATUS" > "$STATUS_FILE"
fi

STATUS_ANTERIOR=$(cat "$STATUS_FILE")

if [ "$STATUS" != "$STATUS_ANTERIOR" ]; then
    echo "$STATUS" > "$STATUS_FILE"

    if [ "$STATUS" -eq 200 ]; then
        MSG="✅ SITE ONLINE - Código HTTP: $STATUS"
    else
        MSG="⚠️ SITE FORA DO AR - Código HTTP: $STATUS"
    fi

    curl -H "Content-Type: application/json"          -X POST          -d "{"content": "$MSG"}"          "$WEBHOOK_URL"

    echo "$DATA - Status mudou para $STATUS. Mensagem enviada ao Discord."
else
    echo "$DATA - Status $STATUS (sem mudança)."
fi
```

## Local e timer de monitoramento sobre o script de fato

### cat monitoramento.service 
```
[Unit]
Description=Serviço de Monitoramento do Site

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitoramento.sh
StandardOutput=journal
```

### cat monitoramento.timer 
```
[Unit]
Description=Timer para rodar monitoramento a cada 1 minuto

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
Unit=monitoramento.service

[Install]
WantedBy=timers.target
```



---

## 🧪 Etapa 4 – Testes e Documentação

- Acesso via navegador ao IP público
- Derrubado o serviço Nginx e testado alerta via webhook
- Verificado arquivo de log com status correto

---

## 🧩 Etapa 5 – (Desafio Bônus - Não Implementado)

Não foi implementada a injeção automática via User Data para provisionamento completo da instância EC2 com Nginx, página HTML e script.

---

## 🔐 Segurança e Dados Sensíveis

- Remoção de: ARNs de recursos, IDs de conta, Webhooks reais

---

## 🛠️ Infraestrutura via AWS CLI (detalhes da criação completa)

Conteúdo completo dos comandos utilizados: criação de VPC, sub-redes, route tables, security groups, instâncias EC2, NAT Instance e configurações avançadas.

# Detalhes sobre Código da Criação da Infra – Passo 1

## Criação de Proteção de Orçamento (não foi automatizado)

```bash
aws budgets create-budget   --account-id 304052674272   --budget '{
    "BudgetName": "ProtectBudget15",
    "BudgetLimit": {
      "Amount": "15.00",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "TimePeriod": {}
  }'
```

## ✅ Etapa 2 – Criar notificação com alerta de 100% via SNS

```bash
aws budgets create-notification   --account-id 304052674272   --budget-name ProtectBudget15   --notification '{
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 100,
    "ThresholdType": "PERCENTAGE",
    "NotificationState": "ALARM"
  }'   --subscribers '[
    {
      "SubscriptionType": "SNS",
      "Address": "arn:aws:sns:us-east-1:304052674272:StopServicesBudgetSNS"
    }
  ]'
```

---

## Comandos AWS Rápidos

```bash
Elastic IP associado: 54.156.90.91

# Acessos via SSH

# Bastion Host (IP público)
ssh -i PB-Key.pem ubuntu@54.156.90.91

# EC2 Privada
ssh -i PB-Key.pem ubuntu@10.0.3.91

# NAT Instance (via Bastion)
ssh -i PB-Key.pem ec2-user@10.0.2.33

# Aliases e atalhos criados no SSH
awslogin          # Renovar token SSO
ssh bastion       # Login no Bastion Host
ssh ec2-privada   # ProxyJump para EC2 Privada
ssh nat-instance  # ProxyJump para NAT Instance

# Comandos EC2 (start, stop, describe)
aws ec2 stop-instances --instance-ids i-0423faf1da0b9xxxx
aws ec2 start-instances --instance-ids i-0423faf1da0b9xxxx
aws ec2 describe-instances --instance-ids i-0423faf1da0b9xxxx

# CUIDADO: Excluir instância
aws ec2 terminate-instances --instance-ids i-xxxxxxxxx
```
---

## Criação da Infraestrutura

### 1.1 Criar VPC

```bash
aws ec2 create-vpc   --cidr-block 10.0.0.0/16   --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=PB-VPC}]'
```

### 1.2 Criar Internet Gateway

```bash
aws ec2 create-internet-gateway   --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=PB-IGW}]'
```

### 1.3 Criar Sub-redes

```bash
# Sub-redes Públicas
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Public-1}]'

aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Public-2}]'

# Sub-redes Privadas
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.3.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Private-1}]'

aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.4.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Private-2}]'
```

### 1.4 Criar e Associar Tabela de Rotas

```bash
aws ec2 create-route-table --vpc-id <VPC_ID> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PB-Public-RT}]'

aws ec2 create-route   --route-table-id <ROUTE_TABLE_ID>   --destination-cidr-block 0.0.0.0/0   --gateway-id <IGW_ID>
```

### 1.5 Associar Sub-redes Públicas à Tabela de Rotas

```bash
aws ec2 associate-route-table --subnet-id <SUBNET_ID_1> --route-table-id <ROUTE_TABLE_ID>
aws ec2 associate-route-table --subnet-id <SUBNET_ID_2> --route-table-id <ROUTE_TABLE_ID>
```



# Detalhes sobre Código da Criação da Infra – Passo 1

## Criação de Proteção de Orçamento (não foi automatizado)

```bash
aws budgets create-budget   --account-id 304052674272   --budget '{
    "BudgetName": "ProtectBudget15",
    "BudgetLimit": {
      "Amount": "15.00",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "TimePeriod": {}
  }'
```

## ✅ Etapa 2 – Criar notificação com alerta de 100% via SNS

```bash
aws budgets create-notification   --account-id 304052674272   --budget-name ProtectBudget15   --notification '{
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 100,
    "ThresholdType": "PERCENTAGE",
    "NotificationState": "ALARM"
  }'   --subscribers '[
    {
      "SubscriptionType": "SNS",
      "Address": "arn:aws:sns:us-east-1:304052674272:StopServicesBudgetSNS"
    }
  ]'
```

---

## Comandos AWS Rápidos

```bash
Elastic IP associado: 54.156.90.91

# Acessos via SSH
ssh -i PB-Key.pem ubuntu@54.156.90.91      # Bastion Host (IP público)
ssh -i PB-Key.pem ubuntu@10.0.3.91         # EC2 Privada
ssh -i PB-Key.pem ec2-user@10.0.2.33       # NAT Instance (via Bastion)

# Aliases e atalhos criados no SSH
awslogin               # Renovar token SSO
ssh-bastion            # Login no Bastion Host
ssh-ec2-privada        # ProxyJump para EC2 Privada
ssh-nat-instance       # ProxyJump para NAT Instance

# Comandos EC2 (start, stop, describe)
aws ec2 stop-instances --instance-ids i-0423faf1da0b99905
aws ec2 start-instances --instance-ids i-0423faf1da0b99905
aws ec2 describe-instances --instance-ids i-0423faf1da0b99905

aws ec2 stop-instances --instance-ids i-029654cd64e8edcc4
aws ec2 start-instances --instance-ids i-029654cd64e8edcc4
aws ec2 describe-instances --instance-ids i-029654cd64e8edcc4

aws ec2 stop-instances --instance-ids i-0f7bea244e1ec1237
aws ec2 start-instances --instance-ids i-0f7bea244e1ec1237
aws ec2 describe-instances --instance-ids i-0f7bea244e1ec1237

# CUIDADO: Excluir instância
aws ec2 terminate-instances --instance-ids i-xxxxxxxxx
```

---

## Criação da Infraestrutura

### 1.1 Criar VPC

```bash
aws ec2 create-vpc   --cidr-block 10.0.0.0/16   --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=PB-VPC}]'
```

### 1.2 Criar Internet Gateway

```bash
aws ec2 create-internet-gateway   --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=PB-IGW}]'
```

### 1.3 Criar Sub-redes

```bash
# Sub-redes Públicas
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Public-1}]'
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Public-2}]'

# Sub-redes Privadas
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.3.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Private-1}]'
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.4.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PB-Private-2}]'
```

### 1.4 Criar e Associar Tabela de Rotas

```bash
aws ec2 create-route-table --vpc-id <VPC_ID> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PB-Public-RT}]'

aws ec2 create-route   --route-table-id <ROUTE_TABLE_ID>   --destination-cidr-block 0.0.0.0/0   --gateway-id <IGW_ID>
```

### 1.5 Associar Sub-redes Públicas à Tabela de Rotas

```bash
aws ec2 associate-route-table --subnet-id <SUBNET_ID_1> --route-table-id <ROUTE_TABLE_ID>
aws ec2 associate-route-table --subnet-id <SUBNET_ID_2> --route-table-id <ROUTE_TABLE_ID>
```
---

## 2. EC2 e NAT

### Criar Instância NAT

```bash
aws ec2 run-instances   --image-id ami-0c55b159cbfafe1f0 \  # Amazon Linux 2 AMI
  --count 1   --instance-type t3.micro   --key-name PB-Key   --subnet-id <SUBNET_ID_PUBLICA>   --associate-public-ip-address   --security-group-ids <SG_ID>   --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=PB-NAT-Instance}]'
```

### Criar Bastion Host

```bash
aws ec2 run-instances   --image-id ami-0c55b159cbfafe1f0   --count 1   --instance-type t3.micro   --key-name PB-Key   --subnet-id <SUBNET_ID_PUBLICA>   --associate-public-ip-address   --security-group-ids <SG_ID>   --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=8,DeleteOnTermination=true,VolumeType=gp2}' \
	--tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=PB - JUL 2025},{Key=CostCenter,Value=CO92000024},{Key=Project,Value=PB - JUL 2025}]' \
    'ResourceType=volume,Tags=[{Key=Name,Value=PB - JUL 2025},{Key=CostCenter,Value=CO92000024},{Key=Project,Value=PB - JUL 2025}]'

```

### Depois crie e associe o Elastic IP:

aws ec2 allocate-address

Elastic IP criado: 54.156.90.91

aws ec2 associate-address \
	--instance-id i-0423faf1da0b99905 \
	--allocation-id eipalloc-0ec8a1511abe2a3fb

### Autorizar as portas 22 e 80 no Security Group

aws ec2 authorize-security-group-ingress --group-id sg-0416d21d85b943018 \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-id sg-0416d21d85b943018 \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

### Instância Privada com Nginx

```bash
aws ec2 run-instances \
	--image-id ami-09ac0b140f63d3458 \
	--count 1 \
	--instance-type t3.micro \
	--key-name PB-Key \
	--security-group-ids sg-0416d21d85b943018 \
	--subnet-id subnet-066c7b47da349b9ad \
	--associate-public-ip-address \
	--block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=8,DeleteOnTermination=true,VolumeType=gp2}' \
	--tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=PB - JUL 2025},{Key=CostCenter,Value=CO92000024},{Key=Project,Value=PB - JUL 2025}]' \
    'ResourceType=volume,Tags=[{Key=Name,Value=PB - JUL 2025},{Key=CostCenter,Value=CO92000024},{Key=Project,Value=PB - JUL 2025}]'

```

---

## 3. Regras de Segurança

### Criar Grupos de Segurança

```bash
aws ec2 create-security-group   --group-name PB-SG-Bastion   --description "Permitir SSH do meu IP"   --vpc-id <VPC_ID>
```

```bash
aws ec2 authorize-security-group-ingress   --group-id <SG_ID>   --protocol tcp   --port 22   --cidr <SEU_IP>/32
```

### Regras para Web Server

```bash
aws ec2 create-security-group   --group-name PB-SG-Web   --description "Permitir tráfego HTTP interno"   --vpc-id <VPC_ID>
```

```bash
aws ec2 authorize-security-group-ingress   --group-id <SG_ID_WEB>   --protocol tcp   --port 80   --source-group <SG_ID_NAT>
```

---

## 4. Serviços de Monitoramento (CloudWatch e Logs)

### Visualizar logs via AWS CLI

```bash
aws logs describe-log-groups
aws logs describe-log-streams --log-group-name "/var/log/nginx/access.log"
aws logs get-log-events   --log-group-name "/var/log/nginx/access.log"   --log-stream-name "<NOME_DO_STREAM>"
```

---

## 5. Anexos e Alias (Exemplo)

### ~/.ssh/config (proxy jump)

```sshconfig
Host bastion
  HostName 54.156.90.91
  User ubuntu
  IdentityFile ~/PB-Key.pem

Host ec2-privada
  HostName 10.0.3.91
  User ubuntu
  IdentityFile ~/PB-Key.pem
  ProxyJump bastion

Host nat-instance
  HostName 10.0.2.33
  User ec2-user
  IdentityFile ~/PB-Key.pem
  ProxyJump bastion
```

---

**FIM DO PROJETO - INFRAESTRUTURA AWS DOCUMENTADA** ✅
