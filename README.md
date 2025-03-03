# Entrega Desafio - Infraestrutura AWS com Terraform
## Descrição do Código - Tarefa 1
Este desafio utiliza Terraform para elaborar uma infraestrutura na AWS, criando uma VPC com uma sub-rede pública, um gateway de internet e regras de segurança para acesso SSH. Além disso, ele elabora uma instância EC2 executando Debian 12, gera uma chave SSH para acesso seguro e exibe o IP público da máquina.

Linha 1 / provider "aws" {}
-O código define o provedor AWS, utilizando a região "us-east-1" para implementação;

Linha 5 / variable "projeto" {}
-A variável "projeto" armazena o nome do projeto, permitindo reutilização no código Terraform e utiliza "VExpenses" como padrão;

Linha 11 / variable "candidato" {}
-A variável candidato armazena o nome do candidato, permitindo personalizar recursos no Terraform. O valor padrão do código é "SeuNome";

Linha 17 / resource "tls_private_key" "ec2_key" {}
-O recurso tls_private_key "ec2_key" gera uma chave privada RSA de 2048 bits.

Linha 22 / resource "aws_key_pair" "ec2_key_pair" {}
-O recurso aws_key_pair cria um par de chaves na AWS, relacionado a chave pública de EC2. O nome da chave é gerado pelo ${var.projeto}-${var.candidato}-key, garantindo organização e a chave pública vem de tls_private_key.ec2_key.

Linha 27 / resource "aws_vpc" "main_vpc" {}
-O recurso aws_vpc "main_vpc" cria uma VPC na AWS, definindo a faixa de IPs (10.0.0.0/16), permitindo DNS e ativando nomes DNS. O nome é gerado com ${var.projeto}-${var.candidato}-vpc

Linha 37 / resource "aws_subnet" "main_subnet" {}
-O recurso aws_subnet cria uma sub-rede dentro da VPC, associada à aws_vpc.main_vpc, com intervalo de IPs 10.0.1.0/24 e localizada na zona us-east-1a. 

Linha 47 / resource "aws_internet_gateway" "main_igw" {}
-O recurso aws_internet_gateway "main_igw" cria um gateway de internet para permitir que recursos dentro da VPC acessem a internet.

Linha 55/ resource "aws_route_table" "main_route_table" {}
-Esse recurso define uma tabela de rotas associada à VPC, permitindo que todo o tráfego externo (0.0.0.0/0) seja encaminhado para o internet gateway, garantindo que as instâncias dentro da subnet pública tenham acesso à internet.

Linha 68 / resource "aws_route_table_association" "main_association" {}
-O aws_route_table_association garante que uma subnet use uma tabela de rotas específica dentro da VPC. Sem essa associação, a subnet pode usar a tabela errada ou perder conectividade com a internet e outros serviços.

Linha 77 / resource "aws_security_group" "main_sg" {}
-Esse resouce referente ao security group, controla o tráfego de rede para as instâncias dentro da VPC. Ele permite conexões SSH de qualquer IP, e de saída permitem todo o tráfego, garantindo que a instância possa se comunicar livremente com a internet.

Linha 107 / data "aws_ami" "debian12" {}
-Esse data source filtra as imagens pelo nome, garantindo que sejam versões de 64 bits, e pelo tipo de virtualização (hvm). Além disso, restringe as AMIs ao proprietário "679593333241".

Linha 123 / resource "aws_instance" "debian_ec2" {}
-Esse código cria uma instância EC2 baseada na AMI mais recente do Debian 12, do tipo t2.micro, em uma sub-rede pública com IP público. Usa uma chave SSH para acesso seguro e um grupo de segurança para controle de tráfego.

Linha 149 / output "private_key" {}
-O output "private_key" exibe a chave privada gerada para acessar a EC2. O valor vem de tls_private_key.ec2_key, e sensitive = true evita que a chave apareça no terminal, garantindo mais segurança.

Linhha 155 / output "ec2_public_ip" {}
-O output "ec2_public_ip" exibe o IP público da EC2 criada. Ele recupera o valor de aws_instance.debian_ec2.public_ip, permitindo fácil acesso ao servidor sem precisar buscar manualmente na AWS.

## Arquivo main.tf modificado
Localizado em outro arquivo do repositório para melhor vizualização.

## Descrição do Código - Tarefa 2
Adição da tag "managed-by": 
A tag "managed-by" = "terraform" foi adicionada aos recursos para indicar que são gerenciados pelo Terraform. Isso facilita a identificação e evita conflitos de alterações manuais.

Personalização da variável candidato: 
A variável "candidato", definida na linha 11, permite personalizar os recursos dentro do Terraform. O valor padrão era "SeuNome", mas foi atualizado para "MuriloLimadeOliveira"

Remoção da linha de tags em aws_route_table_association: 
Esse recurso não suporta tags, e sua remoção evita erros na execução do código.

Ouve separação das regras de entrada e saída do aws_security_group: 
Antes, as regras estavam dentro do próprio recurso aws_security_group. Agora, foram movidas para recursos individuais, permitindo um gerenciamento mais organizado. 

Adição de uma regra para tráfego HTTP na porta 80: 
Permite que a instância EC2 receba conexões HTTP, essencial para o funcionamento do Nginx. 

Modificação do user data: 
Agora, o script de inicialização da EC2 garante que o Nginx seja instalado e iniciado automaticamente ao provisionar a instância, melhorando a automação do ambiente.

Armazenamento seguro da chave privada: 
Foi adicionado o recurso local_file.private_key, que salva a chave privada gerada pelo Terraform em um arquivo local (${var.projeto}-${var.candidato}-key.pem). Isso garante que apenas o usuário proprietário possa acessá-la, aumentando a segurança do acesso à instância EC2.

## Instruções de Uso
Instale o Terraform em seu sistema.
Configure suas credenciais da AWS.
Clone este repositório e navegue até o diretório do projeto.
Execute terraform init para inicializar o Terraform.
Execute terraform apply e confirme a execução.
O IP público da instância será exibido na saída do Terraform.
