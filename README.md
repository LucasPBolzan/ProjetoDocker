# Trabalho de Deploy do WordPress com Docker e AWS

## Sumário
1. [Introdução](#introdução)
2. [Configuração da VPC e Subnets](#configuração-da-vpc-e-subnets)
3. [Criação dos Grupos de Segurança](#criação-dos-grupos-de-segurança)
4. [Criação do Banco de Dados RDS](#criação-do-banco-de-dados-rds)
5. [Criação do EFS (Elastic File System)](#criação-do-efs)
6. [Configuração das Instâncias EC2 com Docker e EFS](#configuração-das-instâncias-ec2-com-docker-e-efs)
7. [Configuração do Load Balancer](#configuração-do-load-balancer)
8. [Finalização](#finalização)

## Introdução
Este documento descreve o processo de deploy de uma aplicação WordPress utilizando Docker, um banco de dados MySQL no Amazon RDS, e o armazenamento de arquivos estáticos no AWS EFS (Elastic File System). Além disso, o serviço é balanceado através de um Load Balancer (LB). 

## Configuração da VPC e Subnets

1. **Criação da VPC**:
   - Crie uma VPC com CIDR adequado (ex: `10.0.0.0/16`).
   
2. **Subnets**:
   - Crie subnets públicas e privadas dentro da VPC.

3. **Internet Gateway (IGW)**:
   - Crie um IGW e associe à VPC.

4. **NAT Gateway**:
   - Crie um NAT Gateway nas subnets públicas para permitir que as instâncias nas subnets privadas acessem a internet.

## Grupos de Segurança

1. **Load Balancer (ALB-SG)**
   - **Entrada**: HTTP (TCP, porta 80) de qualquer IP (0.0.0.0/0).
   - **Saída**: Todo tráfego permitido.

2. **Instâncias EC2 (EC2-SG)**
   - **Entrada**:
     - HTTP (TCP, porta 80) do ALB-SG.
     - NFS (TCP, porta 2049) do EFS-SG.
   - **Saída**: Todo tráfego permitido.

3. **Banco de Dados RDS (RDS-SG)**
   - **Entrada**: MySQL/Aurora (TCP, porta 3306) do EC2-SG.
   - **Saída**: Todo tráfego permitido.

4. **EFS (EFS-SG)**
   - **Entrada**: NFS (TCP, porta 2049) do EC2-SG.
   - **Saída**: Todo tráfego permitido.

## Criação do Banco de Dados RDS

1. **Criação do RDS**:
   - Escolha o engine MySQL.
   - Defina as credenciais (usuário e senha do banco).
   - Configure o acesso a partir das subnets privadas e atribua o grupo de segurança específico para o banco de dados.

2. **Endpoint do RDS**:
   - Guarde o endpoint para configurar no arquivo de ambiente do WordPress.

## Criação do EFS

1. **Criação do EFS (Elastic File System)**:
   - Crie um EFS e configure os mount targets para as subnets em que as instâncias EC2 serão lançadas.
   
2. **Configuração do EFS**:
   - Defina as permissões adequadas para que o EFS possa ser acessado pelas instâncias EC2.

## Configuração das Instâncias EC2 com Docker e EFS

1. **Criação das Instâncias EC2**:
   - Lance instâncias EC2 nas subnets privadas e use o script de `user_data` para configurar automaticamente o ambiente.
``` bash
 #!/bin/bash

# Atualizando o sistema, bem como instalando o Docker e o EFS
sudo yum update -y
sudo yum install amazon-efs-utils nfs-utils -y
sudo amazon-linux-extras install docker -y

# Iniciando o serviço do Docker e habilitando para sempre iniciar com o sistema
sudo systemctl enable docker
sudo service docker start
sudo usermod -aG docker ec2-user

# Instalando o Docker Compose
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo mv /usr/local/bin/docker-compose /bin/docker-compose

# Criando o diretório para o WordPress no EFS
cd /mnt
sudo mkdir efs
cd efs
sudo mkdir wordpress

# Montando o EFS
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-058d386fd77f07916.efs.us-east-1.amazonaws.com:/ efs
sudo chown ec2-user:ec2-user /mnt/efs

# Criando o arquivo docker-compose.yml localmente
cat <<EOF > /home/ec2-user/docker-compose.yml
version: '3'
services:
  wordpress:
    image: wordpress:latest
    volumes:
      - /mnt/efs/wordpress:/var/www/html
    restart: always
    ports:
      - 80:80
      - 8080:8080
    environment:
      WORDPRESS_DB_HOST: databasewp.ch44cooycc7p.us-east-1.rds.amazonaws.com:3306
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: senha123
      WORDPRESS_DB_NAME: databasewp
EOF

# Habilitando a montagem do EFS sempre que o sistema for reiniciado
echo "fs-03fea74ec7e0948bb.efs.us-east-1.amazonaws.com:/ efs nfs defaults 0 0" | sudo tee -a /etc/fstab

# Subindo o container do Docker Compose
cd /home/ec2-user
sudo docker-compose up -d
```
     
2. **Docker Compose**:
   - O arquivo `docker-compose.yml` é criado automaticamente e configura o container WordPress, utilizando o EFS para armazenar os arquivos estáticos e conectando-se ao RDS para o banco de dados.

## Configuração do Load Balancer

1. **Criação do Load Balancer (Classic)**:
   - Crie um Classic Load Balancer (CLB) e configure-o para rotear o tráfego para as instâncias EC2.
   - Defina a porta de saída da aplicação como `80` ou `8080`.

2. **Configuração do DNS**:
   - Configure o DNS para apontar para o Load Balancer, garantindo que o serviço WordPress esteja acessível através de uma URL e não por um IP público.

## Finalização

1. **Verificação do Serviço**:
   - Acesse o Load Balancer via browser e verifique se a página de login do WordPress está disponível.



