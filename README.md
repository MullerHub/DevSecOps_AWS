# Projeto AWS - Infraestrutura Básica

## Parte 1 - VPC e Sub-redes

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=VPC-Projeto}]'

# Sub-rede pública
aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.0.1.0/24 --availability-zone sa-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Publica-A}]'

# Sub-rede privada
aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.0.2.0/24 --availability-zone sa-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Privada-A}]'

# Internet Gateway
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Projeto-IGW}]'
aws ec2 attach-internet-gateway --internet-gateway-id <IGW-ID> --vpc-id <VPC-ID>

# Tabela de rota pública
aws ec2 create-route-table --vpc-id <VPC-ID> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Rota-Publica}]'
aws ec2 create-route --route-table-id <ROUTE-TABLE-ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW-ID>
aws ec2 associate-route-table --route-table-id <ROUTE-TABLE-ID> --subnet-id <SUBNET-ID-PUBLICA>
```

## Parte 2 - NAT Instance

```bash
# EC2 NAT configurada manualmente:
# Habilitar forwarding
sudo vim /etc/sysctl.conf
# Adicionar:
net.ipv4.ip_forward = 1
sudo sysctl -p

# NAT via iptables
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Persistir com rc.local
sudo vim /etc/rc.local
# Conteúdo:
#!/bin/bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
exit 0
sudo chmod +x /etc/rc.local

# Desabilitar Source/Dest Check
aws ec2 modify-instance-attribute --instance-id <ID-DA-EC2-NAT> --no-source-dest-check

# Tabela de rota privada
aws ec2 create-route-table --vpc-id <VPC-ID> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Rota-Privada}]'
aws ec2 create-route --route-table-id <ROUTE-TABLE-ID-PRIVADA> --destination-cidr-block 0.0.0.0/0 --instance-id <ID-DA-EC2-NAT>
aws ec2 associate-route-table --route-table-id <ROUTE-TABLE-ID-PRIVADA> --subnet-id <SUBNET-ID-PRIVADA>
```

## Parte 3 - Bastion Host e Instâncias

```bash
# Criar par de chaves
aws ec2 create-key-pair --key-name ProjetoKey --query 'KeyMaterial' --output text > ProjetoKey.pem
chmod 400 ProjetoKey.pem

# Security group do Bastion
aws ec2 create-security-group --group-name bastion-sg --description "SG do bastion" --vpc-id <VPC-ID>
aws ec2 authorize-security-group-ingress --group-id <SG-ID-BASTION> --protocol tcp --port 22 --cidr 0.0.0.0/0

# Instância Bastion (sub-rede pública)
aws ec2 run-instances \
--image-id ami-<ubuntu> \
--count 1 \
--instance-type t2.micro \
--key-name ProjetoKey \
--security-group-ids <SG-ID-BASTION> \
--subnet-id <SUBNET-ID-PUBLICA> \
--associate-public-ip-address \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Bastion}]'

# Instância Privada (sem IP público)
aws ec2 run-instances \
--image-id ami-<ubuntu> \
--count 1 \
--instance-type t2.micro \
--key-name ProjetoKey \
--security-group-ids <SG-ID-PRIVATE> \
--subnet-id <SUBNET-ID-PRIVADA> \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Privada}]'
```

## Parte 4 - Acesso via Bastion + Nginx

```bash
# Conectar ao Bastion
ssh -i ProjetoKey.pem ubuntu@<IP-BASTION>

# Do Bastion para a instância privada
ssh -i ProjetoKey.pem ubuntu@<IP-PRIVADA>

# Configuração Nginx (dentro da privada)
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

## Parte 5 - AWS Budgets + SNS

```bash
# Criar tópico SNS
aws sns create-topic --name AlertaOrcamento

# Assinar e-mail (confirmar depois)
aws sns subscribe --topic-arn <TOPICO-ARN> --protocol email --notification-endpoint seu@email.com

# Criar orçamento
aws budgets create-budget --account-id <ACCOUNT-ID> --budget file://orcamento.json

# Exemplo de orcamento.json:
{
  "Budget": {
    "BudgetName": "BudgetAlerta",
    "BudgetLimit": {
      "Amount": "0.01",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {},
    "CostTypes": {
      "IncludeTax": true,
      "IncludeSubscription": true,
      "UseBlended": false,
      "IncludeRefund": true,
      "IncludeCredit": true,
      "IncludeUpfront": true,
      "IncludeRecurring": true,
      "IncludeOtherSubscription": true,
      "IncludeSupport": true,
      "IncludeDiscount": true,
      "UseAmortized": false
    },
    "TimePeriod": {
      "Start": "2025-07-01_00:00",
      "End": "2087-12-31_00:00"
    }
  },
  "NotificationsWithSubscribers": [
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 0.01,
        "ThresholdType": "ABSOLUTE_VALUE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "SNS",
          "Address": "<TOPICO-ARN>"
        }
      ]
    }
  ]
}
```


