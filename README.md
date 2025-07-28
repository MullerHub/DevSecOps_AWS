
# Projeto AWS - Infraestrutura Web com Monitoramento

## ✅ Visão Geral

Este projeto consiste em criar uma infraestrutura básica na AWS, composta por uma VPC personalizada, instâncias EC2, servidor Nginx e um sistema de monitoramento com alertas via Webhook (Telegram, Slack ou Discord).

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
   - Arquivo localizado em `/var/www/html/index.html`
   - Conteúdo exemplo:
     ```html
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

### Script em Bash (exemplo):

```bash
#!/bin/bash

URL="http://10.0.3.91"
WEBHOOK_URL="https://discord.com/api/webhooks/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
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


