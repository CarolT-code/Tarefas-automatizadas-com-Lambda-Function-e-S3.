# Desafio - Tarefas Automatizadas com AWS Lambda, Amazon S3 e DynamoDB  
Este repositório reúne anotações, explicações e comandos utilizados durante o estudo de automação de tarefas usando AWS Lambda, Amazon S3, DynamoDB e API Gateway, incluindo a simulação local com LocalStack.

O objetivo é fornecer um guia completo para estudantes e desenvolvedores que desejam aprender a construir pipelines serverless capazes de processar arquivos automaticamente, registrar dados estruturados e expor informações via API.

---

# 1. Amazon S3

Amazon S3 é um serviço de armazenamento de objetos altamente escalável e durável. Ele permite armazenar qualquer tipo de arquivo, como imagens, vídeos, documentos, logs e backups.

## 1.1 Características e Vantagens

- Durabilidade: replicação automática garantindo proteção contra falhas.  
- Disponibilidade: acesso contínuo aos dados.  
- Escalabilidade: aumenta a capacidade conforme necessário, sem configuração adicional.  
- Segurança: criptografia, controle de acesso (IAM e Bucket Policies) e monitoramento com CloudTrail.

---

# 2. AWS Lambda

AWS Lambda é um serviço serverless que executa código em resposta a eventos sem necessidade de servidores.  
É ideal para automação, integração de sistemas e processamento de dados.

## 2.1 Características

- Execução por evento (S3, DynamoDB, SNS, API Gateway, SQS, etc).  
- Tempo limite de execução padrão de até 15 minutos.  
- Escalabilidade automática baseada na demanda.  
- Custo baseado no número de execuções e tempo de processamento.  

## 2.2 Principais Vantagens

- Execução sob demanda.  
- Integração nativa com serviços AWS.  
- Baixo custo.  
- Zero manutenção de servidores.

---

# 3. Arquitetura do Projeto: Upload, Processamento e Registro no DynamoDB

Este fluxo representa uma arquitetura serverless comum:

1. O usuário faz upload de um arquivo (CSV ou JSON) em um bucket S3.  
2. O evento `s3:ObjectCreated` dispara uma função Lambda.  
3. A Lambda processa o conteúdo, extrai informações e grava na tabela DynamoDB.  
4. Outra Lambda (opcional) consulta o banco e expõe esses dados através de API Gateway.

Este padrão é ideal para pipelines de ingestão de dados, ETL simples, automação de relatórios e validação de arquivos.

---

# 4. Trabalhando Localmente com LocalStack

LocalStack é um ambiente que simula serviços AWS localmente.  
É amplamente utilizado para desenvolvimento e testes sem custos, permitindo execução de pipelines serverless completos.

## 4.1 O que é o LocalStack

- Projeto open-source lançado em 2016 pela Atlantis Software, somente em 2017 tornou-se público.  
- Simula serviços como Lambda, S3, DynamoDB, SQS, SNS, API Gateway, CloudFormation e outros.  
- Possui versão gratuita e versões pagas com recursos avançados.

## 4.2 Objetivo

Permitir que desenvolvedores criem, testem e validem infraestruturas serverless e integrações AWS **sem acessar recursos reais**, ideal para ambientes de CI/CD.

---

# 5. Configuração do Ambiente Local

Antes de começar:

- Instale Python  
- Instale AWS CLI  
- Instale e execute LocalStack  
- Configure credenciais falsas no terminal:

```powershell
$env:AWS_ACCESS_KEY_ID="test"
$env:AWS_SECRET_ACCESS_KEY="test"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:AWS_DEFAULT_OUTPUT="json"
```

# 6. Criando a arquiteruta 

## 6.1 Criar o Bucket S3

```
awslocal s3api create-bucket --bucket notas-fiscais-upload
```
Verifique:

```
aws s3api get-bucket-notification-configuration \
--bucket notas-fiscais-upload \
--endpoint-url=http://localhost:4566

```

## 6.2 Criar a Tabela DynamoDB
```
aws dynamodb create-table \
  --endpoint-url=http://localhost:4566 \
  --table-name NotasFiscais \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

Verificar tabelas:
```
aws dynamodb list-tables --endpoint-url=http://localhost:4566
```
## 6.3 Criar a Função Lambda

Empacote o código em um arquivo:
```
zip lambda_function.zip grava_db.py
```

Criar a Lambda:
```
aws lambda create-function \
  --function-name ProcessarNotasFiscais \
  --runtime python3.9 \
  --role arn:aws:iam::000000000000:role/lambda-role \
  --handler grava_db.lambda_handler \
  --zip-file fileb://lambda_function.zip \
  --endpoint-url=http://localhost:4566
```
Verificar função criada:
```
aws lambda list-functions --endpoint-url=http://localhost:4566
```
## 6.4 Conceder Permissão ao S3 para Invocar a Lambda
```
aws lambda add-permission \
  --function-name ProcessarNotasFiscais \
  --statement-id s3-trigger-permission \
  --action "lambda:InvokeFunction" \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::notas-fiscais-upload" \
  --endpoint-url=http://localhost:4566
```
## 6.5 Criar Trigger no Bucket S3

Arquivo notification_roles.json:
```
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}

```
Enviar configuração:
```
aws s3api put-bucket-notification-configuration \
  --bucket notas-fiscais-upload \
  --notification-configuration file://notification_roles.json \
  --endpoint-url=http://localhost:4566
```
## 6.6 Testar o Processamento

Gerar arquivo:
```
python gerar_dados.py

```
Enviar arquivo:
```
aws s3 cp notas_fiscais_2025.json s3://notas-fiscais-upload \
--endpoint-url=http://localhost:4566
```

Consultar logs:
```
docker logs localstack
```
# 7. Criando e Integrando a API no API Gateway
## 7.1 Criar API
```
aws apigateway create-rest-api \
  --name "NotasFiscaisAPI" \
  --endpoint-url=http://localhost:4566
```
## 7.2 Obter ID do Recurso Raiz
```
aws apigateway get-resources \
  --rest-api-id <id> \
  --endpoint-url=http://localhost:4566
```
## 7.3 Criar Recurso /notas
```
aws apigateway create-resource \
  --rest-api-id <id> \
  --parent-id <root-id> \
  --path-part "notas" \
  --endpoint-url=http://localhost:4566
```
## 7.4 Adicionar Métodos GET e POST
```
aws apigateway put-method \
  --rest-api-id <id> \
  --resource-id <resource-id> \
  --http-method POST \
  --authorization-type "NONE" \
  --endpoint-url=http://localhost:4566
```

```
aws apigateway put-method \
  --rest-api-id <id> \
  --resource-id <resource-id> \
  --http-method GET \
  --authorization-type "NONE" \
  --endpoint-url=http://localhost:4566
```
## 7.5 Integrar com Lambda

POST:
```
aws apigateway put-integration \
  --rest-api-id <id> \
  --resource-id <resource-id> \
  --http-method POST \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" \
  --endpoint-url=http://localhost:4566
```

GET:

```
aws apigateway put-integration \
  --rest-api-id <id> \
  --resource-id <resource-id> \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" \
  --endpoint-url=http://localhost:4566
```
## 7.6 Conceder Permissão para API Invocar a Lambda
```
aws lambda add-permission \
  --function-name ProcessarNotasFiscais \
  --statement-id apigateway-access \
  --action "lambda:InvokeFunction" \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:000000000000:<id>/*/POST/notas" \
  --endpoint-url=http://localhost:4566
```
## 7.7 Deployment da API
```
aws apigateway create-deployment \
  --rest-api-id <id> \
  --stage-name dev \
  --endpoint-url=http://localhost:4566
```
## 7.8 Testar a API

POST:
```
Invoke-RestMethod -Uri "http://localhost:4566/restapis/<id>/dev/_user_request_/notas" `
                  -Method POST `
                  -ContentType "application/json" `
                  -Body '{"id": "NF-999", "cliente": "João Silva", "valor": 1000.0, "data_emissao": "2025-01-31"}'
```

GET:
```
Invoke-RestMethod -Uri "http://localhost:4566/restapis/<id>/dev/_user_request_/notas" `
                  -Method GET
```
