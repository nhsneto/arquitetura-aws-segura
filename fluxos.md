# Documentação dos Fluxos de Comunicação e Regras de Tráfego

**Projeto:** Proposta de Arquitetura Cloud Segura (AWS)  
**Disciplina/Tema:** Segurança Cloud: IAM, Redes e Boas Práticas  
**Artefato:** Matriz e Especificação dos Fluxos de Comunicação (`fluxos.pdf` / `fluxos.md`)  
**Região AWS:** `sa-east-1` (São Paulo)  
**Ambiente:** Produção (`app-prod`)

---

## 1. Visão Geral da Arquitetura e Modelo de Comunicação

A arquitetura implementada foi desenhada com base no modelo de **Defesa em Profundidade** (*Defense-in-Depth*) e nos princípios do **Zero Trust** e **Menor Privilégio** (*Least Privilege*). Nenhum recurso interno possui endereço IP público ou rota direta de entrada a partir da Internet. 

Toda a infraestrutura de rede está confinada em uma **Virtual Private Cloud (VPC)** com bloco de endereçamento CIDR `10.0.0.0/16`, distribuída em duas Zonas de Disponibilidade (**AZ-a** e **AZ-b**) da região `sa-east-1` para prover alta disponibilidade e resiliência:

* **Sub-redes Públicas (`10.0.1.0/24` e `10.0.2.0/24`):** Isolam os pontos de contato externos controlados (Application Load Balancer) conectados ao Internet Gateway (IGW).
* **Sub-redes Privadas de Aplicação (`10.0.11.0/24` e `10.0.12.0/24`):** Hospedam os nós de processamento da API (instâncias Amazon EC2) e o ponto de terminação da VPN administrativa (AWS Client VPN). Sem rota direta para o IGW.
* **Sub-redes Privadas de Dados (`10.0.21.0/24` e `10.0.22.0/24`):** Abrigam a camada de persistência com Amazon RDS MySQL em configuração Multi-AZ (instância Primária na AZ-a e Standby síncrono na AZ-b). Isolamento total da Internet e das sub-redes públicas.
* **Serviços Gerenciados e Endpoints Privados:** O acesso aos serviços AWS (Amazon S3, AWS Secrets Manager, AWS KMS e Amazon CloudWatch Logs) ocorre 100% pelo backbone interno da AWS por meio de **VPC Endpoints** (Gateway e Interface/PrivateLink), eliminando a necessidade de saída para a internet pública (NAT Gateway).

---

## 2. Diagrama Conceitual dos Fluxos de Rede

Abaixo é apresentado o mapeamento dos canais de comunicação autorizados entre as diferentes zonas e componentes:

```
[ Usuários / Clientes Externos ]
              |
              | HTTPS (TCP 443)
              v
     [ Internet Gateway ]
              |
              | Inspeção L7
      [ AWS WAF Web ACL ]
              |
              v
[ Application Load Balancer (ALB) ] (Sub-redes Públicas)
              |
              | TCP 3000 (app-prod-alb-sg -> app-prod-api-sg)
              v
 [ API Backend - EC2 Cluster ] (Sub-redes Privadas APP)
       |                 |                   |
       | TCP 3306        | HTTPS 443         | HTTPS 443
       |                 | (Gateway EP)      | (Interface EP)
       v                 v                   v
[ Amazon RDS MySQL ]  [ Amazon S3 ]   [ Secrets Manager / KMS / Logs ]
  (Multi-AZ Dados)      (Uploads)          (AWS PrivateLink)
       ^
       | TCP 3306 (Admin sob demanda)
       |
[ AWS Client VPN ] <--- HTTPS / OpenVPN (TCP/UDP 443) + MFA --- [ Equipe Admin ]
       |
       +---> AWS SSM / SSH 22 (Acesso Administrativo à API)
```

---

## 3. Matriz Completa de Conexões Permitidas

A tabela abaixo documenta detalhadamente todos os fluxos de comunicação estritamente autorizados no ambiente, contendo a origem, destino, camada/mecanismo de controle, protocolo, porta e finalidade técnica e de segurança.

| ID | Origem | Destino | Protocolo | Porta | Mecanismo de Controle | Finalidade e Descrição |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **FL-01** | Internet (`0.0.0.0/0`) | Application Load Balancer (ALB) | TCP / HTTPS | `443` | `app-prod-alb-sg` + AWS WAF | **Acesso público seguro à aplicação:** Único ponto de entrada público. Tráfego TLS obrigatório (certificado ACM). O AWS WAF inspeciona contra SQLi, XSS, rate-limiting e bots maliciosos antes de encaminhar ao ALB. |
| **FL-02** | Internet (`0.0.0.0/0`) | Application Load Balancer (ALB) | TCP / HTTP | `80` | `app-prod-alb-sg` + Listener ALB | **Redirecionamento obrigatório:** A porta 80 aceita conexões apenas para emitir redirecionamento HTTP 301 imediato e permanente para HTTPS (443). Nenhuma requisição de aplicação é atendida em texto plano. |
| **FL-03** | ALB (`app-prod-alb-sg`) | Instâncias API EC2 (`app-prod-api-sg`) | TCP | `3000` | `app-prod-api-sg` (Ingress referenciando SG) | **Encaminhamento de tráfego web/API:** O ALB distribui as requisições autenticadas e inspecionadas para as instâncias EC2 nas AZs privadas. As instâncias aceitam conexões na porta 3000 **somente** se originadas pelo Security Group do ALB. |
| **FL-04** | Instâncias API EC2 (`app-prod-api-sg`) | Amazon RDS MySQL (`app-prod-db-sg`) | TCP | `3306` | `app-prod-db-sg` (Ingress referenciando SG) | **Acesso à base de dados relacional:** Comunicação direta da API com a instância Primária do RDS MySQL. O Security Group do RDS autoriza tráfego exclusivamente a partir de `app-prod-api-sg`. |
| **FL-05** | RDS MySQL Primário (AZ-a) | RDS MySQL Standby (AZ-b) | TCP | `3306` | Grupos de segurança internos do RDS | **Replicação síncrona Multi-AZ:** Mantém a réplica de standby sincronizada para failover automático sem perda de dados (RPO=0) e com alto RTO. |
| **FL-06** | Instâncias API EC2 (`app-prod-api-sg`) | VPC Gateway Endpoint S3 (`com.amazonaws.sa-east-1.s3`) | TCP / HTTPS | `443` | Tabela de rotas privadas + Bucket Policy S3 | **Armazenamento e recuperação de arquivos de usuários:** A API realiza uploads e downloads privados no bucket S3. O tráfego não sai para a internet pública, fluindo diretamente pelo barramento privado da AWS. |
| **FL-07** | Instâncias API EC2 (`app-prod-api-sg`) | Interface Endpoints AWS PrivateLink (`app-prod-vpce-sg`) | TCP / HTTPS | `443` | `app-prod-vpce-sg` + `app-prod-api-sg` (Egress) | **Recuperação de segredos e chaves criptográficas:** Conexão segura e privada da API para Secrets Manager (credenciais dinâmicas do banco) e AWS KMS (decriptação/cifragem de envelopes). |
| **FL-08** | Instâncias API EC2 / ALB / RDS | Amazon CloudWatch Logs & Metrics | TCP / HTTPS | `443` | Interface VPC Endpoints (`vpce-logs`) / AWS Backbone | **Envio de telemetria e logs de auditoria:** Coleta de logs de acesso do ALB, logs de erro/aplicação da API, logs de queries lentas do MySQL e métricas operacionais para monitoramento contínuo. |
| **FL-09** | Equipe Administrativa (Internet) | AWS Client VPN Endpoint | UDP / TCP | `443` (ou `1194`) | `app-prod-vpn-sg` + Autenticação Federada / MFA | **Túnel de acesso administrativo seguro:** Ponto único de entrada para manutenção. Exige autenticação multifator (MFA), validação de certificados e verificação de regras de autorização de rede do Client VPN. |
| **FL-10** | AWS Client VPN (`app-prod-vpn-sg`) | Instâncias API EC2 (`app-prod-api-sg`) | TCP / HTTPS | `443` (SSM) / `22` (SSH restrito) | `app-prod-api-sg` + IAM Policies | **Gerenciamento de instâncias:** Administração prioritária sem portas abertas via AWS Systems Manager (SSM Session Manager). Caso seja indispensável o SSH (22), este só aceita pacotes vindos do CIDR restrito da VPN administrativa. |
| **FL-11** | AWS Client VPN (`app-prod-vpn-sg`) | Amazon RDS MySQL (`app-prod-db-sg`) | TCP | `3306` | `app-prod-db-sg` (Regra condicional e restrita) | **Manutenção extraordinária de banco:** Conexão administrativa direta ao RDS sob demanda, estritamente monitorada, autenticada e liberada apenas para o grupo de administradores da VPN. |
| **FL-12** | Recursos da VPC (VPC Flow Logs) | CloudWatch Logs / S3 | HTTPS | `443` | IAM Service Role para VPC Flow Logs | **Registro de auditoria de pacotes de rede:** Captura metadados de tráfego de rede (IP origem, destino, portas, pacotes aceitos e rejeitados) em todas as interfaces elásticas da VPC. |

---

## 4. Matriz de Conexões Expressamente Bloqueadas

A postura de segurança adotada é orientada pelo princípio do **bloqueio padrão (*Default Deny*)**. Todas as conexões que não foram expressamente mapeadas acima são barradas. Abaixo estão listados os principais cenários de conexão bloqueados e os respectivos mecanismos de contenção:

| Tráfego / Vetor | Origem | Destino | Protocolo / Porta | Mecanismo de Bloqueio | Justificativa de Segurança e Risco Mitigado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Acesso direto da Internet à API** | Internet (`0.0.0.0/0`) | Instâncias EC2 da API | Qualquer porta (ex: `80`, `443`, `3000`, `22`) | Sub-redes Privadas (sem IP público, sem rota para IGW) e Security Group `app-prod-api-sg` | **Proteção contra ataques diretos:** As instâncias não possuem IPs públicos e as sub-redes privadas não contam com rotas para o Internet Gateway. Nenhuma ameaça externa consegue atingir os servidores de aplicação diretamente. |
| **Acesso direto da Internet ao Banco de Dados** | Internet (`0.0.0.0/0`) | Amazon RDS MySQL | TCP `3306` (MySQL) e demais | Flag `Publicly Accessible = False` no RDS + Sub-rede sem rota externa + SG `app-prod-db-sg` | **Prevenção total de vazamento de dados e ransomware:** A base de dados não é exposta na internet sob hipótese alguma. Todo acesso é mediado pela API ou via túnel administrativo autenticado. |
| **Acesso direto do ALB ao Banco de Dados** | ALB (`app-prod-alb-sg`) | Amazon RDS MySQL (`app-prod-db-sg`) | Qualquer porta | `app-prod-db-sg` (sem regra de Ingress para o ALB) + Isolamento entre sub-redes | **Isolamento de camadas (Tier Separation):** Impede que um comprometimento do balanceador ou vetor SSRF consiga atingir a camada de banco de dados diretamente. |
| **Acesso HTTP não cifrado à aplicação** | Clientes Externos | ALB / API Backend | TCP `80` (dados de aplicação) | Redirecionamento permanente no ALB (301 para 443) | **Proteção da integridade e confidencialidade:** Elimina risco de interceptação de dados, ataques Man-in-the-Middle (MitM) e furto de tokens ou senhas em trânsito. |
| **SSH administrativo aberto à Internet** | Internet (`0.0.0.0/0`) | EC2 API ou Bastion | TCP `22` | Security Group `app-prod-api-sg` + ausência de IP público | **Eliminação de ataques de força bruta e scan de vulnerabilidades:** A porta SSH nunca fica aberta para o mundo (`0.0.0.0/0`). O acesso só ocorre por túnel VPN com MFA ou via AWS SSM autenticado via IAM. |
| **Acesso público aos arquivos no S3** | Internet pública (`0.0.0.0/0`) | Bucket Amazon S3 de Uploads | HTTP / HTTPS | Amazon S3 Block Public Access (100% ativo) + S3 Bucket Policy restritiva | **Prevenção de vazamento acidental de arquivos:** O recurso S3 Block Public Access bloqueia todas as ACLs e políticas públicas. Apenas a IAM Role da API e o VPC Endpoint específico possuem concessão de acesso. |
| **Tráfego de saída irrestrito para a Internet** | Instâncias API EC2 | Internet pública | Qualquer protocolo / porta | Ausência de NAT Gateway e rota `0.0.0.0/0` nas tabelas de rota privadas | **Mitigação de comando e controle (C2) e exfiltração:** Servidores comprometidos não conseguem baixar malwares externos ou enviar dados para servidores de atacantes na internet. O acesso aos serviços AWS necessários é feito exclusivamente via VPC Endpoints internos. |
| **Tráfego lateral não autorizado entre instâncias da API** | Instância EC2 API A | Instância EC2 API B | Portas internas não utilizadas | Security Group `app-prod-api-sg` (Ingress restrito somente a portas e origens necessárias) | **Contenção de movimentação lateral:** Se uma instância for afetada, ela não consegue varrer ou atacar instâncias vizinhas desnecessariamente. |
| **Requisições maliciosas na camada de aplicação** | Atacantes na Internet | ALB | HTTP/HTTPS (L7 - SQLi, XSS, Bad Bots, Flood) | Regras ativas do AWS WAF Web ACL | **Filtragem de exploits:** O AWS WAF descarta pacotes de ataque antes que cheguem a consumir recursos do ALB ou da API backend. |

---

## 5. Especificação Técnica dos Grupos de Segurança (Security Groups)

Os Security Groups atuam como firewalls virtuais com estado (*stateful*) no nível de interface de rede elástica (ENI). Uma conexão de entrada permitida autoriza automaticamente a resposta de saída, e vice-versa.

### 5.1. `app-prod-alb-sg` (Application Load Balancer)
* **Localização:** Sub-redes Públicas A e B (`10.0.1.0/24`, `10.0.2.0/24`)
* **Regras de Entrada (Ingress):**
  * `Protocolo: TCP | Porta: 443 | Origem: 0.0.0.0/0` — Entrada HTTPS segura para todos os clientes legítimos.
  * `Protocolo: TCP | Porta: 80 | Origem: 0.0.0.0/0` — Entrada HTTP para redirecionamento obrigatório para HTTPS.
* **Regras de Saída (Egress):**
  * `Protocolo: TCP | Porta: 3000 | Destino: app-prod-api-sg` — Encaminhamento de requisições exclusivamente para as instâncias da API backend.

### 5.2. `app-prod-api-sg` (Instâncias EC2 da API)
* **Localização:** Sub-redes Privadas de Aplicação A e B (`10.0.11.0/24`, `10.0.12.0/24`)
* **Regras de Entrada (Ingress):**
  * `Protocolo: TCP | Porta: 3000 | Origem: app-prod-alb-sg` — Aceita tráfego da aplicação estritamente vindo do ALB.
  * `Protocolo: TCP | Porta: 22 | Origem: app-prod-vpn-sg` *(Opcional / Contingência)* — Acesso SSH permitido unicamente através do túnel da VPN interna.
* **Regras de Saída (Egress):**
  * `Protocolo: TCP | Porta: 3306 | Destino: app-prod-db-sg` — Conexão estrita para o RDS MySQL.
  * `Protocolo: TCP | Porta: 443 | Destino: app-prod-vpce-sg` — Comunicação com Interface Endpoints (Secrets Manager, KMS, CloudWatch Logs).
  * `Protocolo: TCP | Porta: 443 | Destino: Gateway Endpoint S3 (Prefix List AWS)` — Acesso privado aos buckets S3.

### 5.3. `app-prod-db-sg` (Amazon RDS MySQL)
* **Localização:** Sub-redes Privadas de Dados A e B (`10.0.21.0/24`, `10.0.22.0/24`)
* **Regras de Entrada (Ingress):**
  * `Protocolo: TCP | Porta: 3306 | Origem: app-prod-api-sg` — Conexão exclusiva das instâncias autorizadas da API backend.
  * `Protocolo: TCP | Porta: 3306 | Origem: app-prod-vpn-sg` *(Restrito/Administrativo)* — Acesso administrativo para manutenções programadas de banco via VPN.
* **Regras de Saída (Egress):**
  * *Padrão AWS:* Somente tráfego de resposta (stateful) ou tráfego interno de replicação Multi-AZ gerenciado pelo RDS.

### 5.4. `app-prod-vpce-sg` (Interface Endpoints - PrivateLink)
* **Localização:** Sub-redes Privadas de Aplicação
* **Regras de Entrada (Ingress):**
  * `Protocolo: TCP | Porta: 443 | Origem: app-prod-api-sg` — Permite que as instâncias da API consumam as APIs de Secrets Manager, CloudWatch Logs e KMS de forma privada.
* **Regras de Saída (Egress):**
  * Tráfego de retorno stateful para as ENIs privadas dos serviços AWS.

### 5.5. `app-prod-vpn-sg` (AWS Client VPN Endpoint)
* **Localização:** Associação de Sub-rede Privada
* **Regras de Entrada (Ingress):**
  * `Protocolo: UDP/TCP | Porta: 443 | Origem: 0.0.0.0/0` — Estabelecimento do túnel criptografado TLS/OpenVPN mediante autenticação multifator (MFA).
* **Regras de Saída (Egress):**
  * `Protocolo: TCP | Portas: 443, 22 | Destino: app-prod-api-sg` — Encaminhamento administrativo controlado para a camada de servidores.
  * `Protocolo: TCP | Portas: 3306 | Destino: app-prod-db-sg` — Encaminhamento para a camada de banco de dados para administradores autorizados.

---

## 6. Fluxos de Comunicação Fim a Fim (Detalhamento Paso a Passo)

### Fluxo 1: Requisição de Usuário Externo e Processamento Web (HTTPS)
1. O cliente inicia uma conexão HTTPS segura apontando para o domínio da aplicação via Internet.
2. O tráfego alcança o **Internet Gateway (IGW)** e é direcionado para o **Application Load Balancer (ALB)** nas sub-redes públicas.
3. O **AWS WAF** intercepta a requisição antes da camada de aplicação do ALB, verificando se há ataques comuns (injeção SQL, cross-site scripting, scanners maliciosos e limites de requisição por IP).
4. Sendo aprovada a inspeção, o ALB encerra a sessão TLS utilizando o certificado gerenciado pelo **AWS Certificate Manager (ACM)** e reencaminha a requisição via HTTP na porta interna `3000` para uma das instâncias EC2 ativas nas sub-redes privadas (`AZ-a` ou `AZ-b`).
5. O Security Group `app-prod-api-sg` aceita o pacote apenas porque o IP emissor pertence à interface de rede do `app-prod-alb-sg`.

### Fluxo 2: Comunicação com a Base de Dados Relacional
1. Durante o processamento da requisição, a aplicação EC2 necessita consultar ou persistir dados.
2. A API consulta o **AWS Secrets Manager** através do **Interface VPC Endpoint** para recuperar as credenciais de conexão do banco de dados (sem senhas em texto plano em arquivos de configuração).
3. A API abre uma conexão TCP na porta `3306` com o endpoint DNS do **Amazon RDS Multi-AZ**.
4. O tráfego trafega inteiramente pela rede privada da VPC até a sub-rede de dados (`10.0.21.0/24`).
5. O `app-prod-db-sg` valida a conexão, autentica o usuário do banco e processa a query.
6. De forma transparente e síncrona, as operações de escrita são replicadas pelo RDS entre a AZ-a e a AZ-b na porta `3306`.

### Fluxo 3: Upload e Armazenamento Seguro de Arquivos
1. O cliente envia um arquivo através da API.
2. A instância EC2 da API assume sua **IAM Role** de instância para obter credenciais temporárias de segurança (via STS).
3. A instância envia o arquivo ao bucket **Amazon S3** via chamada HTTPS na porta `443`.
4. A tabela de rotas da sub-rede privada intercepta o destino do S3 através da rota direcionada ao **Gateway VPC Endpoint** (`pl-xxxxxxx`).
5. O pacote flui diretamente pela infraestrutura da AWS, sem passar por NAT Gateway nem Internet Gateway.
6. O S3 avalia a **Bucket Policy**, validando que a requisição partiu do VPC Endpoint autorizado e pelo perfil IAM da aplicação.
7. O arquivo é armazenado no S3 com cifragem no lado do servidor ativada utilizando chaves KMS gerenciadas pelo cliente (**SSE-KMS**).

### Fluxo 4: Manutenção e Acesso Administrativo
1. O administrador de infraestrutura inicia o cliente de VPN a partir de sua estação de trabalho.
2. A conexão atinge o **AWS Client VPN Endpoint** na porta `443`, exigindo autenticação federada com **MFA obrigatório**.
3. Uma vez autenticado, o túnel é estabelecido e o administrador recebe um IP virtual do pool VPN (`app-prod-vpn-sg`).
4. Para manutenção nos servidores, o administrador utiliza o **AWS Systems Manager (Session Manager)** via console AWS ou CLI, estabelecendo uma sessão de terminal criptografada via HTTPS sem abrir portas de entrada nas EC2.
5. Caso necessite de acesso SSH direto de emergência, o tráfego na porta `22` é aceito pelo `app-prod-api-sg` unicamente porque o endereço IP de origem se encontra no túnel administrativo autenticado.

### Fluxo 5: Auditoria, Monitoramento e Trilha de Segurança
1. Todas as chamadas de API executadas na conta (seja pelo console, CLI ou automação) são registradas pelo **AWS CloudTrail**, gerando trilhas de auditoria criptografadas com KMS e salvas em bucket de logs com controle de integridade.
2. As métricas de uso de CPU, memória, latência do ALB e códigos HTTP de erro são enviadas para o **Amazon CloudWatch**.
3. O **VPC Flow Logs** monitora e registra todo o tráfego aceito e rejeitado em todas as interfaces da VPC, permitindo a detecção de tentativas de varredura de portas ou conexões anômalas.

---

## 7. Roteamento de Sub-redes e Isolamento de Rede

| Tipo de Sub-rede | CIDRs | Associação de Rota Padrão (`0.0.0.0/0`) | Acesso à Internet | Serviços Contidos |
| :--- | :--- | :--- | :--- | :--- |
| **Sub-redes Públicas** | `10.0.1.0/24` (AZ-a)<br>`10.0.2.0/24` (AZ-b) | Direcionada para `igw-xxxxxxxx` (Internet Gateway) | Bidirecional (Entrada e Saída) | Application Load Balancer (ALB) |
| **Sub-redes Privadas APP** | `10.0.11.0/24` (AZ-a)<br>`10.0.12.0/24` (AZ-b) | **Nenhuma rota para a Internet** (Sem IGW e Sem NAT GW) | Apenas Entrada via ALB e Saída via VPC Endpoints | Instâncias EC2 da API Backend, Client VPN Endpoint |
| **Sub-redes Privadas DADOS**| `10.0.21.0/24` (AZ-a)<br>`10.0.22.0/24` (AZ-b) | **Isolamento Total** (Roteamento estritamente local `10.0.0.0/16`) | Nenhum acesso externo | Amazon RDS MySQL Primário e Standby Multi-AZ |

---

## 8. Resumo de Conformidade com os Requisitos do Projeto

* [x] **Acesso Seguro via HTTPS (Req. 1):** ALB com certificado ACM, término TLS e redirecionamento de HTTP (80) para HTTPS (443).
* [x] **Proteção de Tráfego e Distribuição (Req. 2):** AWS WAF associado ao ALB para proteção contra ameaças da camada de aplicação e balanceamento de carga entre zonas.
* [x] **Processamento Seguro pela API (Req. 3):** Instâncias EC2 em sub-redes privadas sem IP público, consumindo serviços AWS via VPC Endpoints internos.
* [x] **Banco de Dados Isolado (Req. 4):** RDS MySQL em sub-redes privadas dedicadas, sem rota pública e acessível unicamente pela API na porta 3306.
* [x] **Armazenamento de Arquivos com Acesso Controlado (Req. 5):** Bucket Amazon S3 com Block Public Access total, SSE-KMS, acessado privadamente via Gateway Endpoint.
* [x] **Acesso Administrativo Seguro (Req. 6):** AWS Client VPN com MFA obrigatório e AWS Systems Manager Session Manager (acesso seguro sem chave SSH exposta).
* [x] **Princípio do Menor Privilégio e Comunicação Mínima (Req. 7 e 9):** Security Groups restritos por ID de grupo e portas estritas; IAM Roles para serviços sem credenciais fixadas em código.
* [x] **Auditoria e Monitoramento (Req. 8):** CloudWatch Logs, Métricas e Alarmes, CloudTrail para governança e VPC Flow Logs para auditoria de conexões.

