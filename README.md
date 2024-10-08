# Documentação do Projeto AWS

## Sumário
1. [Configuração da VPC](#configuração-da-vpc)
2. [Criação de Sub-redes](#criação-de-sub-redes)
3. [Configuração do Gateway](#configuração-do-gateway)
4. [Criação do RDS](#criação-do-rds)
5. [Criação do EFS](#criação-do-efs)
6. [Uso do Script de Inicialização da Instância](#uso-do-script-de-inicialização-da-instância)
7. [Configuração do Docker Compose](#configuração-do-docker-compose)

## Configuração da VPC

1. **Acesse o Console da AWS** e navegue até o serviço **VPC**.
2. **Crie uma nova VPC**:
   - Nome: `MinhaVPC`
   - CIDR Block: `10.0.0.0/16`
3. **Salve o ID da VPC**, pois será necessário para as próximas etapas.

## Criação de Sub-redes

### Sub-redes Públicas

1. **Crie uma Sub-rede pública**:
   - Nome: `PublicSubnet`
   - VPC: `MinhaVPC`
   - CIDR Block: `10.0.1.0/24`
   - Availability Zone: Escolha uma (por exemplo, `us-east-1a`).

### Sub-redes Privadas

1. **Crie uma Sub-rede privada**:
   - Nome: `PrivateSubnet`
   - VPC: `MinhaVPC`
   - CIDR Block: `10.0.2.0/24`
   - Availability Zone: Escolha uma (por exemplo, `us-east-1b`).

## Configuração do Gateway

1. **Crie um Internet Gateway**:
   - Nome: `MinhaIGW`
2. **Anexe o Internet Gateway à VPC**:
   - Selecione `MinhaVPC` e anexe `MinhaIGW`.
3. **Atualize a tabela de rotas da Sub-rede pública**:
   - Adicione uma rota:
     - Destino: `0.0.0.0/0`
     - Target: `MinhaIGW`

## Criação do RDS

1. **Acesse o Console da AWS** e navegue até o serviço **RDS**.
2. **Crie uma nova instância de banco de dados**:
   - Tipo de banco de dados: `MySQL`
   - Nome da instância: `database-1`
   - Tipo de instância: `db.t3.micro`
   - VPC: `MinhaVPC`
   - Sub-rede: `PrivateSubnet`
   - Configuração de segurança: Selecione um grupo de segurança que permita acesso ao MySQL (porta 3306) de sua aplicação.

## Criação do EFS

1. **Acesse o Console da AWS** e navegue até o serviço **EFS**.
2. **Crie um novo sistema de arquivos EFS**:
   - Nome: `MeuEFS`
   - VPC: `MinhaVPC`
   - Configure as opções de segurança para permitir acesso a partir de suas instâncias EC2.

## Uso do Script de Inicialização da Instância

O seguinte script pode ser utilizado para inicializar sua instância EC2:

```bash
#!/bin/bash
# Atualiza o sistema e instala o Docker e EFS
yum update -y
yum install docker -y
yum install amazon-efs-utils -y

# Inicia o serviço Docker
systemctl start docker
systemctl enable docker

# Instala o Docker Compose
curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
mv /usr/local/bin/docker-compose /bin/docker-compose

# Cria diretórios para EFS
mkdir -p /mnt/efs/wordpress

# Monta o EFS
mount -t nfs4 -o nfsvers=4.1 fs-0ae75c1b0a5f7958e.efs.us-east-1.amazonaws.com:/ /mnt/efs

# Habilita montagem automática
echo "fs-0ae75c1b0a5f7958e.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs defaults 0 0" >> /etc/fstab

# Inicia o Docker Compose
cd /home/ec2-user
docker-compose up -d
```

## Crie um arquivo docker-compose.yml para configurar o WordPress:
```bash
version: '3'
services:
  wordpress:
    image: wordpress:latest
    volumes:
      - /mnt/efs/wordpress:/var/www/html
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: database-1.cr0o8ky00uod.us-east-1.rds.amazonaws.com:3306
      WORDPRESS_DB_USER: teste
      WORDPRESS_DB_PASSWORD: teste123
      WORDPRESS_DB_NAME: database-1
```
## COnclusão
Com essas etapas, você terá configurado sua VPC, sub-redes, RDS, EFS, e um ambiente Docker com WordPress. Certifique-se de que suas regras de segurança estão configuradas corretamente para permitir o tráfego necessário entre os componentes.
