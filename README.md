# Migração de um Workload rodando em um Data Center Corporativo para a AWS utilizando o serviço do Amazon EC2 e RDS
Tutorial de Migração: Aplicação On-Premise para AWS.



### **Parte 1: Implementação "Criação EC2 e RDS de acordo 'sizing'"**

#### **1.1. Criar VPC e Subnets:**
   - **Ação:** Crie uma VPC.
     - **Configuração:**
       - Selecione "VPC Only".
       - Nome (Name tag): `vpc-minha-aplicacao`
       - CIDR: `10.0.0.0/16` (Garanta que não haja sobreposição com sua rede on-premise).

#### **1.2. Criar Subnets Pública e Privadas:**
   - **Ação:** Crie as seguintes subnets dentro da `vpc-minha-aplicacao`.
     - **Public Subnet:**
       - Nome: `Public Subnet`
       - Zona de Disponibilidade (AZ): `us-east-1a`
       - CIDR: `10.0.0.0/24`
     - **Private Subnet 1:**
       - Nome: `Private Subnet 1`
       - AZ: `us-east-1a`
       - CIDR: `10.0.1.0/24`
     - **Private Subnet 2:**
       - Nome: `Private Subnet 2`
       - AZ: `us-east-1b`
       - CIDR: `10.0.2.0/24`
     - *Nota: Duas subnets privadas em AZs diferentes são necessárias para a alta disponibilidade do RDS.*

![image](https://github.com/user-attachments/assets/ddc11398-5923-4817-b86c-3a4d61314d4b)



#### **1.3. Criar Instância EC2 (Servidor da Aplicação):**
   - **Ação:** Lance uma nova instância EC2.
     - **Configuração:**
       - Nome: `awsuse1app01`
       - Sistema Operacional: Ubuntu `22.04 LTS`
       - Tipo de Instância: `t2.micro`
       - Par de Chaves (Key Pair): `ssh-aws-minha-aplicacao` (Crie uma pasta `aws-mod03` e salve a chave .pem nela).
       - Configurações de Rede:
         - VPC: `vpc-minha-aplicacao`
         - Subnet: `Public Subnet`
         - Atribuir IP público automaticamente: `Enable`
         - Firewall (Security Groups): Crie um novo chamado `app01-sg` permitindo tráfego nas portas `22` (SSH) e `8080` (Aplicação).

![image](https://github.com/user-attachments/assets/80ad76bd-dc30-4173-b475-7132f510ced2)


![image](https://github.com/user-attachments/assets/2aa30fbc-9c0c-45a4-81be-c983cd9c8a82)



#### **1.4. Criar Banco de Dados RDS (MySQL):**
   - **Ação:** Crie uma nova instância de banco de dados RDS.
     - **Configuração:**
       - Selecione "Standard create".
       - Motor: MySQL, Versão: `MySQL 8.0.35` (ou a mais próxima disponível da `8.0.xx`).
       - Modelo (Template): `Free Tier`.
       - Identificador da Instância de BD: `awsuse1db01`.
       - Credenciais: Usuário `admin`, Senha `admin123456`.
       - Classe da Instância de BD: `db.t2.micro` (ou `db.t3.micro` se `t2.micro` não estiver disponível).
       - Conectividade:
         - **Selecione: (*) Don’t connect to an EC2 compute resource.**
         - VPC: `vpc-minha-aplicacao`.
         - Grupo de Segurança da VPC: Mantenha o `default` por enquanto.
         - Zona de Disponibilidade (AZ): `us-east-1a`.
         - Porta do Banco de Dados: `3306` (verifique em "Additional configuration").

![image](https://github.com/user-attachments/assets/5ad350e2-ad02-4f68-9d5c-5562f42590da)


---

### **Parte 2: Instalação e Configuração dos Pacotes para App e Conexão com o BD**

#### **2.1. Configurar Instância (Ambiente de Desenvolvimento):**
   - **Ação:** Redimensione o volume EBS da instância para `30 GB`.
     - Verifique o tamanho atual: `df -kh` e `lsblk`.
     - Acesse o console EC2, encontre o volume EBS da instância e modifique seu tamanho para `30 GB`.
     - Verifique novamente: `df -kh` e `lsblk`. Se não atualizado, reinicie a instância: `sudo reboot`.

#### **2.2. Preparar Conexão com a Instância EC2:**
   - **Ação:** No terminal cloud shell, faça upload do arquivo da chave `ssh-aws-minha-aplicacao.pem`.
   - **Ação:** Crie um Internet Gateway (IGW) e configure o roteamento.
     - No console VPC, crie um Internet Gateway:
       - Nome: `igw-use1`.
       - Anexe-o à VPC `vpc-minha-aplicacao`.
     - Nas Tabelas de Roteamento (Route Table) da sua VPC (a principal, ou a associada à subnet pública):
       - Edite as rotas.
       - Adicione uma rota: Destino `0.0.0.0/0`, Alvo (Target) `Internet Gateway (igw-use1)`.
   - **Ação:** Tente conectar-se à instância EC2 via SSH.
     - Comando: `ssh -i ssh-aws-minha-aplicacao.pem ubuntu@YOUR-EC2-PUBLIC-IP` (Substitua `YOUR-EC2-PUBLIC-IP` pelo IP público da sua EC2).
     - Se receber um aviso "UNPROTECTED PRIVATE KEY FILE!":
       - Solução: `chmod 400 ssh-aws-minha-aplicacao.pem`.

![image](https://github.com/user-attachments/assets/de86740f-9265-4cf0-9785-55e74463c7e8)



#### **2.3. Instalar Aplicação e Suas Dependências na Instância EC2:**
  
   - **Ação:** Instale os pacotes e bibliotecas necessários.
     ```bash
     sudo apt update
     sudo apt install python3-dev -y
     sudo apt install python3-pip -y
     sudo apt install build-essential libssl-dev libffi-dev -y
     sudo apt install libmysqlclient-dev -y
     sudo apt install unzip -y
     sudo apt install libpq-dev libxml2-dev libxslt1-dev libldap2-dev -y
     sudo apt install libsasl2-dev libffi-dev -y
     sudo apt install python3.12-venv
     python3 -m venv .wiki
     source .wiki/bin/activate
     pip install Flask==2.3.3
     export PATH=$PATH:/home/ubuntu/.local/bin/ # Adicione ao seu .bashrc para persistência
     pip3 install wtforms
     sudo apt install pkg-config
     python -m pip install Flask-MySQLdb==0.2.0
     pip3 install passlib
     ```
   - **Ação:** Instale o MySQL Client na EC2.
     ```bash
     sudo apt-get install mysql-client -y
     ```

---

![image](https://github.com/user-attachments/assets/17f6b3ee-65da-403f-8e10-8906e5eec1af)



### **Parte 3: Go Live**

#### **3.1. Configurar Segurança para o RDS:**
   - **Ação:** Crie um novo Security Group (SG) para permitir o acesso da EC2 ao RDS.
     - No console VPC > Security Groups:
       - Nome: `EC2toRDS-sg`.
       - Descrição: `SG para permitir acesso ao MySQL atraves da aplicacao rodando no EC2.`
       - VPC: `vpc-minha-aplicacao`.
       - Regras de Entrada (Inbound rules):
         - Tipo: `MYSQL/Aurora`.
         - Origem (Source): IP Privado da sua instância EC2 ou o SG da EC2 (`app01-sg`). *Usar `0.0.0.0/0` é inseguro para produção, mas pode ser usado temporariamente para teste se o IP privado/SG não funcionar de imediato.*
   - **Ação:** Associe o novo SG (`EC2toRDS-sg`) à instância RDS (`awsuse1db01`).
     - No console RDS, selecione a instância `awsuse1db01` e clique em "Modify".
     - Na seção "Connectivity", adicione o `EC2toRDS-sg` e remova o `default` (se ainda estiver lá).
     - Continue e aplique as modificações imediatamente ("Apply immediately").
     
![image](https://github.com/user-attachments/assets/39be585e-763c-406a-a43a-e0b04def1118)



#### **3.2. Fazer Download dos Arquivos da Aplicação e Dump do Banco de Dados na EC2:**
   - **Ação:** Conectado à sua instância EC2 via SSH.
     ```bash
     wget https://github.com/phillrog/migracao-onpremise-aws/raw/refs/heads/main/files/wikiapp.zip
     wget https://raw.githubusercontent.com/phillrog/migracao-onpremise-aws/refs/heads/main/files/dump.sql
     ```

#### **3.3. Conectar ao Servidor MySQL no RDS e Importar Dados:**
   - **Ação:** Copie o Endpoint do seu RDS (disponível no console RDS).
   - **Ação:** Conecte-se ao RDS a partir da EC2.
     ```bash
     mysql -h <endpoint_do_rds> -P 3306 -u admin -p
     # Exemplo: mysql -h awsuse1db01.c49c0kiwyqwt.us-east-1.rds.amazonaws.com -P 3306 -u admin -p
     ```
     - Senha: `admin123456`.
   - **Ação:** Dentro do prompt do MySQL, crie o banco de dados e importe o dump.
     ```sql
     SHOW DATABASES;
     CREATE DATABASE wikidb;
     SHOW DATABASES;
     USE wikidb;
     SHOW TABLES;
     SOURCE /home/ubuntu/dump.sql;
     SHOW TABLES;
     SELECT * FROM articles;
     ```
     ou 
     ```
     SHOW TABLES;
      mysql -h awsuse1db01.c49c0kiwyqwt.us-east-1.rds.amazonaws.com -P 3306 -u  admin -p wikidb < dump.sql 
     SHOW TABLES;
     ```
   - **Ação:** Crie um usuário específico para a aplicação no banco de dados.
     ```sql
     CREATE USER 'wiki'@'%' IDENTIFIED BY 'admin123456';
     GRANT ALL PRIVILEGES ON wikidb.* TO 'wiki'@'%';
     FLUSH PRIVILEGES;
     EXIT;
     ```
     - mysql -h endpoint_do_banco -P 3306 -u  admin -p
     
![image](https://github.com/user-attachments/assets/fbfd9019-5979-4c52-b3f8-63e4fc2d4a4e)


![image](https://github.com/user-attachments/assets/ac6d2f70-ab53-4359-96bd-efd2121411b4)



#### **3.4. Configurar e Executar a Aplicação:**
   - **Ação:** Descompacte os arquivos da aplicação.
     ```bash
     unzip wikiapp.zip
     ```
   - **Ação:** Edite o arquivo de configuração da aplicação para apontar para o RDS.
     ```bash
     cd wikiapp/
     vi wiki.py
     ```
     - Pressione `INSERT` para editar.
     - Na seção `# Config MySQL`:
       - Altere `MYSQL_HOST` para o endpoint do seu RDS (ex: `'awsuse1db01.c49c0kiwyqwt.us-east-1.rds.amazonaws.com'`).
       - Altere `MYSQL_USER` para `'wiki'`.
     - Pressione `ESC`, depois digite `:x` e pressione Enter para salvar e sair.
   - **Ação:** Execute a aplicação.
     ```bash
     python3 wiki.py
     ```
![image](https://github.com/user-attachments/assets/e29fb3f4-9de4-4dee-976b-383b67719e2d)


![image](https://github.com/user-attachments/assets/5d139a74-ec8e-4305-b566-f77bc65278a7)



#### **3.5. Validar a Migração:**
   - **Ação:** Abra um navegador e acesse `http://<IP_PUBLICO_EC2>:8080`.
     - Use as credenciais de login: `admin` / `admin`.
   - **Ação:** Crie um novo artigo com seu nome como evidência.

---
Obs: É no procotolo http não é https.

![image](https://github.com/user-attachments/assets/7fbad868-31b2-4c26-bdf7-2c317996cb88)


### **Pós Go Live!**
- **Ação:** Monitore a estabilização da aplicação.
- **Ação:** Considere configurar suporte contínuo e otimizações.

---



## **Resultado da migração**

![image](https://github.com/user-attachments/assets/0fc7bd3d-8edc-43e6-84cf-3adbdfb8930b)


![image](https://github.com/user-attachments/assets/fd6d9fdb-ab9c-4a11-9621-7e0742a59950)

# FIM

### **Limpeza dos Recursos (IMPORTANTE):**
- **Ação:** Pare a instância.
- **Ação:** Exclua a instância RDS.
  - No console RDS, selecione a instância, clique em "Actions" > "Delete".
  - Desmarque a criação de snapshot final e a retenção de backups automáticos. Confirme a exclusão.
- **Ação:** Termine a instância EC2.
  - No console EC2, selecione a instância, clique em "Instance state" > "Terminate".

---

![image](https://github.com/user-attachments/assets/5d54e9d6-9ba2-408b-a801-aedb404a4f4e)


![image](https://github.com/user-attachments/assets/7bcb2490-c286-4728-a019-7c045f7f31d8)

