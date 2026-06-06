# 🧾 Tarefas Automatizadas com Lambda Function e S3

> Laboratório prático da [DIO](https://dio.me) — Automatizando o processamento de Notas Fiscais com AWS Lambda, S3 e DynamoDB utilizando LocalStack.

---

## 📋 Índice

- [Sobre o Projeto](#-sobre-o-projeto)
- [Arquitetura](#-arquitetura)
- [Pré-requisitos](#-pré-requisitos)
- [Configuração do Ambiente](#-configuração-do-ambiente)
- [Passo a Passo](#-passo-a-passo)
  - [1. Criar o Bucket S3](#1-criar-o-bucket-s3)
  - [2. Criar a Tabela no DynamoDB](#2-criar-a-tabela-no-dynamodb)
  - [3. Criar a Função Lambda](#3-criar-a-função-lambda)
  - [4. Configurar o Trigger do S3](#4-configurar-o-trigger-do-s3)
  - [5. Criar a API Gateway](#5-criar-a-api-gateway)
  - [6. Testar o Fluxo Completo](#6-testar-o-fluxo-completo)
- [Insights e Aprendizados](#-insights-e-aprendizados)
- [Recursos Úteis](#-recursos-úteis)

---

## 📌 Sobre o Projeto

Este laboratório implementa um pipeline serverless para **processamento automático de notas fiscais**. Quando um arquivo JSON é enviado ao bucket S3, uma função Lambda é disparada automaticamente para ler o arquivo e persistir os dados no DynamoDB. Uma API REST via API Gateway também é exposta para inserção e consulta dos registros.

Todo o ambiente é simulado localmente usando **LocalStack**, eliminando a necessidade de uma conta AWS ativa durante o desenvolvimento.

### Recursos AWS utilizados

| Serviço | Função |
|---|---|
| **S3** | Armazenamento dos arquivos de notas fiscais |
| **Lambda** | Processamento automático ao receber novos arquivos |
| **DynamoDB** | Banco de dados NoSQL para persistência das notas |
| **API Gateway** | Exposição de endpoints REST (GET e POST) |

---

## 🏗 Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                      LocalStack                          │
│                                                         │
│   Upload JSON        Trigger          Grava registro    │
│  ┌──────────┐  ───►  ┌──────────┐  ───►  ┌──────────┐  │
│  │    S3    │        │  Lambda  │         │ DynamoDB │  │
│  │  Bucket  │        │ Function │         │  Table   │  │
│  └──────────┘        └──────────┘         └──────────┘  │
│                           ▲                    ▲        │
│                           │                    │        │
│                      ┌──────────┐               │        │
│                      │   API    │───────────────┘        │
│                      │ Gateway  │  POST / GET /notas     │
│                      └──────────┘                        │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ Pré-requisitos

- [Docker](https://www.docker.com/) instalado e em execução
- [LocalStack](https://docs.localstack.cloud/getting-started/installation/) (CLI ou via Docker)
- [AWS CLI](https://aws.amazon.com/cli/) configurado
- Python 3.9+

---

## ⚙️ Configuração do Ambiente

### Iniciando o LocalStack via Docker

```bash
docker run -d \
  --name localstack \
  -p 4566:4566 \
  -p 4571:4571 \
  -e SERVICES=ALL \
  -e DEBUG=1 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack
```

### Alternativa: LocalStack CLI

```bash
# Instalar via pip
pip install localstack

# Iniciar
localstack start

# Verificar status
localstack --version
```

### Configurar variáveis de ambiente do AWS CLI

> ⚠️ As credenciais **não precisam ser válidas** para o LocalStack, mas devem estar definidas.

**Linux/macOS:**
```bash
export AWS_ACCESS_KEY_ID="test"
export AWS_SECRET_ACCESS_KEY="test"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_DEFAULT_OUTPUT="json"
```

**Windows (PowerShell):**
```powershell
$env:AWS_ACCESS_KEY_ID="test"
$env:AWS_SECRET_ACCESS_KEY="test"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:AWS_DEFAULT_OUTPUT="json"
```

**Ou via `aws configure`:**
```
AWS Access Key ID: test
AWS Secret Access Key: test
Default region name: us-east-1
Default output format: json
```

### Verificar saúde do LocalStack

```powershell
Invoke-RestMethod -Uri "http://localhost:4566/_localstack/health"
```

---

## 🚀 Passo a Passo

### 1. Criar o Bucket S3

```bash
awslocal s3api create-bucket --bucket notas-fiscais-upload
```

**Verificar:**
```bash
awslocal s3 ls
```

---

### 2. Criar a Tabela no DynamoDB

```bash
aws dynamodb create-table \
  --endpoint-url=http://localhost:4566 \
  --table-name NotasFiscais \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

**Verificar:**
```bash
aws dynamodb list-tables --endpoint-url=http://localhost:4566
```

> 💡 Use o [NoSQL Workbench for DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.settingup.html) para visualizar e consultar os dados de forma gráfica.

---

### 3. Criar a Função Lambda

Primeiro, empacote o código Python em um arquivo `.zip`:

```bash
zip lambda_function.zip grava_db.py
```

Depois, crie a função no LocalStack:

```bash
aws lambda create-function \
  --function-name ProcessarNotasFiscais \
  --runtime python3.9 \
  --role arn:aws:iam::000000000000:role/lambda-role \
  --handler grava_db.lambda_handler \
  --zip-file fileb://lambda_function.zip \
  --endpoint-url=http://localhost:4566
```

**Verificar:**
```bash
aws lambda list-functions --endpoint-url=http://localhost:4566
```

> 🔑 A role `arn:aws:iam::000000000000:role/lambda-role` é fictícia e aceita pelo LocalStack sem validação real de IAM.

---

### 4. Configurar o Trigger do S3

**Conceder permissão ao S3 para invocar a Lambda:**

```bash
aws lambda add-permission \
  --function-name ProcessarNotasFiscais \
  --statement-id s3-trigger-permission \
  --action "lambda:InvokeFunction" \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::notas-fiscais-upload" \
  --endpoint-url=http://localhost:4566
```

**Configurar a notificação no bucket S3** (arquivo `notification_roles.json`):

```json
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}
```

```bash
aws s3api put-bucket-notification-configuration \
  --bucket notas-fiscais-upload \
  --notification-configuration file://notification_roles.json \
  --endpoint-url=http://localhost:4566
```

**Validar a configuração:**

```bash
aws s3api get-bucket-notification-configuration \
  --bucket notas-fiscais-upload \
  --endpoint-url=http://localhost:4566
```

**Gerar e enviar arquivo de teste:**

```bash
# Gerar dados fake
python gerar_dados.py

# Upload para o S3 (dispara a Lambda automaticamente!)
aws s3 cp notas_fiscais_2025.json s3://notas-fiscais-upload \
  --endpoint-url=http://localhost:4566
```

---

### 5. Criar a API Gateway

**Criar a REST API:**

```bash
aws apigateway create-rest-api \
  --name "NotasFiscaisAPI" \
  --endpoint-url=http://localhost:4566
```

**Obter o ID da API e do recurso raiz:**

```bash
aws apigateway get-resources \
  --rest-api-id <API_ID> \
  --endpoint-url=http://localhost:4566
```

**Criar o recurso `/notas`:**

```bash
aws apigateway create-resource \
  --rest-api-id <API_ID> \
  --parent-id <ROOT_RESOURCE_ID> \
  --path-part "notas" \
  --endpoint-url=http://localhost:4566
```

**Configurar métodos POST e GET:**

```bash
# Método POST
aws apigateway put-method \
  --rest-api-id <API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method POST \
  --authorization-type "NONE" \
  --endpoint-url=http://localhost:4566

# Método GET
aws apigateway put-method \
  --rest-api-id <API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method GET \
  --authorization-type "NONE" \
  --endpoint-url=http://localhost:4566
```

**Integrar com a Lambda:**

```bash
# Integração POST
aws apigateway put-integration \
  --rest-api-id <API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method POST \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" \
  --endpoint-url=http://localhost:4566

# Integração GET
aws apigateway put-integration \
  --rest-api-id <API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" \
  --endpoint-url=http://localhost:4566
```

**Conceder permissões e fazer deploy:**

```bash
# Permissão POST
aws lambda add-permission \
  --function-name ProcessarNotasFiscais \
  --statement-id apigateway-access \
  --action "lambda:InvokeFunction" \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:000000000000:<API_ID>/*/POST/notas" \
  --endpoint-url=http://localhost:4566

# Permissão GET
aws lambda add-permission \
  --function-name ProcessarNotasFiscais \
  --statement-id apigateway-access-get \
  --action "lambda:InvokeFunction" \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:000000000000:<API_ID>/*/GET/notas" \
  --endpoint-url=http://localhost:4566

# Deploy
aws apigateway create-deployment \
  --rest-api-id <API_ID> \
  --stage-name dev \
  --endpoint-url=http://localhost:4566
```

---

### 6. Testar o Fluxo Completo

**Via PowerShell (POST):**

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:4566/restapis/<API_ID>/dev/_user_request_/notas" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{"id": "NF-999", "cliente": "João Silva", "valor": 1000.0, "data_emissao": "2025-01-31"}'
```

**Via PowerShell (GET):**

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:4566/restapis/<API_ID>/dev/_user_request_/notas" `
  -Method GET
```

**Via curl (Linux/macOS):**

```bash
curl -X POST http://localhost:4566/restapis/<API_ID>/dev/_user_request_/notas \
  -H "Content-Type: application/json" \
  -d '{"id": "NF-001", "cliente": "Maria Souza", "valor": 500.0, "data_emissao": "2025-06-01"}'
```

**Verificar logs do LocalStack:**

```bash
docker logs localstack
```

---

## 💡 Insights e Aprendizados

### O que é LocalStack?
LocalStack é uma ferramenta que simula serviços AWS localmente, permitindo desenvolvimento e testes sem custo e sem precisar de uma conta AWS real. É ideal para ambientes de desenvolvimento e CI/CD.

### Event-Driven Architecture
A integração S3 → Lambda é um exemplo claro de arquitetura orientada a eventos. O upload de um arquivo é o **evento** que dispara o processamento automaticamente, sem polling ou intervenção manual.

### IDs Dinâmicos na API Gateway
Um ponto importante é que o LocalStack gera IDs aleatórios para cada recurso criado (API ID, Resource ID). Por isso, é necessário consultar esses IDs com `get-resources` antes de usá-los nos próximos comandos.

### Role fictícia é suficiente no LocalStack
O LocalStack não valida permissões IAM por padrão, então `arn:aws:iam::000000000000:role/lambda-role` funciona sem que a role exista de fato. Em produção na AWS real, a role precisa existir com as políticas corretas.

### Integração AWS_PROXY
Ao usar `--type AWS_PROXY` na API Gateway, o evento completo da requisição HTTP é passado diretamente para a Lambda como objeto `event`. A função precisa retornar um objeto com `statusCode`, `headers` e `body`.

---

## 📚 Recursos Úteis

- [Documentação do LocalStack](https://docs.localstack.cloud/)
- [LocalStack Desktop](https://docs.localstack.cloud/user-guide/tools/localstack-desktop/)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Amazon S3 Object Lambda com CloudFormation](https://docs.aws.amazon.com/pt_br/AmazonS3/latest/userguide/olap-using-cfn-template.html)
- [NoSQL Workbench for DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.settingup.html)
- [AWS CLI Command Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)

---

## 🖼️ Screenshots

> Veja a pasta [`/images`](./images) para capturas de tela do ambiente configurado no LocalStack Desktop, incluindo:
> - Stack Overview com DynamoDB, S3 e Lambda provisionados
> - Tabela `NotasFiscais` no DynamoDB após inserção de dados
> - Bucket `notas-fiscais-upload` no S3

---

*Desenvolvido como parte do bootcamp na [DIO - Digital Innovation One](https://dio.me) 🚀*
