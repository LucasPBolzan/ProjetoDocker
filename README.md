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

### 2. Configuração de Tabelas de Rotas
#### 2.1. Criação da Tabela de Rotas Pública
Acesse a seção de Tabelas de Rotas e clique em "Criar Tabela de Rotas".  
**Denomine a tabela como:** rt-wp-public e vincule-a à VPC correspondente.  
Clique em "Criar Tabela de Rotas".  
Após a criação, dirija-se à seção de Rotas e adicione a seguinte entrada:  
**Destino:** 0.0.0.0/0  
**Alvo:** Internet Gateway  
**Internet Gateway:** escolha o que foi criado anteriormente.  
Acesse a aba de Associações de Sub-rede e conecte as sub-redes públicas.

#### 2.2. Criação da Tabela de Rotas Privada
Repita o procedimento para configurar a tabela de rotas privada:  
**Nome:** rt-wp-private  
**Não adicione uma rota para Internet.**  
Vincule as sub-redes privadas a esta tabela de rotas.


### 3. Criando Conexões de Rede
#### 3.1. Internet Gateway (IGW)
Navegue até Internet Gateway e clique em Criar Internet Gateway.  
**Nomeie o gateway como:** wp-igw e clique em Criar.  
Após criado, vá para a seção de Ações e selecione Vincular à VPC. Selecione a VPC vpc-wp.

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

```
### 5. Configurando Auto Scaling
#### 5.1. Criar um Launch Configuration
No console EC2, vá para a seção **Auto Scaling** e clique em **Launch Configurations**.  
Clique em **Create Launch Configuration**.  
Selecione a Amazon Linux 2 AMI, escolha o tipo de instância (t3.small), e configure as opções de segurança e chave conforme necessário.

#### 5.2. Criar um Auto Scaling Group
Após criar o Launch Configuration, clique em **Auto Scaling Groups** e em **Create Auto Scaling Group**.  
Configure as seguintes opções:  
**Nome:** asg-wp  
**VPC:** vpc-wp  
**Subnets:** selecione as subnets privadas (wp-privada-01 e wp-privada-02).  
Defina as políticas de escalabilidade conforme desejado.

### 6. Configurando o Load Balancer
#### 6.1. Criar Security Group para o Load Balancer
Crie um novo security group com as seguintes regras de entrada:  
- **HTTP (80)** de qualquer lugar (0.0.0.0/0)  
- **HTTPS (443)** de qualquer lugar (0.0.0.0/0)

#### 6.2. Configuração do Application Load Balancer
No painel EC2, crie um novo Application Load Balancer com as seguintes configurações:  
**Nome:** lb-wp  
**Tipo:** Externo (voltado para a internet)  
**VPC:** vpc-wp  
**Subnets:** Escolha as duas sub-redes públicas.  
**Grupo de Segurança:** lb-sg  

Em seguida, ajuste as configurações do Listener e do Target Group:  
**Listener:** HTTP na porta 80  
**Target Group:**  
- **Nome:** tg-wp
- **Protocolo:** HTTP  
- **Porta:** 80  
- **Tipo de Alvo:** Instâncias  
- **Caminho de Verificação de Saúde:** /  

Por fim, adicione as instâncias EC2 ao Target Group.

### 7. Criando as Instâncias EC2
#### 7.1. Lançar Instâncias EC2
No console EC2, clique em **Launch Instance**.  
Selecione a Amazon Linux 2 AMI.  
Escolha o tipo de instância: t3.small.  
Em **Configure Instance Details:**  
**Network:** selecione a VPC vpw-wp.  
**Subnet:** escolha uma das subnets privadas.  
Ative a opção **Auto-assign Public IP** como **Disable**.  
Em **Storage**, mantenha as configurações padrão.  
Em **Tags**, adicione as tags conforme necessário.  
Em **Configure Security Group**, selecione um grupo de segurança já existente ou crie um novo que permita o tráfego entre as instâncias EC2.  
Revise e clique em **Launch**. Escolha ou crie um par de chaves SSH.

#### 7.2. Conectar-se às Instâncias EC2
Conecte-se à sua instância EC2 via SSH utilizando o par de chaves que você criou:

```bash
ssh -i "sua-chave.pem" ec2-user@<private-ip-instance>
```

#### 7.3. Montar o Sistema de Arquivos EFS
Execute os seguintes comandos dentro do terminal da instância:

```bash
# Monta o sistema de arquivos
sudo mount -t efs -o tls fs-0867e43994669bc80:/ /mnt/efs

# Verifica se a montagem está correta
df -h
```

### 8. Criando o Arquivo docker-compose.yml
#### 8.1. Criar o arquivo docker-compose.yml
Crie um arquivo chamado **docker-compose.yml** na sua instância EC2 com o seguinte conteúdo:

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress
    container_name: wordpress
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: "<nome-do-serviço>"
      WORDPRESS_DB_USER: "<seu-usuario>"
      WORDPRESS_DB_PASSWORD: "<sua-senha>"
      WORDPRESS_DB_NAME: "<nome-do-banco-de-dados>"
      TZ: "America/Sao_Paulo"
    volumes:
      - /mnt/efs:/var/www/html
    networks:
      - wp-network

  db:
    image: mysql:5.7
    container_name: mysql
    environment:
      MYSQL_DATABASE: "<seu-banco-de-dados>"
      MYSQL_USER: "<seu-usuario>"
      MYSQL_PASSWORD: "<sua-senha>"
      TZ: "America/Sao_Paulo"
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wp-network

networks:
  wp-network:
    driver: bridge

volumes:
  db_data:  # Volume para persistir os dados do banco de dados

```

### 9. Iniciar o Docker Compose
Navegue até o diretório onde o arquivo **docker-compose.yml** está localizado.  
Execute o seguinte comando para iniciar os contêineres:

```bash
sudo docker-compose up -d
```

### 10. Acessando o WordPress
Após configurar o Application Load Balancer e registrar suas instâncias EC2 no Target Group, você pode acessar o WordPress através do DNS do Load Balancer.

1. No Console da AWS, navegue até a seção **EC2**.
2. Clique em **Load Balancers** no menu lateral.
3. Selecione o Load Balancer que você criou (**projeto-2-compass-lb**).
4. Copie o **DNS Name** exibido na parte inferior da página.
5. Abra um navegador e cole o **DNS Name** copiado na barra de endereços.

