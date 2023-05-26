# Imersão TFTEC Azure ao vivo em SP

Esse projeto descreve os principais passos utilizados no nosso treinamento presencial de SP.
Temos como objetivo construir uma infraestrutura completa, simulando um cenário real de uma empresa que tem a necessidade de utilizar diversos recursos do Azure para rodar o seu negócio.

# Estrutura
O desenho de arquitetura informado abaixo mostra alguns detalhes de como está configurada a estrutura da empresa que iremos construir.

![TFTEC Cloud](https://github.com/raphasi/imersaoazure/blob/main/Arquitetura.png "Arquitetura Imersão")

## STEP01 - Criar um Resource Group e aplicar Azure Policy
1- Criar Resource Group
```cmd
   # Criar um Resource Group
   Nome: rg-azure
   Região: East US
 ```
2- Aplicar Azure Policy
   - Aplicar Policy no nível do Resource Group criado no passo anterior
   - Aplicar policy para recursos herdarem a TAG do RG-AZURE (Inherit a tag from the resource group)
   - Aplicar policy para limitar o tamanho das VMs para B2S e D2S_V3 (Allowed virtual machine size SKUs)
         - Configurar mensagem de não complice: Você escolheu um SKU de VM não permitido, Utilize apenas B2S ou D2S_V3
   


## STEP02 - Criar a topologia de rede HUB-SPOKE
Iremos utilizar pelo menos, 5 regiões diferentes do Azure, com o objetivo de atendermos as restrições de vCpus existentes em contas trial do Azure.

1 - Criar a estrutura da VNET-HUB:

```cmd
   # Criar um Resource Group
   Nome: rg-azure
   Região: East US
   
   # VNET-HUB
   Nome: vnet-hub
   Região: East-US
   Adress Space: 10.10.0.0/16
      
   # Subnets   
   Subnet: sub-srv
   Address Space: 10.10.1.0.0/24
   
   Subnet: sub-db
   Address Space: 10.10.2.0.0/24
   
   Subnet: sub-pvtendp
   Address Space: 10.10.252.0.0/24
   
   Subnet: AzureBastionSubnet
   Address Space: 10.10.250.0.0/24
   
   Subnet: GatewaySubnet
   Address Space: 10.10.251.0.0/24
   
```

2 - Criar a estrutura da VNET-SPOKE01:

```cmd
   # VNET-SPOKE01
   Nome: vnet-spoke01
   Região: UK South
   Adress Space: 10.20.0.0/16
   
   # Subnets
   Subnet: sub-intra
   Address Space: 10.20.1.0.0/24
     
```

3 - Criar a estrutura da VNET-SPOKE02:

```cmd
   # VNET-SPOKE02
   Nome: vnet-spoke02
   Região: Japan East
   Adress Space: 10.30.0.0/16
   
   # Subnets
   Subnet: sub-web
   Address Space: 10.30.1.0.0/24
   
   Subnet: sub-appgw
   Address Space: 10.30.250.0.0/24
     
```

## STEP02 - Deploy dos NSGs
Criar 3 NSGs de acordo com o modelo abaixo:
```cmd
   Nome: nsg-hub
   Região: east-us
   Associar Subnet: sub-srv e sub-db
```

```cmd
   Nome: nsg-intra
   Região: uk-south
   Associar Subnet: sub-intra
```

```cmd
   Nome: nsg-web
   Região: japan-east
   Associar Subnet: sub-web
```

## STEP03 - Deploy das VMs
Para todas as VMs iremos o tamanho B2S e utilizar o sistema operacional Windows Server 2022.

```cmd
   # EAST US
   Nome: vm-apps
   Região: east-us
   Vnet: vnet-hub
   Subnet: sub-srv
```

```cmd
   # UK SOUTH
   Nome: vm-intra01
   Região: uk-south
   Zone: Zone 1
   Vnet: vnet-spoke01
   Subnet: sub-intra
   
   Nome: vm-intra02
   Região: uk-south
   Zone: Zone 2
   Vnet: vnet-spoke01
   Subnet: sub-intra
```

```cmd 
   # JAPAN EAST
   Nome: vm-web01
   Região: japan-east
   Zone: Zone 1
   Vnet: vnet-spoke02
   Subnet: sub-web
   
   Nome: vm-web02
   Região: japan-east
   Zone: Zone 1
   Vnet: vnet-spoke02
   Subnet: sub-web
 ```  
 
   
2 - Deploy Azure Bastion

```cmd
   Nome: bastion01
   Região: East US
   SKU: Standard
   Configuration: Shareable
```

3 - Desabilitar o firewall de todas as VMs via Run Command, usando o seguinte comando:

```cmd
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

4 - Acessar as VMs via Bastion e realizar teste de ping entre VMs de vnets diferentes.

5 - Criar peering entre as seguintes vnets:
```cmd
vnet-hub/vnet-spoke01 - vnet-spoke01/vnet-hub
vnet-hub/vnet-spoke02 - vnet-spoke02/vnet-hub
```
4 - Realizar teste de ping entre VMs de vnets diferentes novamente.

## STEP04 - Configuração de domínios e certificados
1 - Criar um DNS Zone (Zona de DNS pública no Azure).

2 - Apontar registros NS (name server) do provedor público para o Azure.
Testar a validação do DNS com o seguinte comando:
```cmd
nslookup -type=SOA tftec.cloud
```
3 - Gerar um certificado digital válido:
Opções:
OPÇÃO01: https://app.zerossl.com
OPÇÃO02: https://punchsalad.com/ssl-certificate-generator/

4 - Converter o certificado para PFX:

https://www.sslshopper.com/ssl-converter.html


## STEP05 - Deploy Azure Key Vault
1- Deploy Azure Key Vault:
```cmd
   Nome: kvault-certs
   Região: east-us
   Usar pvt enpdoint na vnet-hub e subnet sub-pvtendp 
```

2- Fazer upload do certificado PFX no Key Vault

3- Criar um managed identify e conceder permissão no Key Vault como:
```cmd
   Secret: Get
```

## STEP06 - Deploy estrutura INTRANET
1- Instalar IIS nas VMS vm-intra01 e vm-intra02:
```cmd
Install-WindowsFeature -name Web-Server -IncludeManagementTools
```

2- Baixar código da aplicação aplicação do seguinte link do Github:

https://github.com/tftec-raphael

3- Copiar pasta da aplicação para o seguinte caminho
```cmd
c:\inetpub\wwwroot
```

4- Criar um novo site no IIS
```cmd
Nome: WebSite
Porta: 80
```


## STEP07 - Deploy Azure Load Balancer
1- Deploy Load Balancer 
```cmd
   Nome: lb-intra
   Tipo: Interno
   Região: uk-south
   Sku: Standard 
   Backend: bepool-intra
```
## STEP08 - Deploy Nat Gateway
1- Deploy Nat Gateway
```cmd
   Nome: nat-gw01
   Região: uk-south
   Tipo: Public IP Address
```
2- Associar a subnet sub-intra




## STEP09 - Deploy Aplicação Web Site
1- Instalar IIS nas VMS vm-intra01 e vm-intra02:
```cmd
Install-WindowsFeature -name Web-Server -IncludeManagementTools
```
2- Instalar .net core:

https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-7.0.5-windows-hosting-bundle-installer

3- Baixar código da aplicação aplicação do seguinte link do Github:

https://github.com/tftec-raphael

4- Copiar pasta da aplicação para o seguinte caminho
```cmd
c:\inetpub\wwwroot
```
5- Criar um novo site no IIS
```cmd
Nome: WebSite
Porta: 80
```

## STEP10 - Deploy SQL Server
1- Criar um novo SQL Server
```cmd
Nome: sqlsrvtftec01 (usar um nome único)
Location: east-us
Authentication method: Use SQL authentication   
   user: adminsql
   pass: Partiunuvem@2023
Allow Azure services and resources to access this server: YES

```



## STEP11 - Deploy SQL Database
1- Deploy SQL Datyabase
```cmd
Nome: xxxxxxx
Server: Usar o server criado no passo anterior
Compute + storage: Usar o Service Tier - DTU BASED - BASIC 
Backup storage redundancy: LRS
Add current client IP address: YES
Add Private Endpoint: pvt-sqldb01
Usar pvt enpdoint na vnet-hub e subnet sub-pvtendp
```


## STEP12 - Imoportar Database 
1- Instalar o SSMS
```cmd
Acessar o servidor vm-apps e instalar o SQL Management Studio
```
https://aka.ms/ssmsfullsetup

2- Importar database aplicação WebSite
```cmd
Abrir o SQL Management Studio
Alterar formato de autenticação para SQL authentication  
Logar com usuário e senha criados no passo anterior
Importar o database usando a opção de dacpac
```
3- Alterar connection string dos servidores
```cmd
Acessar as VMs vm-web01 e vm-web02
Acessar o arquivo applicationsetting.json 
Alterar a connection string o nome do banco de acordo com o banco criado
Abrir o promt de comando como administrador e realizar executar o comando iisreset nas duas vms
Testar abrir a aplicação usando localhost
```


## STEP13 - Application Gateway
1- Deplou Apg Gateway
```cmd
   Nome: appgw-site
   Região: japan-east
   Tier: Standard 2
   Tipo: Public IP Address
   Backend: bepool-web
   
   Rule01: rule-https-web
   Listener: lst-443
   Protocol: HTTPS - Escolher o certificado do Key Vault
   Backend Setting: bset80
   
   
   Rule01: rule-http-web
   Listener: lst-80
   Protocol: HTTPS
   Backend Setting: bset80
   Target type: Redirection
```

2- Criar um ASG
```cmd
   Nome: asg-web
   Região: japan-east   
```
3- Associar a placa de rede das seguintes VMs ao ASG:
```cmd
  vm-web01
  vm-web02
```
4- Liberar regra no NSG nsg-web
```cmd
Criar regra, liberando qualquer origem, setar o destino com o Application Security Group e usar portas 80 e 443.
```
5- Ajustar registro DNS externo
```cmd
Acessar a zona de DNS público e criar um registro do A.
Usar Alias Record Set e apontar para o IP público do App Gateway.
```

## STEP14 - Deploy Storage Account
1- Deploy Storage Account
```cmd
   Nome: tftecimages01 (usar seu nome exclusivo)
   Região: east-us
   Performance: Standard
   Redundancy: GRS - Marcar opção para  Read Access
   Usar pvt enpdoint na vnet-hub e subnet sub-pvtendp
   Storage Sub-Resource: Blob
```
2- Criar um Container blob
```cmd
   Nome: images
```
3- Criar um private endpoint
```cmd
   Usar pvt enpdoint na vnet-hub e subnet sub-pvtendp
   Storage Sub-Resource: File
```
4- Mapear Azure Files
```cmd
   Mapear o Azure Files nas seguintes VMs:
   vm-intra01
   vm-intra02
```
Validar se a conexão está apontando para ip interno


## STEP15 - Deploy Aplicação Imagens (WebApp)
1- Criar um App Service Plan
```cmd
   Nome: appplanimages
   Operating System: Windows
   Região: east-us
   Pricing plan: Standard S2
   ```
2- Criar um WebApp
```cmd
   Nome: tftecimages01 (usar seu nome exclusivo)
   Publish: Code
   Runtime Stack: .NET6
   Região: east-us
   Escolher AppServicePlan já criado
```
3- Deploy da aplicação
Baixar o zip da aplicação em 
https://portal.tftecprime.com.br

4- Instalar o Azure CLI

Windows
https://aka.ms/installazurecliwindows

macOS
https://learn.microsoft.com/pt-br/cli/azure/install-azure-cli-macos

5- Realizar o deploy da aplicação para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login
az webapp deploy --resource-group <group-name> --name <app-name> --src-path <zip-package-path>
```
6- Ajustar application setting para endereço do Storage Account



## STEP16 - Deploy VPN Site to Site (S2S)
1- Criar a estrutura da VNET-ONPREMISES

```cmd
   # Criar um Resource Group
   Nome: rg-onpremises
   Região: Brazil South
   
   # VNET-ONPREMISES
   Nome: vnet-onpremises
   Região: Brazil-South
   Adress Space: 192.168.0.0/16
      
   # Subnets   
   Subnet: sub-onpremises
   Address Space: 192.168.1.0.0/24
```

2- Deploy Virtual Network Gateway
```cmd
   Nome: vng01
   Região: brazil-south
   Gateway type: VPN
   SKU: VpnGw1
   Virtual Network: vnet-onpremises
   Public IP address name: pip-vng01
   Enable active-active mode: Disabled
   
OBS: Deploy pode levar mais de 30 minutos
```
3- Deploy VM Firewall
  ```cmd
   Nome: vm-fw
   Região: brazil-south
   Vnet: vnet-onpremises
   Subnet: sub-onpremises
   Usar um IP público stático
   Habilitar o forwarding da placa de rede
   ```
4- Instalar a feature de RAS na VM-FW
   - Criar uma interface: Azure utilizando o IP público do Virtual Network Gateway
   - Criar rota estática para rede 10.10.0.0/16
   - Criar rota estática para rede 10.20.0.0/16
   - Criar rota estática para rede 10.30.0.0/16
   - Configurar conexão persistente
   - Configurar uma chave compartilhada para conexão VPN

5- Criar um Local Network Gateway
   ```cmd
   Nome: lng01
   Região: brazil-south
   Endpoint: IP Address
   IP address: IP Público da vm-fw
   Address Space(s): 192.168.0.0/16
   ```
   
6- Criar um Connection
```cmd
   Nome: VPNAzure-Onpremises
   Connection Type: Site-to-Site
   Shared key (PSK): mesmo criado na configurção do RAS
   ```
 7- Deploy VM Client
  ```cmd
   Nome: vm-clieny
   Região: brazil-south
   Sistema Operacional: Windows 11
   Vnet: vnet-onpremises
   Subnet: sub-onpremises
   ```

## STEP17 - Realizar ajustes do perring na VNETs 
1- Ajustar Peering entre HUB e os Spokes (HUB)
```cmd
Virtual network gateway or Route Server
Use this virtual network's gateway or Route Server
```
2- Ajustar Peering entre HUB e os Spokes (Spoke01 e Spoke02)
```cmd
Virtual network gateway or Route Server
Use the remote virtual network's gateway or Route Server
```

## STEP18 - Deploy Route Table
1- Criar Route Table
```cmd
Nome: rtable-onpremises
Região: brazil-south
```
2- Criar rotas
```cmd
# ROTA REDE HUB
Nome: route-hub
Addres: 10.10.0.0/16
Next Hope: Virtual Appliance

# ROTA REDE SPOKE01
Nome: route-spoke01
Addres: 10.20.0.0/16
Next Hope: Virtual Appliance

# ROTA REDE SPOKE02
Nome: route-spoke02
Addres: 10.30.0.0/16
Next Hope: Virtual Appliance
```
3- Associar Route Table a subnet
```cmd
Associar o Route Table a subnet sub-onpremises
```

## STEP19 - Deploy VPN Point to Site
   - Address pool: 172.16.0.0/24
   - Tunnel type: OpenSSL
   - Authentication type: Azure Active Directory
     Dados do Azure Active Directory
```cmd
Tenant ID: https://login.microsoftonline.com/"your Directory ID"
Audience: 41b23e61-6c1e-4545-b367-cd054e0ed4b4 
Issuer: https://sts.windows.net/"your Directory ID"/
```  

## STEP20 - Deploy Azure Backup

## STEP21 - Container Registry
  ```cmd
   Nome: acrimagestftec01 (nome deve ser único)
   Região: brazil-south
   SKU: Standard
   ```

## STEP22 - Deploy Cluster AKS
  ```cmd
   Cluster present configuration: Standard
   Nome: aks-app01
   Região: australia-east
   AKS Pricing Tier: Standard
   Kubernetes version: Default
   Scale method: Autoscale
   Network configuration: Kubenet
   Container Registry:
   ```
      
## STEP23 - Deploy APIM
  ```cmd
   Cluster present configuration: Standard
   Nome: aks-app01
   Região: brazil-south
   AKS Pricing Tier: Standard
   Kubernetes version: Default
   Scale method: Autoscale
   Network configuration: Kubenet
   Cont
   ```
   
## STEP24 - Deploy Azure SQL Database
```cmd
Nome: xxxxxxx
Server: Usar o server criado no passo anterior
Compute + storage: Usar o Service Tier - DTU BASED - BASIC 
Backup storage redundancy: LRS
Add current client IP address: YES
Add Private Endpoint: pvt-sqldb02
Usar pvt enpdoint na vnet-hub e subnet sub-pvtendp
```


## STEP25 - Deploy WebApp
1- Criar um WebApp
```cmd
   Nome: tftecapi01 (usar seu nome exclusivo)
   Publish: Code
   Runtime Stack: .NET6
   Região: east-us
   Escolher AppServicePlan já criado
```
2- Deploy da aplicação
Baixar o zip da aplicação em 
https://portal.tftecprime.com.br

3- Realizar o deploy da aplicação para o WebApp
Abrir o Powershell ou Terminal e executar o seguinte comando:
```cmd
az login
az webapp deploy --resource-group <group-name> --name <app-name> --src-path <zip-package-path>
```
4- Ajustar application setting para endereço do SQL Database


## STEP19 - Deploy APIM
