# EAD Microservices Architecture

Este projeto é uma plataforma de EAD baseada em uma arquitetura de microsserviços moderna, utilizando o ecossistema Spring Cloud e Docker para infraestrutura e monitoramento.

## 🛠 Tecnologias e Versões
- **Linguagem**: Java 21
- **Framework**: Spring Boot 3.5.7
- **Cloud**: Spring Cloud 2025.0.0
- **Bancos de Dados**: PostgreSQL 15 (Isolados por serviço)
- **Mensageria**: RabbitMQ (Isolados por serviço)
- **Service Discovery**: Netflix Eureka
- **Configuração Centralizada**: Spring Cloud Config
- **Logs Centralizados**: ELK Stack (Elasticsearch 7.17, Logstash, Kibana)
- **API Gateway**: Spring Cloud Gateway

## 📌 Arquitetura e Portas

### Serviços Core
| Serviço | Porta | Descrição |
| :--- | :--- | :--- |
| **Eureka Server** | 8761 | Registro e Descoberta de Serviços |
| **Config Server** | 8888 | Servidor de Configurações Centralizado |
| **API Gateway** | 8080 | Ponto de entrada único e roteamento |
| **AuthUser Service** | 8087 | Gestão de usuários e autenticação |
| **Course Service** | 8082 | Gestão de cursos e módulos |
| **Notification Service** | 8084 | Gestão de notificações via RabbitMQ |

### Infraestrutura (Containers)
| Componente | Portas | Serviço Relacionado |
| :--- | :--- | :--- |
| **PostgreSQL (Auth)** | 5432 | AuthUser Service |
| **PostgreSQL (Course)** | 5433 | Course Service |
| **PostgreSQL (Notif)** | 5434 | Notification Service |
| **RabbitMQ (Auth)** | 5672, 15672 | AuthUser Service |
| **RabbitMQ (Course)** | 5673, 15673 | Course Service |
| **RabbitMQ (Notif)** | 5674, 15674 | Notification Service |
| **Elasticsearch** | 9200 | ELK Stack (Logs) |
| **Kibana** | 5601 | ELK Stack (Visualização) |

---

## 🚀 Passo a Passo para Execução

### 1. Pré-requisitos
Certifique-se de ter instalado em sua máquina:
- **JDK 21**
- **Maven 3.9+**
- **Docker e Docker Compose**

### 2. Preparação da Infraestrutura
Antes de subir os microsserviços, é necessário iniciar os bancos de dados, brokers e a stack de logs.

```bash
# Na raiz do projeto, execute os comandos para subir cada infraestrutura:

# 1. Bancos e RabbitMQ do AuthUser
cd authuser && docker-compose up -d && cd ..

# 2. Bancos e RabbitMQ do Course
cd course && docker-compose up -d && cd ..

# 3. Bancos e RabbitMQ do Notification
cd notification && docker-compose up -d && cd ..

# 4. ELK Stack (Elasticsearch, Logstash, Kibana) na raiz
docker-compose -f docker-compose-elk.yml up -d
```

### 3. Compilação dos Projetos
Execute o comando Maven na raiz de **cada diretório** ou use um script de automação:

```bash
# Comandos individuais por diretório:
mvn clean package -DskipTests
```
*Certifique-se de que os arquivos `.jar` foram gerados na pasta `target` de cada módulo.*

### 4. Inicialização dos Microsserviços (ORDEM OBRIGATÓRIA)
Devido às dependências de descoberta e configuração, os serviços devem ser iniciados na seguinte ordem:

#### A. Serviços de Infraestrutura Spring
1. **Eureka Server**:
   ```bash
   cd eureka-server && java -jar target/eureka-server-0.0.1-SNAPSHOT.jar
   ```
2. **Config Server**: (Aguarde o Eureka estar online em http://localhost:8761)
   ```bash
   cd config-server && java -jar target/config-server-0.0.1-SNAPSHOT.jar
   ```

#### B. Microsserviços de Negócio
3. **AuthUser Service**:
   ```bash
   cd authuser && java -jar target/authuser-0.0.1-SNAPSHOT.jar
   ```
4. **Course Service**:
   ```bash
   cd course && java -jar target/course-0.0.1-SNAPSHOT.jar
   ```
5. **Notification Service**:
   ```bash
   cd notification && java -jar target/notification-0.0.1-SNAPSHOT.jar
   ```

#### C. Gateway
6. **API Gateway**:
   ```bash
   cd api-gateway && java -jar target/api-gateway-0.0.1-SNAPSHOT.jar
   ```

---

## 🔍 Verificação e Monitoramento

### Dashboards Disponíveis
- **Discovery (Eureka)**: [http://localhost:8761](http://localhost:8761) - Verifique se todos os serviços aparecem como `UP`.
- **Configurações Centralizadas**: [http://localhost:8888/ead-authuser-service/default](http://localhost:8888/ead-authuser-service/default)
- **Logs (Kibana)**: [http://localhost:5601](http://localhost:5601) - Crie um "Index Pattern" como `ead-logs-*` para visualizar os logs.
- **Elasticsearch Stats**: [http://localhost:9200](http://localhost:9200)

### RabbitMQ Management
- **AuthUser**: [http://localhost:15672](http://localhost:15672) (guest/guest)
- **Course**: [http://localhost:15673](http://localhost:15673) (guest/guest)
- **Notification**: [http://localhost:15674](http://localhost:15674) (guest/guest)

---

## 🛠 Rotas de API (Via Gateway)
Todas as requisições devem ser feitas através do Gateway na porta **8080**:
- **AuthUser**: `http://localhost:8080/ead-authuser/**`
- **Course**: `http://localhost:8080/ead-course/**`
- **Notification**: `http://localhost:8080/ead-notification/**`

---

## 📝 Notas de Versão (Principais Mudanças)
- **Modernização**: Atualização completa de Java 11 para **Java 21**.
- **Segurança**: Migração para Spring Security 6.x e JJWT 0.12.6.
- **Jakarta EE**: Migração de `javax.*` para `jakarta.*`.
- **Logs**: Implementação de Log4j2 com suporte a envio para Logstash via TCP/UDP.
- **Config**: Implementação de Config Server para centralização de segredos e propriedades.