# Arquitetura de Microsserviços para Plataforma EAD: Uma Implementação Moderna com Spring Cloud

## Resumo

Este artigo apresenta uma implementação completa de uma plataforma de Educação a Distância (EAD) baseada em arquitetura de microsserviços, utilizando tecnologias modernas do ecossistema Spring Cloud. O projeto demonstra a aplicação de padrões arquiteturais essenciais como Service Discovery, API Gateway, configuração centralizada, mensageria assíncrona e observabilidade através de logs centralizados.

## 1. Introdução

A arquitetura de microsserviços tornou-se fundamental para sistemas distribuídos modernos, oferecendo escalabilidade, manutenibilidade e resiliência. Este projeto implementa uma plataforma EAD completa, demonstrando como diferentes serviços podem colaborar de forma independente e eficiente.

## 2. Stack Tecnológica

### 2.1 Core Technologies
- **Java 21**: Versão LTS mais recente com melhorias de performance e recursos modernos
- **Spring Boot 3.5.7**: Framework principal com suporte completo ao Jakarta EE
- **Spring Cloud 2025.0.0**: Conjunto de ferramentas para sistemas distribuídos
- **Maven**: Gerenciamento de dependências e build

### 2.2 Infraestrutura de Dados
- **PostgreSQL 15**: Banco relacional isolado por microsserviço
- **RabbitMQ**: Message broker para comunicação assíncrona
- **Docker & Docker Compose**: Containerização da infraestrutura

### 2.3 Observabilidade e Monitoramento
- **ELK Stack**: Elasticsearch 7.17, Logstash, Kibana
- **Logstash Logback Encoder**: Integração nativa para logs estruturados
- **Spring Boot Actuator**: Métricas e health checks

## 3. Arquitetura do Sistema

### 3.1 Visão Geral
O sistema é composto por 6 microsserviços principais organizados em camadas:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Config Server │    │  Eureka Server  │    │   ELK Stack     │
│     (8888)      │    │     (8761)      │    │ (9200,5601,5044)│
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   API Gateway   │
                    │     (8080)      │
                    └─────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ AuthUser    │    │   Course    │    │Notification │
│ Service     │    │  Service    │    │  Service    │
│  (8087)     │    │   (8082)    │    │   (8084)    │
└─────────────┘    └─────────────┘    └─────────────┘
```

### 3.2 Microsserviços de Negócio

#### AuthUser Service (8087)
- **Responsabilidade**: Gestão de usuários, autenticação e autorização
- **Tecnologias**: Spring Security 6.x, JWT (JJWT 0.12.6)
- **Banco**: PostgreSQL (porta 5432)
- **Mensageria**: RabbitMQ (portas 5672/15672)
- **Funcionalidades**:
  - Cadastro e autenticação de usuários
  - Geração e validação de tokens JWT
  - Publicação de eventos de usuário via RabbitMQ
  - Comunicação HTTP com Course Service

#### Course Service (8082)
- **Responsabilidade**: Gestão de cursos, módulos e matrículas
- **Banco**: PostgreSQL (porta 5433)
- **Mensageria**: RabbitMQ (portas 5673/15673)
- **Funcionalidades**:
  - CRUD de cursos e módulos
  - Gestão de matrículas
  - Consumo de eventos de usuário
  - Publicação de comandos de notificação

#### Notification Service (8084)
- **Responsabilidade**: Gestão e envio de notificações
- **Banco**: PostgreSQL (porta 5434)
- **Mensageria**: RabbitMQ (portas 5674/15674)
- **Funcionalidades**:
  - Processamento de comandos de notificação
  - Histórico de notificações enviadas
  - Integração com provedores de notificação

### 3.3 Serviços de Infraestrutura

#### Config Server (8888)
- **Função**: Configuração centralizada opcional
- **Tecnologia**: Spring Cloud Config
- **Benefícios**: Configurações externalizadas e versionadas

#### Eureka Server (8761)
- **Função**: Service Discovery e Service Registry
- **Tecnologia**: Netflix Eureka
- **Benefícios**: Descoberta automática de serviços e load balancing

#### API Gateway (8080)
- **Função**: Ponto de entrada único e roteamento
- **Tecnologia**: Spring Cloud Gateway
- **Funcionalidades**:
  - Roteamento baseado em path
  - Load balancing automático
  - Integração com Eureka para descoberta de serviços

## 4. Padrões Arquiteturais Implementados

### 4.1 Database per Service
Cada microsserviço possui sua própria instância PostgreSQL, garantindo:
- **Isolamento de dados**: Falhas em um serviço não afetam outros
- **Autonomia tecnológica**: Cada serviço pode evoluir independentemente
- **Escalabilidade independente**: Recursos podem ser alocados conforme demanda

### 4.2 Event-Driven Architecture
Implementada através do RabbitMQ com padrões:

#### Publish-Subscribe (AuthUser → Course)
```java
// AuthUser publica eventos de usuário
@Value("${ead.broker.exchange.userEvent}")
String userEventExchange;

// Course consome eventos via FANOUT exchange
@RabbitListener(bindings = @QueueBinding(
    value = @Queue("${ead.broker.queue.userEventQueue.name}"),
    exchange = @Exchange(value = "${ead.broker.exchange.userEventExchange}", 
                        type = ExchangeTypes.FANOUT)
))
```

#### Command Pattern (Course → Notification)
```java
// Course envia comandos de notificação
@Value("${ead.broker.exchange.notificationCommandExchange}")
String notificationExchange;

// Notification processa comandos via TOPIC exchange
@RabbitListener(bindings = @QueueBinding(
    value = @Queue("${ead.broker.queue.notificationCommandQueue.name}"),
    exchange = @Exchange(value = "${ead.broker.exchange.notificationCommandExchange}", 
                        type = ExchangeTypes.TOPIC),
    key = "${ead.broker.key.notificationCommandKey}"
))
```

### 4.3 Circuit Breaker Pattern
Implementado no AuthUser Service para comunicação com Course Service:

```java
@CircuitBreaker(name = "circuitbreakerInstance")
public Page<CourseDto> getAllCoursesByUser(UUID userId, Pageable pageable, String token) {
    // Implementação com fallback automático
}

public Page<CourseDto> circuitbreakerfallback(UUID userId, Pageable pageable, Throwable t) {
    log.error("Circuit breaker ativado: {}", t.toString());
    return new PageImpl<>(new ArrayList<>());
}
```

## 5. Observabilidade e Monitoramento

### 5.1 ELK Stack Integration
Implementação completa de logs centralizados:

#### Logstash Configuration
```yaml
input {
  tcp {
    port => 5000
    codec => json
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "ead-logs-%{+YYYY.MM.dd}"
  }
}
```

#### Logback Configuration (por microsserviço)
```xml
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>localhost:5000</destination>
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
        <providers>
            <timestamp/>
            <logLevel/>
            <loggerName/>
            <mdc/>
            <message/>
            <pattern>
                <pattern>{"service": "ead-authuser-service"}</pattern>
            </pattern>
        </providers>
    </encoder>
</appender>
```

### 5.2 Structured Logging
Cada microsserviço envia logs estruturados em JSON com:
- **Identificação do serviço**: Campo `service` único
- **Timestamp preciso**: Para correlação temporal
- **Contexto MDC**: Para rastreamento distribuído
- **Níveis de log**: INFO, DEBUG, ERROR para filtragem

## 6. Segurança

### 6.1 JWT Authentication
Implementação moderna com JJWT 0.12.6:

```java
@Component
public class JwtProvider {
    @Value("${ead.auth.jwtSecret}")
    private String jwtSecret;
    
    @Value("${ead.auth.jwtExpirationMs}")
    private int jwtExpirationMs;
    
    public String generateJwt(Authentication authentication) {
        return Jwts.builder()
            .subject(userPrincipal.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
            .signWith(getSigningKey())
            .compact();
    }
}
```

### 6.2 Spring Security 6.x
Configuração moderna com SecurityFilterChain:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/auth/**").permitAll()
            .anyRequest().authenticated()
        )
        .addFilterBefore(authenticationJwtFilter(), UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

## 7. Containerização e Deploy

### 7.1 Docker Compose Strategy
Cada microsserviço possui sua infraestrutura isolada:

```yaml
# authuser/docker-compose.yml
services:
  postgres-authuser:
    image: postgres:15-alpine
    ports: ["5432:5432"]
    
  rabbitmq-authuser:
    image: rabbitmq:3-management-alpine
    ports: ["5672:5672", "15672:15672"]
```

### 7.2 ELK Stack Centralizado
```yaml
# docker-compose-elk.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.10
    ports: ["9200:9200"]
    
  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.10
    ports: ["5000:5000"]
    depends_on: [elasticsearch]
    
  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.10
    ports: ["5601:5601"]
    depends_on: [elasticsearch]
```

## 8. Ordem de Inicialização e Dependências

### 8.1 Sequência Crítica
```bash
# 1. Infraestrutura
docker-compose up -d  # Bancos + RabbitMQ + ELK

# 2. Serviços de Descoberta
java -jar config-server.jar     # Opcional
java -jar eureka-server.jar     # Obrigatório

# 3. Microsserviços de Negócio (paralelo)
java -jar authuser.jar
java -jar course.jar
java -jar notification.jar

# 4. Gateway (último)
java -jar api-gateway.jar
```

### 8.2 Health Checks
Monitoramento via Eureka Dashboard (http://localhost:8761):
- Status UP/DOWN de cada serviço
- Instâncias registradas
- Heartbeat e renovação de lease

## 9. Comunicação Entre Serviços

### 9.1 Síncrona (HTTP)
AuthUser → Course Service via RestTemplate:

```java
@Component
public class CourseClient {
    @Value("${ead.api.url.course}")
    String courseServiceUrl;
    
    public Page<CourseDto> getAllCoursesByUser(UUID userId, Pageable pageable, String token) {
        String url = courseServiceUrl + "/courses/users/" + userId;
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", token);
        
        return restTemplate.exchange(url, HttpMethod.GET, 
            new HttpEntity<>(headers), 
            new ParameterizedTypeReference<ResponsePageDto<CourseDto>>() {}
        ).getBody();
    }
}
```

### 9.2 Assíncrona (RabbitMQ)
Event publishing e consuming com configurações específicas:

```yaml
# Configurações RabbitMQ por serviço
ead:
  broker:
    exchange:
      userEvent: ead.userevent
      notificationCommandExchange: ead.notificationcommand
    queue:
      userEventQueue:
        name: ead.userevent.course.queue
    key:
      notificationCommandKey: ead.notificationcommand.key
```

## 10. Benefícios da Arquitetura

### 10.1 Escalabilidade
- **Horizontal**: Cada serviço pode ser replicado independentemente
- **Vertical**: Recursos alocados conforme demanda específica
- **Tecnológica**: Diferentes tecnologias por contexto de negócio

### 10.2 Resiliência
- **Isolamento de falhas**: Problemas em um serviço não propagam
- **Circuit Breaker**: Proteção contra cascata de falhas
- **Graceful degradation**: Fallbacks implementados

### 10.3 Manutenibilidade
- **Deploys independentes**: Atualizações sem impacto sistêmico
- **Equipes autônomas**: Ownership por contexto de negócio
- **Testabilidade**: Testes isolados por serviço

### 10.4 Observabilidade
- **Logs centralizados**: Visão unificada via Kibana
- **Rastreamento distribuído**: Correlação entre serviços
- **Métricas específicas**: Health checks e performance

## 11. Considerações de Produção

### 11.1 Configurações Externalizadas
```yaml
# Uso de variáveis de ambiente
ead:
  auth:
    jwtSecret: ${EAD_AUTH_JWTSECRET:default-secret}
    jwtExpirationMs: ${EAD_AUTH_JWTEXPIRATIONMS:86400000}
```

### 11.2 Profiles Spring
Configurações específicas por ambiente (dev, staging, prod).

### 11.3 Monitoramento Avançado
- Métricas de negócio via Micrometer
- Alertas baseados em logs estruturados
- Dashboards Kibana para análise operacional

## 12. Conclusão

Esta implementação demonstra uma arquitetura de microsserviços moderna e completa, aplicando padrões consolidados da indústria. O projeto oferece:

- **Base sólida** para sistemas distribuídos educacionais
- **Padrões reutilizáveis** para outros domínios
- **Observabilidade completa** para operação em produção
- **Escalabilidade horizontal** para crescimento orgânico

A combinação de Spring Cloud, containerização Docker e ELK Stack fornece uma plataforma robusta, observável e escalável para aplicações empresariais modernas.

## Referências

- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/)
- [ELK Stack Documentation](https://www.elastic.co/guide/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)