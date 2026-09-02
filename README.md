#   Projeto Agro - DevOps CP4

# 



# 

# \---

# 

#  📁 Estrutura do projeto

# 

# ```text

# DevOps-cp4

# │

# ├── README.md

# │

# ├── azure

# │   └── CRUD

# │

# ├── database

# │   └── 01\_SQL\_GS\_26.sql

# │

# └── docker

# &#x20;   ├── Dockerfile

# &#x20;   └── Dockerfile.oracle

# ```

# 

# \---

# 

# \#  1 - Login na Azure

# 

# ```powershell

# az login

# ```

# 

# \---

# 

# \#  2 - Criar Resource Group

# 

# ```powershell

# az group create `

# &#x20; --name rg-projeto-agro `

# &#x20; --location southafricanorth

# ```

# 

# \---

# 

# \#  3 - Build da aplicação Java

# 

# Entrar na pasta da aplicação:

# 

# ```powershell

# cd "..\\Gs-Java\\projeto-agro"

# ```

# 

# Criar a imagem Docker:

# 

# ```powershell

# docker build -t projeto-agro-app .

# ```

# 

# \---

# 

# \#  4 - Build do banco Oracle

# 

# Entrar na pasta do banco:

# 

# ```powershell

# cd "..\\..\\Gs-Banco"

# ```

# 

# Criar a imagem:

# 

# ```powershell

# docker build -t projeto-agro-db:v5 .

# ```

# 

# \---

# 

# \#  5 - Testar Oracle localmente com persistência

# 

# Criar o volume:

# 

# ```powershell

# docker volume create oracle-persist-teste

# ```

# 

# Executar o container:

# 

# ```powershell

# docker run --rm `

# &#x20; --name oracle-persist-teste `

# &#x20; -e ORACLE\_PWD="$ORACLE\_PWD" `

# &#x20; -v oracle-persist-teste:/opt/oracle/oradata `

# &#x20; -p 1523:1521 `

# &#x20; projeto-agro-db:v5

# ```

# 

# Resultado esperado:

# 

# ```text

# DATABASE IS READY TO USE!

# ```

# 

# \---

# 

# \#  6 - Criar Azure Container Registry

# 

# ```powershell

# az acr create `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --name acrprojetoagro `

# &#x20; --sku Basic `

# &#x20; --admin-enabled true

# ```

# 

# \---

# 

# \#  7 - Login no ACR

# 

# ```powershell

# az acr login --name acrprojetoagro

# ```

# 

# \---

# 

# \#  8 - Publicar imagem da aplicação

# 

# Criar a tag:

# 

# ```powershell

# docker tag projeto-agro-app:latest `

# &#x20; acrprojetoagro.azurecr.io/projeto-agro-app:v2

# ```

# 

# Realizar o push:

# 

# ```powershell

# docker push acrprojetoagro.azurecr.io/projeto-agro-app:v2

# ```

# 

# \---

# 

# \#  9 - Publicar imagem do Oracle

# 

# Criar a tag:

# 

# ```powershell

# docker tag projeto-agro-db:v5 `

# &#x20; acrprojetoagro.azurecr.io/projeto-agro-db:v5

# ```

# 

# Realizar o push:

# 

# ```powershell

# docker push acrprojetoagro.azurecr.io/projeto-agro-db:v5

# ```

# 

# \---

# 

# \#  10 - Criar Storage Account

# 

# ```powershell

# az storage account create `

# &#x20; --name stagro25935 `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --location southafricanorth `

# &#x20; --sku Standard\_LRS

# ```

# 

# \---

# 

# \#  11 - Criar Azure File Share

# 

# Obter a chave da Storage Account:

# 

# ```powershell

# $STORAGE\_KEY = az storage account keys list `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --account-name stagro25935 `

# &#x20; --query "\[0].value" `

# &#x20; --output tsv

# ```

# 

# Criar o File Share:

# 

# ```powershell

# az storage share create `

# &#x20; --name oracledata `

# &#x20; --account-name stagro25935 `

# &#x20; --account-key "$STORAGE\_KEY"

# ```

# 

# \---

# 

# \#  12 - Obter credenciais do Azure Container Registry

# 

# Usuário:

# 

# ```powershell

# $ACR\_USER = az acr credential show `

# &#x20; --name acrprojetoagro `

# &#x20; --query username `

# &#x20; --output tsv

# ```

# 

# Senha:

# 

# ```powershell

# $ACR\_PASSWORD = az acr credential show `

# &#x20; --name acrprojetoagro `

# &#x20; --query "passwords\[0].value" `

# &#x20; --output tsv

# ```

# 

# \---

# 

# \#  13 - Definir senha do Oracle

# 

# A senha não deve ser armazenada diretamente no GitHub.

# 

# Definir localmente:

# 

# ```powershell

# $ORACLE\_PWD = "SUA\_SENHA"

# ```

# 

# > Substitua `SUA\_SENHA` somente no terminal local. Não publique a senha real no repositório.

# 

# \---

# 

# \#  14 - Criar container Oracle no Azure

# 

# ```powershell

# az container create `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --name projeto-agro-db `

# &#x20; --location southafricanorth `

# &#x20; --image acrprojetoagro.azurecr.io/projeto-agro-db:v1 `

# &#x20; --registry-login-server acrprojetoagro.azurecr.io `

# &#x20; --registry-username "$ACR\_USER" `

# &#x20; --registry-password "$ACR\_PASSWORD" `

# &#x20; --dns-name-label agrodb64217 `

# &#x20; --ports 1521 `

# &#x20; --os-type Linux `

# &#x20; --cpu 1 `

# &#x20; --memory 2 `

# &#x20; --secure-environment-variables ORACLE\_PWD="$ORACLE\_PWD"

# ```

# 

# \---

# 

# \#  15 - Verificar Oracle no Azure

# 

# ```powershell

# az container show `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --name projeto-agro-db `

# &#x20; --query "{Estado:containers\[0].instanceView.currentState.state}" `

# &#x20; --output table

# ```

# 

# Resultado esperado:

# 

# ```text

# Estado

# \-------

# Running

# ```

# 

# FQDN utilizado pelo banco:

# 

# ```text

# agrodb64217.southafricanorth.azurecontainer.io

# ```

# 

# Porta:

# 

# ```text

# 1521

# ```

# 

# Service Name:

# 

# ```text

# FREEPDB1

# ```

# 

# \---

# 

# \#  16 - Criar container da aplicação no Azure

# 

# ```powershell

# az container create `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --name projeto-agro-app `

# &#x20; --location southafricanorth `

# &#x20; --image acrprojetoagro.azurecr.io/projeto-agro-app:v2 `

# &#x20; --registry-login-server acrprojetoagro.azurecr.io `

# &#x20; --registry-username "$ACR\_USER" `

# &#x20; --registry-password "$ACR\_PASSWORD" `

# &#x20; --dns-name-label agroapp58092 `

# &#x20; --ports 8083 `

# &#x20; --os-type Linux `

# &#x20; --environment-variables `

# &#x20;   DB\_URL="jdbc:oracle:thin:@//agrodb64217.southafricanorth.azurecontainer.io:1521/FREEPDB1" `

# &#x20;   DB\_USERNAME="SYSTEM" `

# &#x20; --secure-environment-variables `

# &#x20;   DB\_PASSWORD="$ORACLE\_PWD"

# ```

# 

# \---

# 

# \#  17 - Verificar aplicação no Azure

# 

# ```powershell

# az container show `

# &#x20; --resource-group rg-projeto-agro `

# &#x20; --name projeto-agro-app `

# &#x20; --query "{Estado:containers\[0].instanceView.currentState.state}" `

# &#x20; --output table

# ```

# 

# Resultado esperado:

# 

# ```text

# Estado

# \-------

# Running

# ```

# 

# \---

# 

# \#  18 - URL da aplicação

# 

# URL pública:

# 

# ```text

# http://agroapp58092.southafricanorth.azurecontainer.io:8083

# ```

# 

# Definir a URL em uma variável para os testes:

# 

# ```powershell

# $BASE\_URL = "http://agroapp58092.southafricanorth.azurecontainer.io:8083"

# ```

# 

# \---

# 

# \#  19 - Testar comunicação entre aplicação e Oracle

# 

# ```powershell

# Invoke-RestMethod `

# &#x20; -Method GET `

# &#x20; -Uri "$BASE\_URL/bioma/99"

# ```

# 

# Exemplo de retorno:

# 

# ```json

# {

# &#x20; "idBioma": 99,

# &#x20; "nome": "ProvaACI",

# &#x20; "descricao": "Inserido direto no Oracle ACI"

# }

# ```

# 

# Esse teste demonstra que a aplicação executada no Azure Container Instances consegue consultar o banco Oracle executado em outro Azure Container Instance.

# 

# \---

# 

# \#  20 - CRUD - POST

# 

# Criar o JSON:

# 

# ```powershell

# $body = @{

# &#x20;   idBioma = 98

# &#x20;   nome = "TesteCRUD"

# &#x20;   descricao = "Criado pela API no ACI"

# } | ConvertTo-Json

# ```

# 

# Executar o POST:

# 

# ```powershell

# Invoke-RestMethod `

# &#x20; -Method POST `

# &#x20; -Uri "$BASE\_URL/bioma/novo" `

# &#x20; -ContentType "application/json" `

# &#x20; -Body $body

# ```

# 

# Resultado:

# 

# ```text

# idBioma nome       descricao

# \------- ----       ---------

# 98      TesteCRUD  Criado pela API no ACI

# ```

# 

# \---

# 

# \#  21 - Validar POST diretamente no Oracle

# 

# Conectar ao Oracle:

# 

# ```powershell

# docker run --rm -it `

# &#x20; container-registry.oracle.com/database/free:latest `

# &#x20; sqlplus system@agrodb64217.southafricanorth.azurecontainer.io`:1521/FREEPDB1

# ```

# 

# A senha será solicitada pelo SQLPlus.

# 

# Executar:

# 

# ```sql

# SELECT ID\_BIOMA, NOME, DESCRICAO

# FROM BIOMA

# WHERE ID\_BIOMA = 98;

# ```

# 

# Resultado:

# 

# ```text

# ID\_BIOMA  NOME       DESCRICAO

# \--------  ---------  -----------------------

# 98        TesteCRUD  Criado pela API no ACI

# ```

# 

# \---

# 

# \#  22 - CRUD - GET

# 

# ```powershell

# Invoke-RestMethod `

# &#x20; -Method GET `

# &#x20; -Uri "$BASE\_URL/bioma/98"

# ```

# 

# Resultado:

# 

# ```json

# {

# &#x20; "idBioma": 98,

# &#x20; "nome": "TesteCRUD",

# &#x20; "descricao": "Criado pela API no ACI"

# }

# ```

# 

# \---

# 

# \#  23 - CRUD - PUT

# 

# Criar o JSON atualizado:

# 

# ```powershell

# $body = @{

# &#x20;   idBioma = 98

# &#x20;   nome = "TestePUT"

# &#x20;   descricao = "Atualizado pela API no ACI"

# } | ConvertTo-Json

# ```

# 

# Executar:

# 

# ```powershell

# Invoke-RestMethod `

# &#x20; -Method PUT `

# &#x20; -Uri "$BASE\_URL/bioma/atualizar/98" `

# &#x20; -ContentType "application/json" `

# &#x20; -Body $body

# ```

# 

# \---

# 

# \#  24 - Validar PUT diretamente no Oracle

# 

# ```sql

# SELECT ID\_BIOMA, NOME, DESCRICAO

# FROM BIOMA

# WHERE ID\_BIOMA = 98;

# ```

# 

# Resultado:

# 

# ```text

# ID\_BIOMA  NOME      DESCRICAO

# \--------  --------  ---------------------------

# 98        TestePUT  Atualizado pela API no ACI

# ```

# 

# \---

# 

# \#  25 - CRUD - DELETE

# 

# ```powershell

# Invoke-RestMethod `

# &#x20; -Method DELETE `

# &#x20; -Uri "$BASE\_URL/bioma/remover/98"

# ```

# 

# \---

# 

# \#  26 - Validar DELETE diretamente no Oracle

# 

# ```sql

# SELECT ID\_BIOMA, NOME, DESCRICAO

# FROM BIOMA

# WHERE ID\_BIOMA = 98;

# ```

# 

# Resultado:

# 

# ```text

# no rows selected

# ```

# 

# Isso demonstra que o registro foi removido do banco Oracle utilizado pela aplicação no Azure.

# 

# \---

# 

# \#  27 - Persistência

# 

# Foi criada uma Storage Account no Azure:

# 

# ```text

# stagro25935

# ```

# 

# Também foi criado o Azure File Share:

# 

# ```text

# oracledata

# ```

# 

# Durante os testes foi validada a persistência local do Oracle utilizando Docker Volume em:

# 

# ```text

# /opt/oracle/oradata

# ```

# 

# O Azure File Share também foi criado para os testes de armazenamento no Azure.

# 

# > Observação: não declarar no relatório que o banco Oracle está utilizando o Azure File Share como volume persistente enquanto essa integração não estiver efetivamente validada.

# 

# \---

# 

# \#  28 - Arquivo DDL

# 

# O script utilizado para criação das estruturas do banco está disponível em:

# 

# ```text

# database/01\_SQL\_GS\_26.sql

# ```

# 

# \---

# 

# \#  29 - Dockerfile da aplicação

# 

# O Dockerfile utilizado para realizar o build da aplicação está disponível em:

# 

# ```text

# docker/Dockerfile

# ```

# 

# A aplicação é executada utilizando usuário não-root dentro do container.

# 

# \---

# 

# \#  30 - Dockerfile do Oracle

# 

# O Dockerfile utilizado para o Oracle está disponível em:

# 

# ```text

# docker/Dockerfile.oracle

# ```

# 

# 

# \#  Tecnologias utilizadas

# 

# \- Java 21

# \- Spring Boot

# \- Maven

# \- Oracle Database

# \- Docker

# \- Azure CLI

# \- Azure Container Registry

# \- Azure Container Instances

# \- Azure Storage Account

# \- Azure File Share

# \- Git

# \- GitHub

# 

# \---

# 

# \#  Principais recursos Azure

# 

# | Recurso | Nome |

# |---|---|

# | Resource Group | `rg-projeto-agro` |

# | Azure Container Registry | `acrprojetoagro` |

# | Container da aplicação | `projeto-agro-app` |

# | Container do banco | `projeto-agro-db` |

# | Storage Account | `stagro25935` |

# | File Share | `oracledata` |

# 

# \---

# 

# 

