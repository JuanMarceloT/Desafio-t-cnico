# Descrição Técnica do Código Original

O arquivo `main.tf` configura a infraestrutura básica na AWS utilizando o Terraform. Aqui está o que cada recurso faz:

- **Provider AWS**: Define a região `us-east-1` para os recursos AWS.
- **Variáveis**: Define variáveis para o nome do projeto e do candidato, permitindo reutilização dos valores em outros recursos.
- **Chave Privada e Par de Chaves**: Gera uma chave privada RSA e cria um par de chaves AWS para acesso SSH à instância EC2.
- **VPC**: Cria uma VPC com um bloco CIDR de `10.0.0.0/16` e suporte DNS habilitado.
- **Subnet**: Cria uma subnet dentro da VPC com o bloco CIDR `10.0.1.0/24` na zona de disponibilidade `us-east-1a`.
- **Internet Gateway**: Cria um gateway de internet associado à VPC, permitindo tráfego de saída.
- **Tabela de Rotas**: Cria uma tabela de rotas que direciona todo o tráfego (`0.0.0.0/0`) para o internet gateway.
- **Associação da Tabela de Rotas**: Associa a tabela de rotas à subnet criada.
- **Security Group**: Cria um grupo de segurança permitindo SSH (porta 22) de qualquer lugar e todo o tráfego de saída.
- **AMI Debian**: Obtém a AMI mais recente do Debian 12 (64 bits) para criar a instância EC2.
- **Instância EC2**: Cria uma instância EC2 com a AMI Debian, tipo `t2.micro`, associada a um IP público e com uma chave SSH para acesso. A instância tem 20 GB de armazenamento e recebe atualizações do sistema via script no `user_data`.
- **Outputs**: Exibe a chave privada (para acesso à instância) e o IP público da instância EC2.

## Instruções de Uso

### Pré-requisitos

- **Terraform**: Certifique-se de que o Terraform esteja instalado no seu sistema. [Instruções de instalação](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli).
- **AWS CLI Configurada**: Tenha as credenciais da AWS configuradas em sua máquina. Use `aws configure` para definir `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY`.
- **Chaves SSH**: Se você estiver usando outro sistema além do Terraform para gerenciar chaves SSH, tenha isso em mente para acessar a instância EC2 posteriormente.

### Comandos

1. **Inicializar o Terraform**:

    ```bash
    terraform init
    ```

2. **Visualizar as alterações**:

    ```bash
    terraform plan
    ```

3. **Aplicar a configuração**:

    ```bash
    terraform apply
    ```

    Confirme a execução digitando `yes` quando solicitado.

4. **Acessar a instância EC2**:

    Use a chave privada exibida no output.

    ```bash
    ssh -i private_key.pem ec2-user@public_ip
    ```

---

# Desafio 2

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
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

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de um IP específico e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "Allow SSH from specific IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP_ADDRESS/32"]  # Substitua pelo seu IP
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
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
    encrypted             = true  # Modificação: criptografia do volume EBS
  }

  user_data = <<-EOF
            #!/bin/bash
            apt-get update -y
            apt-get upgrade -y
            apt-get install nginx -y
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
```

## Resumo das Modificações

- **Restrição de Acesso SSH**:  
  Alterado o bloco de `ingress` do grupo de segurança para permitir o acesso SSH apenas de um IP específico (antes era aberto a qualquer IP).

- **Criptografia do Volume EBS**:  
  Adicionada a linha `encrypted = true` no bloco `root_block_device` da instância EC2 para garantir que o volume de armazenamento esteja criptografado.

- **Automação da Instalação do Nginx**:  
  Adicionado o campo `user_data` na configuração da instância EC2 para automatizar a instalação e inicialização do servidor Nginx após o lançamento da instância.

Essas mudanças melhoram a segurança e automatizam a configuração do servidor web.
