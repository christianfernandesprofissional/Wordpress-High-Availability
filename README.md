# Aplicação WordPress de Alta Disponibilidade na AWS

## Sobre o projeto

Este projeto apresenta uma aplicação **WordPress** com **alta disponibilidade** implementada na **AWS**, seguindo uma arquitetura escalável e segura. A aplicação é executada em **containers Docker** usando **Docker Compose**, hospedados em **instâncias EC2 distribuídas em múltiplas zonas de disponibilidade (AZs)** para garantir redundância e continuidade de serviço. O tráfego é distribuído por um **Elastic Load Balancer (ELB)**, enquanto o **Auto Scaling** garante ajuste automático da quantidade de instâncias conforme a demanda. O armazenamento de arquivos compartilhados é feito pelo **Amazon EFS**, e o banco de dados relacional é gerenciado pelo **Amazon RDS**.

## Tecnologias utilizadas

- **WordPress**: Sistema de gerenciamento de conteúdo (CMS)
- **Docker & Docker Compose**: Containerização da aplicação e gerenciamento dos serviços
- **Amazon EC2**: Hospedagem dos containers WordPress em múltiplas zonas de disponibilidade
- **Amazon RDS**: Banco de dados relacional MySQL gerenciado
- **Amazon EFS**: Sistema de arquivos compartilhado para uploads e conteúdo persistente
- **Elastic Load Balancer (ELB)**: Distribuição de tráfego entre múltiplas instâncias EC2
- **Auto Scaling**: Ajuste automático da quantidade de instâncias EC2 conforme a demanda
- **Bastion Host**: Acesso seguro às instâncias privadas

## Pré-requisitos

Antes de iniciar a implantação da aplicação, é necessário:

- **Conta ativa na AWS** com permissões para criar EC2, RDS, EFS, ELB e Auto Scaling
- Conhecimento básico em **Linux**, **Docker/Docker Compose** e **bancos de dados relacionais (MySQL)** para um bom entendimento do processo mostrado neste documento
