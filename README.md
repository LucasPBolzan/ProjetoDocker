# Documentação: Implantação de uma Aplicação WordPress na AWS
## Passo a Passo

### 1. Criando a VPC e Sub-redes
#### 1.1. Criar VPC
No Console da AWS, vá para VPC e clique em Criar VPC.  
Configure os detalhes:  
**Nome da VPC:** vpc-wp  
**CIDR:** 10.0.0.0/16  
Clique em Criar.

#### 1.2. Criar Sub-redes Públicas
No Console da AWS, navegue até Sub-redes e clique em Criar sub-rede.  
Para a primeira sub-rede pública:  
**Nome da Sub-rede:** subrede-publica1  
**VPC:** vpc-wp  
**Zona de Disponibilidade:** us-east-1a  
**Bloco CIDR:** 10.0.1.0/24  
Clique em Criar Sub-rede.  
Para a segunda sub-rede pública:  
**Nome da Sub-rede:** subrede-publica2  
**Zona de Disponibilidade:** us-east-1b  
**Bloco CIDR:** 10.0.2.0/24  
Clique em Criar Sub-rede.

#### 1.3. Criar Sub-redes Privadas
Repita o processo para criar duas sub-redes privadas:  
Para a primeira sub-rede privada:  
**Nome da Sub-rede:** subrede-privada1  
**Zona de Disponibilidade:** us-east-1a  
**Bloco CIDR:** 10.0.3.0/24  
Para a segunda sub-rede privada:  
**Nome da Sub-rede:** subrede-privada2  
**Zona de Disponibilidade:** us-east-1b  
**Bloco CIDR:** 10.0.4.0/24  

### 2. Criando Tabelas de Rotas
#### 2.1. Tabela de Rotas Pública
Vá para Tabelas de Rotas e clique em Criar Tabela de Rotas.  
**Nomeie a tabela como:** wp-public-route e associe-a à VPC projeto-2-compass-vpc.  
Clique em Criar Tabela de Rotas.  
Após criada, vá até a seção de Rotas e adicione a seguinte rota:  
**Destino:** 0.0.0.0/0  
**Destino do Alvo:** Internet Gateway  
**Internet Gateway:** selecione o criado (por exemplo, wp-igw).  
Vá para a aba de Associações de Sub-rede e associe as sub-rede públicas wp-publica-01 e wp-publica-02.

#### 2.2. Tabela de Rotas Privada
Repita o processo para criar a tabela de rotas privada:  
**Nome:** wp-route-private  
**Não adicione rota para Internet.**  
Associe as sub-redes privadas wp-privada-01 e wp-privada-02 a essa tabela de rotas.

### 3. Criando Conexões de Rede
#### 3.1. Internet Gateway (IGW)
Navegue até Internet Gateway e clique em Criar Internet Gateway.  
**Nomeie o gateway como:** wp-igw e clique em Criar.  
Após criado, vá para a seção de Ações e selecione Vincular à VPC. Selecione a VPC projeto-2-compass-vpc.

#### 3.2. NAT Gateway (NATG)
Vá para NAT Gateway e clique em Criar NAT Gateway.  
**Defina o nome como:** wp-natg.  
**Selecione a sub-rede pública:** wp-publica-01 e crie um IP elástico.  
Clique em Criar NAT Gateway.  
Depois de criado, vá até a Tabela de Rotas Privada e adicione uma rota:  
**Destino:** 0.0.0.0/0  
**Alvo:** NAT Gateway  
**Selecione:** wp-natg.

### 4. Configurando o EFS
#### 4.1. Criar um Sistema de Arquivos EFS
No Console da AWS, navegue até EFS e clique em Criar Sistema de Arquivos.  
**Nome do Sistema de Arquivos:** efs-wp  
**VPC:** selecione a VPC vpc-wp.  
Clique em Criar.  

#### 4.2. Montar o EFS nas Instâncias EC2
Após a criação do sistema de arquivos, copie o ID do EFS (ex: fs-0867e43994669bc80).  
Conecte-se às instâncias EC2 e execute os seguintes comandos no terminal:

```bash
# Instale o cliente EFS
sudo yum install -y amazon-efs-utils

# Crie o ponto de montagem
sudo mkdir /mnt/efs

# Monte o sistema de arquivos
sudo mount -t efs -o tls fs-0867e43994669bc80:/ /mnt/efs

# Verifique se a montagem está correta
df -h
