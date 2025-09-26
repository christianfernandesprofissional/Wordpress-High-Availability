# Aplicação WordPress de Alta Disponibilidade na AWS

## Sobre o projeto

Este projeto apresenta uma aplicação **WordPress** com **alta disponibilidade** implementada na **AWS**, seguindo uma arquitetura escalável e segura. A aplicação é executada em **containers Docker** usando **Docker Compose**, hospedados em **instâncias EC2 distribuídas em múltiplas zonas de disponibilidade (AZs)** para garantir redundância e continuidade de serviço. O tráfego é distribuído por um **Elastic Load Balancer (ELB)**, enquanto o **Auto Scaling** garante ajuste automático da quantidade de instâncias conforme a demanda. O armazenamento de arquivos compartilhados é feito pelo **Amazon EFS**, e o banco de dados relacional é gerenciado pelo **Amazon RDS**.

## Tecnologias utilizadas

* **WordPress**: Sistema de gerenciamento de conteúdo (CMS)
* **Docker \& Docker Compose**: Containerização da aplicação e gerenciamento dos serviços
* **Amazon EC2**: Hospedagem dos containers WordPress em múltiplas zonas de disponibilidade
* **Amazon RDS**: Banco de dados relacional MySQL gerenciado
* **Amazon EFS**: Sistema de arquivos compartilhado para uploads e conteúdo persistente
* **Elastic Load Balancer (ELB)**: Distribuição de tráfego entre múltiplas instâncias EC2
* **Auto Scaling**: Ajuste automático da quantidade de instâncias EC2 conforme a demanda
* **Bastion Host**: Acesso seguro às instâncias privadas

## Pré-requisitos

Antes de iniciar a implantação da aplicação, é necessário:

* **Conta ativa na AWS** com permissões para criar EC2, RDS, EFS, ELB e Auto Scaling
* Conhecimento básico em **Linux**, **Docker/Docker Compose** e **bancos de dados relacionais (MySQL)** para um bom entendimento do processo mostrado neste documento

## Sumário

- [Estrutura de rede;](#Estrutura-de-rede)


***
# Início

<div id="Estrutura-de-rede">
  
## Estrutura de rede



Para iniciarmos nossa implementação, primeiro devemos criar a nossa VPC, para isso na tela inicial da sua conta AWS e entre no menu da VPC escrevendo VPC na barra de pesquisa. Então selecione criar VPC, coloque um nome de sua preferência, preencha sua CIDR e Tags caso queira. A VPC utilizada neste projeto se chama vpcPrincipal. 


![criar-vpc](/imagens/criar-vpc.png "Criar VPC")


Agora com VPC criada somos capazes de criar nossas sub-redes, no painel da VPC selecione "Sub-redes" e depois "Criar sub-rede", serão 4 sub-redes, 2 públicas e 2 privadas. Iremos trabalhar com zonas de disponibilidades diferentes então será uma pública e uma privada para zona us-east-1a e uma pública e uma privada para a zona us-east-1b, mas altere de acordo com sua necessidade.


![sub-redes](/imagens/sub-redes.png "Sub-Redes")


Repare que após a criação das sub-redes ainda não temos como saber qual de fato é uma sub-rede pública, pois nenhuma tem acesso a internet ainda. Para isso devemos criar um Gateway de internet, para isso no painel da VPC acesse "Gateways da Internet", crie seu gateway e associe a sua VPC.



![gateway-internet](/imagens/gateway-internet.png "Gateway de Internet")



Agora que temos um Gateway para Internet podemos criar as rotas necessárias para que as sub-redes públicas tenham acesso a internet. Para isso clique em "Tabelas de rotas" e crie 3 rotas, uma rota será nossa saída para a internet, e as outras duas serão as rotas das sub-redes privadas. Com as 3 rotas criadas selecione a rota pública, e na aba "Rotas" selecione "Editar rotas".


![tabelas-de-rotas](/imagens/tabelas-de-rotas.png "Tabelas de rotas")


Selecione "Adicionar rota" e escolha a opção de Gateway de internet, selecione o Gateway criado e salve as alterações. Após esse procedimento temos uma rota apontando para a saída de internet.


![editar-rotas-gateway-internet](/imagens/editar-rotas-gateway-internet.png "Editar rota pública")



Volte ao menu de sub-redes, e selecione cada rede e verifique se as duas sub-redes públicas estão associadas a tabela de rotas com saída para internet que acabamos de criar. Se necessário edite a associação da tabela de rotas.


![sub-rede-rota-internet](/imagens/sub-rede-rota-internet.png "Sub-rede com rota para internet")



Após configurar as sub-redes públicas é necessário configurar as sub-redes privadas, para isso é necessário criar dois Gateways NAT conectados as sub-redes públicas. Isso é necessário para que nossas instâncias EC2 privadas possam ter acesso a internet e ao mesmo tempo não serem acessíveis da internet. Para criar os Gateways NAT vá em "Gateway NAT" e crie dois gateways, o primeiro associado a sub-rede pública da zona us-east-1a, e outro associado a sub-rede públic us-east-1b. Repare que é necessário também criar IPs elásticos para cada Gateway.



![gateway-nat](/imagens/gateway-nat.png "Criar gateway NAT")



Com os Gateways NAT criados devemos voltar para a tabela de rotas, e agora nas rotas privadas criadas anteriormente, devemos editar a primeira para que ela se conecte ao Gateway NAT associado a sub-rede pública da zona us-east-1a, e editar a segunda rota privada para se conectar ao Gateway NAT associado a sub-rede pública da zona us-east-1b.



![editar-rotas-gateway-nat](/imagens/editar-rotas-gateway-nat.png "Associar sub-rede ao gateway NAT")



Agora a estrutura de rede está completa e pronta para ser utilizada pelas nossas instâncias.

</div>

## Security Groups



Antes de prosseguir com os outros recursos vamos deixar os Security Groups necessários prontos para serem usados, para isso vá até o painel da EC2 e selecione "Security groups". Devemos criar 5 Security Groups, um para o RDS, outro para as instâncias EC2 do Wordpress, um para o Bastion Host, um para o Load Balancer, e um para o EFS.



![Security-groups](/imagens/security-groups.png "Security Groups")



Após a criação vamos configurar cada um dos Security Groups ajustando as regras de entrada, começando pelo Security Group do Bastion Host você deve permitir acesso via SSH (Para criar sua chave SSH no painel EC2 em Rede e segurança clique em Pares de chaves e crie sua chave, caso não queira também é possível criar uma chave durante a criação da instância do Bastion Host).



![regras-de-entrada-bastion](/imagens/regras-de-entrada-bastion.png "Regras de entrada Bastion Host")



Para o Security Group das instâncias do Wordpress configure as regras de entrada permitindo qualquer requisição HTTP, e a entrada via SSH do Security Group do Bastion Host.



![regras-de-entrada-wordpress](/imagens/regras-de-entrada-wordpress.png "Regras de entrada Wordpress EC2")



Para o Security Group do RDS devemos permitir a entrada do tipo MySQL/Aurora para o Security Group das instâncias do Wordpress.



![regras-de-entrada-database](/imagens/regras-de-entrada-database.png "Regras de entrada RDS")


Para o Security Group do EFS devemos permitir a entrada do tipo NFS para o Security Group das instâncias do Wordpress.



![regras-de-entrada-efs](/imagens/regras-de-entrada-efs.png "Regras de entrada EFS")



E por último para o Load Balancer devemos permitir qualquer requisição HTTP de qualquer endereço IP.



![regras-de-entrada-load-balancer](/imagens/regras-de-entrada-load-balancer.png "Regras de entrada Load Balancer")



## RDS



Vamos iniciar a criação do RDS, para isso pesquise RDS na barra de pesquisa e selecione "Aurora e RDS", selecione "Criar um banco de dados". Para este projeto selecione o banco de dados MySQL. Escolha o modelo para seu caso de uso, para este projeto seguiremos com o nível gratuito, com uma imagem t3.micro. Configure o nome do seu banco de dados, o usuário e a senha.


![criacao-rds](/imagens/criacao-rds.png "Criação do RDS")


Em Conectividade selecione a VPC criada anteriormente e também o Security Group do banco de dados. Desça até o final da página e clique em "Criar banco de dados".



![criacao-rds-conectividade](/imagens/criacao-rds-conectividade.png "Criação do RDS")



## EFS



Para criar o EFS digite EFS na barra de pesquisa e clique em "Criar sistema de arquivos" e depois em "Personalizar". Se preferir digite um nome para seu EFS e no final da página clique em "Próximo", na configuração de rede, selecione as duas zonas de disponibilidade que vamos trabalhar, as sub-redes privadas e o grupo de segurança do EFS, clique em "Próximo" até o final e crie o seu EFS


![criacao-efs-conectividade](/imagens/criacao-efs-conectividade.png "Criação do EFS")



## Load Balancer


Para que o nosso Wordpress funcione corretamente e as instâncias não fiquem sobrecarregadas, criar um Load Balancer é essencial para dividir a carga entre as instâncias. Para cria-lo, no painel da EC2 selecione "Load balancers" e então clique em "Criar load balancer". Existem alguns tipos de Load Balancer, mas para este projeto iremos usar o Application Load Balancer, clique em "Criar" e siga para próxima etapa.


Nas configurações básicas digite o nome do seu Load Balancer e vá para a seção de mapeamento do rede, nessa parte temos que selecionar a VPC criada, as zonas de disponibilidade, e as sub-redes públicas de cada zona


Na seção de grupos de segurança, selecione o grupo do Load Balancer criado anteriormente


![criacao-load-balancer](/imagens/criacao-load-balancer.png "Criação do Load Balancer")



Desça para a seleção de grupo de destino, que é para onde nosso Load Balancer irá redirecionar as requisições. Caso você não tenha um grupo de destino, basta clicar em "crie um grupo de destino". Por enquanto nós não temos nenhuma instância para associar ao grupo de destino, mas isso será feito pelo Auto Scaling.


Após a seleção desça ao fim da página e crie o load balancer.

## User-data

Vamos construir nosso user-data para que as instâncias criadas sempre iniciem da maneira correta. Para isso vamos seguir 3 etapas, primeiro vamos ao nosso EFS, selecione seu EFS, clique em "Anexar"  copie o comando de montagem do EFS somente até a parte destacada e salve provisóriamente

![comando-montagem-efs](/imagens/comando-montagem-efs.png "Comando de montagem do EFS")


Depois vá no seu RDS e copie a endpoint do seu RDS


![endpoint-RDS](/imagens/endpoint-RDS.png "Endpoint RDS")



Agorá vá ao seu Load Balancer e copie também, o endpoint fornecido pelo Load Balancer


![endpoint-Load-Balancer](/imagens/endpoint-Load-Balancer.png "Endpoint Load Balancer")


Com o comando de montagem e os endpoints em mãos podemos criar nosso user-data, copie o código abaixo e substitua os textos copiados nos lugares indicados:


    #!/bin/bash

    # Instala Docker
    sudo yum update -y
    # Install Docker
    sudo yum install docker -y
    sudo service docker start
    sudo systemctl enable docker
    sudo usermod -a -G docker ec2-user
    sudo chmod 666 /var/run/docker.sock
    
    # Instala Docker Compose
    sudo curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    
    mkdir /my-compose

    # Instala MySQL
    sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-5.noarch.rpm
    sudo dnf install -y https://dev.mysql.com/get/mysql80-community-release-el9-5.noarch.rpm
    sudo dnf install -y mysql-community-server
    
    RDS_HOST="[SEU ENDPOINT RDS AQUI]"
    RDS_ADMIN_USER="wordpress"
    RDS_ADMIN_PASSWORD="[SUA SENHA DO RDS]"
    WP_DB_NAME="wordpress"
    
    mysql -h $RDS_HOST -u $RDS_ADMIN_USER -p$RDS_ADMIN_PASSWORD <<EOF
    CREATE DATABASE IF NOT EXISTS $WP_DB_NAME CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    EOF
    
    sudo mkdir -p /efs/wp-content
    sudo chmod -R 777 /efs
    
    
    until [SEU COMANDO DE MONTAGEM DO EFS AQUI] /efs/wp-content/; do
        echo "Aguardando EFS ficar disponível..."
        sleep 10
    done
    
    sudo chown -R 33:33 /efs/wp-content

    # Cria o docker-compose
    echo "version: '3.3'" >> /my-compose/docker-compose.yml
    echo >> /my-compose/docker-compose.yml
    echo "services: " >> /my-compose/docker-compose.yml
    echo "  wordpress:" >> /my-compose/docker-compose.yml
    echo "    image: wordpress:latest" >> /my-compose/docker-compose.yml
    echo "    restart: always" >> /my-compose/docker-compose.yml
    echo "    ports:" >> /my-compose/docker-compose.yml
    echo "      - \"80:80\"" >> /my-compose/docker-compose.yml
    echo "    environment:" >> /my-compose/docker-compose.yml
    echo "      WORDPRESS_DB_HOST: [SEU ENDPOINT RDS AQUI]:3306" >> /my-compose/docker-compose.yml
    echo "      WORDPRESS_DB_USER: wordpress" >> /my-compose/docker-compose.yml
    echo "      WORDPRESS_DB_PASSWORD: [SUA SENHA DO RDS]" >> /my-compose/docker-compose.yml
    echo "      WORDPRESS_DB_NAME: wordpress" >> /my-compose/docker-compose.yml
    echo "    volumes:" >> /my-compose/docker-compose.yml
    echo "      - /efs/wp-content:/var/www/html/wp-content" >> /my-compose/docker-compose.yml
    echo >> /my-compose/docker-compose.yml
    
    cd /my-compose
    docker-compose up -d
    
    # Variável do DNS do Load Balancer
    LB_DNS="[SEU ENDPOINT DO LOAD BALANCER AQUI]"
    
    # Atualiza URLs do WordPress para usar o Load Balancer
    mysql -h $RDS_HOST -u $RDS_ADMIN_USER -p$RDS_ADMIN_PASSWORD $WP_DB_NAME <<EOF
    UPDATE wp_options 
      SET option_value='http://$LB_DNS' 
      WHERE option_name IN ('siteurl','home');
    EOF


Com o user-data pronto podemos ir para a próxima etapa.

## Modelo de execução

Agora está ná hora de preparar o modelo para que o Auto scaling crie nossas instâncias, para isso no painel do EC2 selecione "Modelos de execução" e clique em criar modelo de execução. Iremos usar a imagem do Linux da AWS em uma t2.micro

![modelo-de-execucao-1](/imagens/modelo-de-execucao-1.png "Imagem modelo de execução")


Não iremos deixar nenhuma configuração de rede associada, vamos somente associar ao grupo de segurança das instâncias do wordpress

![modelo-de-execucao-2](/imagens/modelo-de-execucao-2.png "Rede e SG modelo de execução")

Desça até detalhes avançados, e no final em "Dados do usuário" cole o user-data que deixamos pronto

![modelo-de-execucao-3](/imagens/modelo-de-execucao-3.png "User-data modelo de execução")


Agora temos nosso modelo de execução pronto para ser usado pelo Auto Scaling.


## Auto Scaling



Com o nosso banco de dados RDS e nosso sistema de arquivos EFS prontos, podemos começar a criar o Auto Scaling que irá construir as instâncias necessárias automaticamente de acordo com a quantidade mínima configurada, e irá ajustar o número de instâncias conforme o número de acessos ao nosso Wordpress.







\[IMAGEM 20]



## User-data

