# Infraestrutura AWS com Terraform
Este desafio utiliza Terraform para provisionar uma infraestrutura na AWS, criando uma VPC com uma sub-rede pública, um gateway de internet e regras de segurança para acesso SSH. Além disso, ele provisiona uma instância EC2 executando Debian 12, gera uma chave SSH para acesso seguro e exibe o IP público da máquina.
## Descrição do Código - Tarefa 1
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
provider "aws" {
  region = "us-east-1"
#Adicionado "managed-by"
  default_tags {
    tags = {
      managed-by = "terraform"
    }
  }
}
variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}
variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "MuriloLimadeOliveira"
}
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id
  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }
  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id
  
  #Retirada da tags
}
#Security Group principal
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH e tráfego HTTP"
  vpc_id      = aws_vpc.main_vpc.id
  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
#Regra de entrada para SSH
resource "aws_vpc_security_group_ingress_rule" "ssh" {
  security_group_id = aws_security_group.main_sg.id
  description       = "Allow SSH from anywhere"
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
  cidr_ipv4         = "0.0.0.0/0"
}
#Regra de entrada para HTTP(Nginx)
resource "aws_vpc_security_group_ingress_rule" "http" {
  security_group_id = aws_security_group.main_sg.id
  description       = "Allow HTTP traffic"
  from_port         = 80
  to_port           = 80
  ip_protocol       = "tcp"
  cidr_ipv4         = "0.0.0.0/0"
}
#Regra de saída permitindo todo tráfego
resource "aws_vpc_security_group_egress_rule" "outbound" {
  security_group_id = aws_security_group.main_sg.id
  description       = "Allow all outbound traffic"
  from_port         = 0
  to_port           = 0
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"
}
data "aws_ami" "debian12" {
  most_recent = true
  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  owners = ["679593333241"]
}
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  #User data configurada para o servidor Nginx
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install -y nginx
              systemctl start nginx
              systemctl enable nginx
              EOF
  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}
output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}

## Descrição do Código - Tarefa 2
Adição da tag "managed-by": 
A tag "managed-by" = "terraform" foi adicionada aos recursos para indicar que são gerenciados pelo Terraform. Isso facilita a identificação e evita conflitos de alterações manuais.
Personalização por variável: 
A variável "candidato", definida na linha 11, permite personalizar os recursos dentro do Terraform. O valor padrão era "SeuNome", mas foi atualizado para "MuriloLimadeOliveira"
Ouve separação das regras de entrada e saída do aws_security_group: 
Antes, as regras estavam dentro do próprio recurso aws_security_group. Agora, foram movidas para recursos individuais, permitindo um gerenciamento mais organizado. 
Remoção da linha de tags em aws_route_table_association: Esse recurso não suporta tags, e sua remoção evita erros na execução do código. 
Adição de uma regra para tráfego HTTP na porta 80: 
Permite que a instância EC2 receba conexões HTTP, essencial para o funcionamento do Nginx. 
Modificação do user data: Agora, o script de inicialização da EC2 garante que o Nginx seja instalado e iniciado automaticamente ao provisionar a instância, melhorando a automação do ambiente.
