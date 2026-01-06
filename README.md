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
- **Logs Centralizados**: ELK Stack (Elasticsearch 7.17, Logstash, Kibana) + Logstash Encoder
- **API Gateway**: Spring Cloud Gateway

## 📌 Arquitetura e Portas

### Serviços Core
| Serviço | Porta | Descrição |
| :--- | :--- | :--- |
| **Config Server** | 8888 | Configuração Centralizada |
| **Eureka Server** | 8761 | Registro e Descoberta de Serviços |
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

### Infraestrutura Opcional (ELK Stack)
| Componente | Portas | Descrição |
| :--- | :--- | :--- |
| **Elasticsearch** | 9200 | Armazenamento de logs |
| **Logstash** | 5044 | Processamento de logs |
| **Kibana** | 5601 | Visualização de logs |

---

## 🚀 Passo a Passo para Execução

### 1. Pré-requisitos
Certifique-se de ter instalado em sua máquina:
- **JDK 21**
- **Maven 3.9+**
- **Docker e Docker Compose**

### 2. Configuração de Variáveis de Ambiente
**IMPORTANTE**: Configure a variável de ambiente JWT antes de iniciar os serviços:

```bash
# Windows (CMD)
set EAD_AUTH_JWTSECRET=your-secret-key-here-minimum-256-bits

# Windows (PowerShell)
$env:EAD_AUTH_JWTSECRET="your-secret-key-here-minimum-256-bits"

# Linux/Mac
export EAD_AUTH_JWTSECRET="your-secret-key-here-minimum-256-bits"
```

*Ou copie `.env.example` para `.env` e configure as variáveis.*

### 3. Preparação da Infraestrutura
Antes de subir os microsserviços, é necessário iniciar os bancos de dados e brokers de cada serviço.

```bash
# Na raiz do projeto, execute os comandos para subir cada infraestrutura:

# 1. Bancos e RabbitMQ do AuthUser
cd authuser && docker-compose up -d && cd ..

# 2. Bancos e RabbitMQ do Course
cd course && docker-compose up -d && cd ..

# 3. Bancos e RabbitMQ do Notification
cd notification && docker-compose up -d && cd ..

# 4. (Opcional) ELK Stack para logs centralizados
# IMPORTANTE: Deve ser iniciado ANTES dos microsserviços para receber logs
docker-compose -f docker-compose-elk.yml up -d
```

### 4. Compilação dos Projetos
Execute o comando Maven na raiz de **cada diretório** ou use um script de automação:

```bash
# Comandos individuais por diretório:
mvn clean package -DskipTests
```
*Certifique-se de que os arquivos `.jar` foram gerados na pasta `target` de cada módulo.*

### 5. Inicialização dos Microsserviços (ORDEM OBRIGATÓRIA)

Para que o sistema funcione corretamente, a ordem de inicialização é crítica devido às dependências entre os serviços.

**Siga rigorosamente esta ordem:**

#### 1º Passo: Infraestrutura (Bancos e Brokers)
Antes de tudo, a infraestrutura deve estar rodando (conforme item 2):
- Bancos de Dados PostgreSQL
- RabbitMQ

#### 2º Passo: Config Server (Opcional)
Responsável pela configuração centralizada. Deve ser iniciado antes dos demais serviços se você quiser usar configurações centralizadas.
```bash
cd config-server && java -jar target/config-server-0.0.1-SNAPSHOT.jar
```
*Aguarde o log "Started ConfigServerApplication" e acesse http://localhost:8888.*

#### 3º Passo: Eureka Server
Responsável pelo Service Discovery.
```bash
cd eureka-server && java -jar target/eureka-server-0.0.1-SNAPSHOT.jar
```
*Aguarde o log "Started EurekaServerApplication" e acesse http://localhost:8761.*

#### 4º Passo: Microsserviços de Negócio
Estes serviços dependem do **Eureka** para descoberta. Podem ser iniciados simultaneamente após o Eureka estar pronto.

- **AuthUser Service**:
  ```bash
  cd authuser && java -jar target/authuser-0.0.1-SNAPSHOT.jar
  ```
- **Course Service**:
  ```bash
  cd course && java -jar target/course-0.0.1-SNAPSHOT.jar
  ```
- **Notification Service**:
  ```bash
  cd notification && java -jar target/notification-0.0.1-SNAPSHOT.jar
  ```

#### 5º Passo: API Gateway
Ponto de entrada único. Deve ser o último para garantir que todas as rotas dos serviços acima já estejam registradas no Eureka.
```bash
cd api-gateway && java -jar target/api-gateway-0.0.1-SNAPSHOT.jar
```

---

## 🔍 Verificação e Monitoramento

### Dashboards Disponíveis
- **Discovery (Eureka)**: [http://localhost:8761](http://localhost:8761) - Verifique se todos os serviços aparecem como `UP`.

### RabbitMQ Management
- **AuthUser**: [http://localhost:15672](http://localhost:15672) (guest/guest)
- **Course**: [http://localhost:15673](http://localhost:15673) (guest/guest)
- **Notification**: [http://localhost:15674](http://localhost:15674) (guest/guest)

### ELK Stack - Logs Centralizados
- **Elasticsearch**: [http://localhost:9200](http://localhost:9200) - API do Elasticsearch
- **Kibana**: [http://localhost:5601](http://localhost:5601) - Dashboard de logs
  - *Crie um Index Pattern como `ead-logs-*` para visualizar os logs*
  - **Logs Estruturados**: Todos os microsserviços enviam logs JSON com identificação de serviço
  - **Rastreamento**: Logs incluem timestamp, nível, logger e contexto MDC para correlação

---

## 🛠 Rotas de API (Via Gateway)
Todas as requisições devem ser feitas através do Gateway na porta **8080**:
- **AuthUser**: `http://localhost:8080/ead-authuser/**`
- **Course**: `http://localhost:8080/ead-course/**`
- **Notification**: `http://localhost:8080/ead-notification/**`

---

## 📊 Integração ELK Stack - Logs Centralizados

### Funcionalidades Implementadas

#### 1. **Elasticsearch como Motor de Busca**
- Indexação automática de logs de todos os microsserviços
- Índices organizados por data: `ead-logs-YYYY.MM.dd`
- Otimizado para buscas rápidas em grandes volumes de logs

#### 2. **Logstash para Processamento**
- Recebe logs via TCP na porta 5000
- Processa e estrutura logs em formato JSON
- Envia automaticamente para Elasticsearch

#### 3. **Logs Estruturados por Microsserviço**
Cada serviço envia logs com identificação única:
- **AuthUser**: `{"service": "ead-authuser-service"}`
- **Course**: `{"service": "ead-course-service"}`
- **Notification**: `{"service": "ead-notification-service"}`
- **API Gateway**: `{"service": "ead-api-gateway"}`

#### 4. **Rastreamento e Correlação**
- Timestamp preciso de cada evento
- Nível de log (INFO, DEBUG, ERROR)
- Contexto MDC para rastreamento entre serviços
- Logger name para identificação da origem

### Como Usar o Kibana
1. Acesse [http://localhost:5601](http://localhost:5601)
2. Vá em **Stack Management > Index Patterns**
3. Crie um pattern: `ead-logs-*`
4. Selecione `@timestamp` como campo de tempo
5. Use **Discover** para visualizar logs em tempo real
6. Filtre por serviço: `service:"ead-authuser-service"`

---

## 📝 Notas de Versão (Principais Mudanças)
- **Modernização**: Atualização completa de Java 11 para **Java 21**.
- **Segurança**: Migração para Spring Security 6.x e JJWT 0.12.6.
- **Jakarta EE**: Migração de `javax.*` para `jakarta.*`.
- **Configurações Locais**: Cada microsserviço possui configurações independentes com propriedades necessárias.
- **Arquitetura Independente**: Cada serviço tem sua própria infraestrutura (PostgreSQL + RabbitMQ).
- **Logs Centralizados**: Integração completa com ELK Stack via Logstash TCP Appender.
- **Logs Estruturados**: Todos os microsserviços enviam logs em formato JSON com identificação de serviço.
- **Comunicação entre Serviços**: AuthUser se comunica com Course Service via HTTP client configurado.