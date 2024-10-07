# Documentação: Implantação de uma Aplicação WordPress na AWS

## Passo a Passo

### 1. Criando a VPC e Sub-redes

#### 1.1. Criar VPC
1. No Console da AWS, vá para **VPC** e clique em **Criar VPC**.
2. Configure os detalhes:
   - Nome da VPC: `vpc-wp`
   - CIDR: `10.0.0.0/16`
3. Clique em **Criar**.

#### 1.2. Criar Sub-redes Públicas
No Console da AWS, navegue até **Sub-redes** e clique em **Criar sub-rede**.
1. Para a primeira sub-rede pública:
   - Nome da Sub-rede: `subrede-publica1`
   - VPC: `vpc-wp`
   - Zona de Disponibilidade: `us-east-1a`
   - Bloco CIDR: `10.0.1.0/24`
   - Clique em **Criar Sub-rede**.
   
2. Para a segunda sub-rede pública:
   - Nome da Sub-rede: `subrede-publica2`
   - Zona de Disponibilidade: `us-east-1b`
   - Bloco CIDR: `10.0.2.0/24`
   - Clique em **Criar Sub-rede**.

#### 1.3. Criar Sub-redes Privadas
Repita o processo para criar duas sub-redes privadas:
1. Para a primeira sub-rede privada:
   - Nome da Sub-rede: `subrede-privada1`
   - Zona de Disponibilidade: `us-east-1a`
   - Bloco CIDR: `10.0.3.0/24`
   
2. Para a segunda sub-rede privada:
   - Nome da Sub-rede: `subrede-privada2`
   - Zona de Disponibilidade: `us-east-1b`
   - Bloco CIDR: `10.0.4.0/24`

---

### 2. Criando Tabelas de Rotas

#### 2.1. Tabela de Rotas Pública
1. Vá para **Tabelas de Rotas** e clique em **Criar Tabela de Rotas**.
   - Nomeie a tabela como `wp-public-route` e associe-a à VPC `projeto-2-compass-vpc`.
   - Clique em **Criar Tabela de Rotas**.
   
2. Após criada, vá até a seção de **Rotas** e adicione a seguinte rota:
   - Destino: `0.0.0.0/0`
   - Destino do Alvo: **Internet Gateway**
   - Internet Gateway: selecione o criado (por exemplo, `wp-igw`).
   
3. Vá para a aba de **Associações de Sub-rede** e associe as sub-rede públicas `wp-publica-01` e `wp-publica-02`.

#### 2.2. Tabela de Rotas Privada
Repita o processo para criar a tabela de rotas privada:
1. Nome: `wp-route-private`
2. Não adicione rota para Internet.
3. Associe as sub-redes privadas `wp-privada-01` e `wp-privada-02` a essa tabela de rotas.

---

### 3. Criando Conexões de Rede

#### 3.1. Internet Gateway (IGW)
1. Navegue até **Internet Gateway** e clique em **Criar Internet Gateway**.
   - Nomeie o gateway como `wp-igw` e clique em **Criar**.
   
2. Após criado, vá para a seção de **Ações** e selecione **Vincular à VPC**. Selecione a VPC `projeto-2-compass-vpc`.

#### 3.2. NAT Gateway (NATG)
1. Vá para **NAT Gateway** e clique em **Criar NAT Gateway**.
   - Defina o nome como `wp-natg`.
   - Selecione a sub-rede pública `wp-publica-01` e crie um **IP elástico**.
   
2. Clique em **Criar NAT Gateway**.
3. Depois de criado, vá até a **Tabela de Rotas Privada** e adicione uma rota:
   - Destino: `0.0.0.0/0`
   - Alvo: **NAT Gateway**
   - Selecione `wp-natg`.

---

### 4. Configurando o Load Balancer

#### 4.1. Criar Security Group para o Load Balancer
1. Crie um novo security group chamado `lb-sg` com as seguintes regras de entrada:
   - **HTTP (80)** de qualquer lugar (`0.0.0.0/0`)
   - **HTTPS (443)** de qualquer lugar (`0.0.0.0/0`)

#### 4.2. Criar o Application Load Balancer
1. No console EC2, crie um novo **Application Load Balancer**:
   - Nome: `projeto-2-compass-lb`
   - Esquema: **Voltado para internet**
   - VPC: `projeto-2-compass-vpc`
   - Mapeamentos: Selecione as duas subnets públicas.
   - Security Group: `lb-sg`.
   
2. Configure **Listener** e **Target Group**:
   - Listener: **HTTP:80**
   - Target Group:
     - Nome: `projeto-2-compass-tg`
     - Protocolo: **HTTP**
     - Porta: **80**
     - Tipo de alvo: **Instâncias**
     - Health check path: `/`
     
3. Registre as instâncias EC2 no Target Group.

---

### 5. Criando as Instâncias EC2

#### 5.1. Lançar Instâncias EC2
1. No console EC2, clique em **Launch Instance**.
2. Selecione a **Amazon Linux 2 AMI**.
3. Escolha o tipo de instância: **t3.small**.
4. Em **Configure Instance Details**:
   - **Network**: selecione a VPC `projeto-2-compass-vpc`.
   - **Subnet**: escolha uma das subnets privadas (`wp-privada-01` ou `wp-privada-02`).
   - Ative a opção **Auto-assign Public IP** como **Disable**.
5. Em **Storage**, mantenha as configurações padrão.
6. Em **Tags**, adicione as tags conforme necessário.
7. Em **Configure Security Group**, selecione um grupo de segurança já existente ou crie um novo que permita o tráfego entre as instâncias EC2.
8. Revise e clique em **Launch**. Escolha ou crie um par de chaves SSH.

#### 5.2. Conectar-se às Instâncias EC2
1. Conecte-se à sua instância EC2 via SSH utilizando o par de chaves que você criou:
   ```bash
   ssh -i "sua-chave.pem" ec2-user@<private-ip-instance>
#### 5.3. Montar o Sistema de Arquivos EFS
Execute os seguintes comandos dentro do terminal da instância:
   ```bash
   # Monta o sistema de arquivos
   sudo mount -t efs -o tls fs-0867e43994669bc80:/ /mnt/efs

   # Verifica se a montagem está correta
   df -h
   ```
### 6. Criando o Arquivo docker-compose.yml

#### 6.1. Criar o arquivo docker-compose.yml
Crie um arquivo chamado `docker-compose.yml` na sua instância EC2 com o seguinte conteúdo:

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress
    container_name: wordpress
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: "db"  # Nome do serviço do banco de dados
      WORDPRESS_DB_USER: "wordpress"
      WORDPRESS_DB_PASSWORD: "wordpress"
      WORDPRESS_DB_NAME: "wordpress"
      TZ: "America/Sao_Paulo"
    volumes:
      - /mnt/efs:/var/www/html
    networks:
      - wp-network

  db:
    image: mysql:5.7  # Use uma versão adequada do MySQL
    container_name: mysql
    environment:
      MYSQL_DATABASE: "wordpress"
      MYSQL_USER: "wordpress"
      MYSQL_PASSWORD: "wordpress"
      MYSQL_ROOT_PASSWORD: "root_password"  # Senha do root (opcional)
      TZ: "America/Sao_Paulo"
    volumes:
      - db_data:/var/lib/mysql  # Persistência dos dados do MySQL
    networks:
      - wp-network

networks:
  wp-network:
    driver: bridge

volumes:
  db_data:  # Volume para persistir os dados do banco de dados
```

## 7. Iniciar o Docker Compose
Navegue até o diretório onde o arquivo `docker-compose.yml` está localizado.  
Execute o seguinte comando para iniciar os contêineres:

```plaintext
sudo docker-compose up -d
```

## 8. Configuração do Load Balancer

### 8.1. Criar Security Group para o Load Balancer
Crie um novo security group `lb-sg` com as seguintes regras de entrada:

- **HTTP (80)** de qualquer lugar (0.0.0.0/0)
- **HTTPS (443)** de qualquer lugar (0.0.0.0/0)

### 8.2. Criar o Application Load Balancer
No console EC2, crie um novo Application Load Balancer:

- **Nome:** projeto-2-compass-lb
- **Esquema:** Voltado para internet
- **VPC:** projeto-2-compass-vpc
- **Mapeamentos:** Selecione as duas subnets públicas
- **Security Group:** lb-sg

Configure o Listener e o Target Group:

- **Listener:** HTTP:80
- **Target Group:**
  - **Nome:** projeto-2-compass-tg
  - **Protocolo:** HTTP
  - **Porta:** 80
  - **Tipo de alvo:** Instâncias
  - **Health check path:** /

Registre as instâncias EC2 no Target Group.

## 9. Acessando o WordPress

Após configurar o Application Load Balancer e registrar suas instâncias EC2 no Target Group, você pode acessar o WordPress através do DNS do Load Balancer.

1. No Console da AWS, navegue até a seção **EC2**.
2. Clique em **Load Balancers** no menu lateral.
3. Selecione o Load Balancer que você criou (projeto-2-compass-lb).
4. Copie o **DNS Name** exibido na parte inferior da página.

5. Abra um navegador e cole o DNS Name copiado na barra de endereços:

6. Você deverá ver a tela de instalação do WordPress. Siga as instruções na tela para configurar seu site WordPress.





    




