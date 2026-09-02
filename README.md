# Projeto Agro - DevOps CP4

Projeto desenvolvido para realizar a conteinerização de uma aplicação Java Spring Boot e de um banco de dados Oracle, com publicação e execução dos containers na Microsoft Azure.

## Estrutura do projeto

```text
DevOps-cp4
│
├── README.md
│
├── app
│   ├── pom.xml
│   └── src
│
├── azure
│   └── CRUD
│
├── database
│   └── 01_SQL_GS_26.sql
│
└── docker
    ├── Dockerfile
    └── Dockerfile.oracle
```

## 1. Login na Azure

```powershell
az login
```

## 2. Criar Resource Group

```powershell
az group create `
  --name rg-projeto-agro `
  --location southafricanorth
```

## 3. Build da aplicação Java

Entrar na pasta da aplicação original:

```powershell
cd "..\Gs-Java\projeto-agro"
```

Criar a imagem Docker:

```powershell
docker build -t projeto-agro-app .
```

## 4. Build do banco Oracle

Entrar na pasta do banco:

```powershell
cd "..\..\Gs-Banco"
```

Criar a imagem:

```powershell
docker build -t projeto-agro-db:v5 .
```

## 5. Testar Oracle localmente com persistência

Criar um volume Docker:

```powershell
docker volume create oracle-persist-teste
```

Executar o Oracle utilizando o volume:

```powershell
docker run --rm `
  --name oracle-persist-teste `
  -e ORACLE_PWD="$ORACLE_PWD" `
  -v oracle-persist-teste:/opt/oracle/oradata `
  -p 1523:1521 `
  projeto-agro-db:v5
```

Resultado esperado:

```text
DATABASE IS READY TO USE!
```

## 6. Criar Azure Container Registry

```powershell
az acr create `
  --resource-group rg-projeto-agro `
  --name acrprojetoagro `
  --sku Basic `
  --admin-enabled true
```

## 7. Login no ACR

```powershell
az acr login --name acrprojetoagro
```

## 8. Publicar imagem da aplicação

Criar a tag:

```powershell
docker tag projeto-agro-app:latest `
  acrprojetoagro.azurecr.io/projeto-agro-app:v2
```

Realizar o push:

```powershell
docker push acrprojetoagro.azurecr.io/projeto-agro-app:v2
```

## 9. Publicar imagem do Oracle

Criar a tag:

```powershell
docker tag projeto-agro-db:v5 `
  acrprojetoagro.azurecr.io/projeto-agro-db:v5
```

Realizar o push:

```powershell
docker push acrprojetoagro.azurecr.io/projeto-agro-db:v5
```

## 10. Criar Storage Account

```powershell
az storage account create `
  --name stagro25935 `
  --resource-group rg-projeto-agro `
  --location southafricanorth `
  --sku Standard_LRS
```

## 11. Criar Azure File Share

Obter a chave da Storage Account:

```powershell
$STORAGE_KEY = az storage account keys list `
  --resource-group rg-projeto-agro `
  --account-name stagro25935 `
  --query "[0].value" `
  --output tsv
```

Criar o File Share:

```powershell
az storage share create `
  --name oracledata `
  --account-name stagro25935 `
  --account-key "$STORAGE_KEY"
```

## 12. Obter credenciais do Azure Container Registry

Obter o usuário:

```powershell
$ACR_USER = az acr credential show `
  --name acrprojetoagro `
  --query username `
  --output tsv
```

Obter a senha:

```powershell
$ACR_PASSWORD = az acr credential show `
  --name acrprojetoagro `
  --query "passwords[0].value" `
  --output tsv
```

## 13. Definir senha do Oracle

A senha do Oracle não deve ser armazenada diretamente no repositório.

Defina a variável somente no terminal local:

```powershell
$ORACLE_PWD = "SUA_SENHA"
```

Substitua `SUA_SENHA` somente no seu terminal. Não publique a senha real no GitHub.

## 14. Criar container Oracle no Azure

```powershell
az container create `
  --resource-group rg-projeto-agro `
  --name projeto-agro-db `
  --location southafricanorth `
  --image acrprojetoagro.azurecr.io/projeto-agro-db:v1 `
  --registry-login-server acrprojetoagro.azurecr.io `
  --registry-username "$ACR_USER" `
  --registry-password "$ACR_PASSWORD" `
  --dns-name-label agrodb64217 `
  --ports 1521 `
  --os-type Linux `
  --cpu 1 `
  --memory 2 `
  --secure-environment-variables ORACLE_PWD="$ORACLE_PWD"
```

## 15. Verificar Oracle no Azure

```powershell
az container show `
  --resource-group rg-projeto-agro `
  --name projeto-agro-db `
  --query "{Estado:containers[0].instanceView.currentState.state}" `
  --output table
```

Resultado esperado:

```text
Estado
-------
Running
```

Dados de conexão utilizados:

```text
Host: agrodb64217.southafricanorth.azurecontainer.io
Porta: 1521
Service Name: FREEPDB1
```

## 16. Criar container da aplicação no Azure

```powershell
az container create `
  --resource-group rg-projeto-agro `
  --name projeto-agro-app `
  --location southafricanorth `
  --image acrprojetoagro.azurecr.io/projeto-agro-app:v2 `
  --registry-login-server acrprojetoagro.azurecr.io `
  --registry-username "$ACR_USER" `
  --registry-password "$ACR_PASSWORD" `
  --dns-name-label agroapp58092 `
  --ports 8083 `
  --os-type Linux `
  --environment-variables `
    DB_URL="jdbc:oracle:thin:@//agrodb64217.southafricanorth.azurecontainer.io:1521/FREEPDB1" `
    DB_USERNAME="SYSTEM" `
  --secure-environment-variables `
    DB_PASSWORD="$ORACLE_PWD"
```

## 17. Verificar aplicação no Azure

```powershell
az container show `
  --resource-group rg-projeto-agro `
  --name projeto-agro-app `
  --query "{Estado:containers[0].instanceView.currentState.state}" `
  --output table
```

Resultado esperado:

```text
Estado
-------
Running
```

## 18. URL da aplicação

URL pública utilizada:

```text
http://agroapp58092.southafricanorth.azurecontainer.io:8083
```

Definir a URL em uma variável:

```powershell
$BASE_URL = "http://agroapp58092.southafricanorth.azurecontainer.io:8083"
```

## 19. Testar comunicação entre aplicação e Oracle

```powershell
Invoke-RestMethod `
  -Method GET `
  -Uri "$BASE_URL/bioma/99"
```

Exemplo de retorno:

```json
{
  "idBioma": 99,
  "nome": "ProvaACI",
  "descricao": "Inserido direto no Oracle ACI"
}
```

Esse teste demonstra a comunicação entre a aplicação Spring Boot executada no Azure Container Instances e o banco Oracle executado em outro container no Azure.

## 20. CRUD - POST

Criar o JSON:

```powershell
$body = @{
    idBioma = 98
    nome = "TesteCRUD"
    descricao = "Criado pela API no ACI"
} | ConvertTo-Json
```

Executar o POST:

```powershell
Invoke-RestMethod `
  -Method POST `
  -Uri "$BASE_URL/bioma/novo" `
  -ContentType "application/json" `
  -Body $body
```

Resultado:

```text
idBioma nome       descricao
------- ----       ---------
98      TesteCRUD  Criado pela API no ACI
```

## 21. Validar POST diretamente no Oracle

Conectar ao Oracle:

```powershell
docker run --rm -it `
  container-registry.oracle.com/database/free:latest `
  sqlplus system@agrodb64217.southafricanorth.azurecontainer.io`:1521/FREEPDB1
```

A senha será solicitada pelo SQLPlus.

Executar:

```sql
SELECT ID_BIOMA, NOME, DESCRICAO
FROM BIOMA
WHERE ID_BIOMA = 98;
```

Resultado:

```text
ID_BIOMA  NOME       DESCRICAO
--------  ---------  -----------------------
98        TesteCRUD  Criado pela API no ACI
```

## 22. CRUD - GET

```powershell
Invoke-RestMethod `
  -Method GET `
  -Uri "$BASE_URL/bioma/98"
```

Resultado:

```json
{
  "idBioma": 98,
  "nome": "TesteCRUD",
  "descricao": "Criado pela API no ACI"
}
```

## 23. CRUD - PUT

Criar o JSON atualizado:

```powershell
$body = @{
    idBioma = 98
    nome = "TestePUT"
    descricao = "Atualizado pela API no ACI"
} | ConvertTo-Json
```

Executar:

```powershell
Invoke-RestMethod `
  -Method PUT `
  -Uri "$BASE_URL/bioma/atualizar/98" `
  -ContentType "application/json" `
  -Body $body
```

## 24. Validar PUT diretamente no Oracle

```sql
SELECT ID_BIOMA, NOME, DESCRICAO
FROM BIOMA
WHERE ID_BIOMA = 98;
```

Resultado:

```text
ID_BIOMA  NOME      DESCRICAO
--------  --------  ---------------------------
98        TestePUT  Atualizado pela API no ACI
```

## 25. CRUD - DELETE

```powershell
Invoke-RestMethod `
  -Method DELETE `
  -Uri "$BASE_URL/bioma/remover/98"
```

## 26. Validar DELETE diretamente no Oracle

```sql
SELECT ID_BIOMA, NOME, DESCRICAO
FROM BIOMA
WHERE ID_BIOMA = 98;
```

Resultado:

```text
no rows selected
```

O resultado demonstra que o registro foi removido do banco Oracle utilizado pela aplicação no Azure.

## 27. Persistência

Foi criada uma Storage Account no Azure:

```text
stagro25935
```

Também foi criado o Azure File Share:

```text
oracledata
```

Durante os testes foi validada a persistência local do Oracle utilizando Docker Volume no diretório:

```text
/opt/oracle/oradata
```

O Azure File Share foi criado para os testes de armazenamento no Azure.

A utilização direta do Azure File Share como volume persistente dos arquivos do Oracle ainda não foi validada. Por esse motivo, essa integração não deve ser apresentada como concluída até que o armazenamento persistente do banco no Azure seja efetivamente comprovado.

## 28. Arquivo DDL

O script utilizado para criação das estruturas do banco está disponível em:

```text
database/01_SQL_GS_26.sql
```

## 29. Código da aplicação

O projeto Spring Boot está disponível em:

```text
app/
```

Principais arquivos:

```text
app/pom.xml
app/src/main/java/
app/src/main/resources/application.properties
```

As configurações de conexão com o banco utilizam variáveis de ambiente para evitar o armazenamento da senha do Oracle no código-fonte.

## 30. Dockerfile da aplicação

O Dockerfile utilizado para realizar o build da aplicação está disponível em:

```text
docker/Dockerfile
```

A aplicação é executada utilizando um usuário não-root dentro do container.

## 31. Dockerfile do Oracle

O Dockerfile utilizado para o Oracle está disponível em:

```text
docker/Dockerfile.oracle
```

## Tecnologias utilizadas

- Java 21
- Spring Boot
- Maven
- Oracle Database
- Docker
- Azure CLI
- Azure Container Registry
- Azure Container Instances
- Azure Storage Account
- Azure File Share
- Git
- GitHub

## Recursos utilizados no Azure

| Recurso | Nome |
| --- | --- |
| Resource Group | `rg-projeto-agro` |
| Azure Container Registry | `acrprojetoagro` |
| Container da aplicação | `projeto-agro-app` |
| Container do banco | `projeto-agro-db` |
| Storage Account | `stagro25935` |
| File Share | `oracledata` |

## Segurança

As credenciais utilizadas pelo projeto não são armazenadas diretamente no código-fonte.

A senha do Oracle é fornecida por variável de ambiente e enviada ao Azure Container Instances através de `--secure-environment-variables`.

As credenciais do Azure Container Registry são obtidas através do Azure CLI durante a execução dos comandos.

Nenhuma senha real, chave da Storage Account ou credencial do Azure deve ser publicada neste repositório.
