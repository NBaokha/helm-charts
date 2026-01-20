# RabbitMQ Developer Guide

Guide til udviklere der skal integrere med RabbitMQ i vores Kubernetes miljø.

**VIGTIGT:** Alle kodeeksempler i denne guide er AI-genererede og udelukkende til illustration. De er ikke testet i produktion og skal ikke anvendes ukritisk. Udviklere skal tilpasse koden til deres specifikke use cases, følge projektets kodestandarder, og teste grundigt før deployment.

## Quick Start

### Connection Information

**Intern Kubernetes URL:**
```
Host: rabbitmq.rabbitmq.svc.cluster.local
Port: 5672 (AMQP)
```

**Credentials:**
- **VIGTIGT:** Brug IKKE admin credentials i applikationer
- Opret en dedikeret bruger per service/application i RabbitMQ Management UI
- Username og password gemmes i Infisical under jeres service's projekt

### Opret Application User

**I RabbitMQ Management UI:**

1. Gå til **Admin** → **Users** → **Add a user**

2. **User settings:**
   ```
   Username: my-service-user
   Password: [generer stærk password]
   Tags: (lad være tom - ingen tags nødvendigt)
   ```

3. **Permissions** (per Virtual Host):
   - Klik på brugeren → **Set permissions**
   - Virtual Host: `/` (default)
   - Configure regexp: `.*` (tillad queue/exchange oprettelse)
   - Write regexp: `.*` (tillad publishing)
   - Read regexp: `.*` (tillad consuming)

**Minimale rettigheder (mere restriktivt):**

Hvis servicen kun skal bruge specifikke queues/exchanges:

```
Configure regexp: ^$           (ingen oprettelse)
Write regexp: ^my-service\..*$ (kun publish til my-service.* exchanges)
Read regexp: ^my-service\..*$  (kun consume fra my-service.* queues)
```

**Tags forklaring:**
- **Ingen tags** = Standard application user (anbefalet)
- **monitoring** = Kan se alle connections/channels
- **management** = Kan bruge management UI
- **administrator** = Fuld admin adgang (brug KUN til admin)

**Best practice:**
- En bruger per service/application
- Minimal permissions (kun hvad servicen behøver)
- Gem credentials i Infisical under service's eget projekt
- ALDRIG hardcode credentials i kode

**Credentials:**
- Hent fra Infisical for din specifikke service
- Admin credentials er kun til platform team og infrastructure management

**Management UI:**
- Stage: https://rabbitmq.stage.saoad.dk
- Production: https://rabbitmq.prod.saoad.dk

## Spring Boot Integration

### 1. Dependencies

Tilføj til `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

Eller `build.gradle`:

```gradle
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

### 2. Application Configuration

**application.yaml:**

```yaml
spring:
  rabbitmq:
    host: rabbitmq.rabbitmq.svc.cluster.local
    port: 5672
    username: ${RABBITMQ_USERNAME}
    password: ${RABBITMQ_PASSWORD}
    
    # Connection settings
    connection-timeout: 10000
    requested-heartbeat: 60
    
    # Listener settings
    listener:
      simple:
        acknowledge-mode: auto
        prefetch: 10
        retry:
          enabled: true
          initial-interval: 3000
          max-attempts: 3
          multiplier: 2
```

### 3. Kubernetes Deployment

**Environment variabler fra Infisical:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    spec:
      containers:
        - name: my-service
          image: my-service:latest
          env:
            - name: RABBITMQ_USERNAME
              valueFrom:
                secretKeyRef:
                  name: rabbitmq-admin-credentials
                  key: RABBITMQ_ADMIN_USERNAME
            - name: RABBITMQ_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: rabbitmq-admin-credentials
                  key: RABBITMQ_ADMIN_PASSWORD
```

**Eller brug envFrom med hele secret:**

```yaml
envFrom:
  - secretRef:
      name: rabbitmq-admin-credentials
```

Så kan du bruge `${RABBITMQ_ADMIN_USERNAME}` og `${RABBITMQ_ADMIN_PASSWORD}` direkte.

## Code Examples

### Producer (Send Messages)

```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderProducer {
    
    private final RabbitTemplate rabbitTemplate;
    
    public OrderProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }
    
    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend(
            "orders.exchange",    // Exchange
            "orders.created",     // Routing key
            order                 // Message
        );
    }
}
```

### Consumer (Receive Messages)

```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Service;

@Service
public class OrderConsumer {
    
    @RabbitListener(queues = "orders.queue")
    public void handleOrder(Order order) {
        System.out.println("Received order: " + order.getId());
        // Process order
    }
}
```

### Queue and Exchange Configuration

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {
    
    public static final String EXCHANGE = "orders.exchange";
    public static final String QUEUE = "orders.queue";
    public static final String ROUTING_KEY = "orders.created";
    
    @Bean
    public Exchange ordersExchange() {
        return ExchangeBuilder
            .topicExchange(EXCHANGE)
            .durable(true)
            .build();
    }
    
    @Bean
    public Queue ordersQueue() {
        return QueueBuilder
            .durable(QUEUE)
            .build();
    }
    
    @Bean
    public Binding ordersBinding() {
        return BindingBuilder
            .bind(ordersQueue())
            .to(ordersExchange())
            .with(ROUTING_KEY)
            .noargs();
    }
}
```

## Best Practices

### 1. Connection Management

Spring Boot håndterer automatisk connection pooling. Standard settings:
- Connection cache size: 1
- Channel cache size: 25

For high-throughput services, overvej at øge channel cache:

```yaml
spring:
  rabbitmq:
    cache:
      channel:
        size: 50
        checkout-timeout: 5000
```

### 2. Error Handling

Implementer retry logic og dead letter queues:

```java
@Bean
public Queue ordersQueue() {
    return QueueBuilder
        .durable("orders.queue")
        .withArgument("x-dead-letter-exchange", "orders.dlx")
        .withArgument("x-dead-letter-routing-key", "orders.dead")
        .withArgument("x-message-ttl", 3600000) // 1 hour
        .build();
}

@Bean
public Queue deadLetterQueue() {
    return QueueBuilder
        .durable("orders.dead.queue")
        .build();
}
```

### 3. Message Serialization

Spring bruger default Jackson for JSON serialization. For custom serialization:

```java
@Bean
public Jackson2JsonMessageConverter messageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

### 4. Health Checks

Spring Boot Actuator inkluderer automatisk RabbitMQ health check:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  health:
    rabbit:
      enabled: true
```

### 5. Monitoring

Brug Spring Boot metrics til at overvåge RabbitMQ:

```yaml
management:
  metrics:
    enable:
      rabbitmq: true
```

Prometheus metrics er tilgængelige på: `http://rabbitmq.rabbitmq.svc.cluster.local:15692/metrics`

## Testing Locally

### Port Forward til RabbitMQ

```powershell
# AMQP
kubectl port-forward -n rabbitmq rabbitmq-server-0 5672:5672

# Management UI
kubectl port-forward -n rabbitmq rabbitmq-server-0 15672:15672
```

Opdater din `application-local.yaml`:

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: [få fra kubectl get secret]
```

### Få credentials

```powershell
# VIGTIGT: Disse credentials er ADMIN credentials
# Brug dem KUN til at oprette application-specifikke brugere
# Brug ALDRIG admin credentials i applikationer

# Username
kubectl get secret rabbitmq-admin-credentials -n rabbitmq -o jsonpath='{.data.RABBITMQ_ADMIN_USERNAME}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Password
kubectl get secret rabbitmq-admin-credentials -n rabbitmq -o jsonpath='{.data.RABBITMQ_ADMIN_PASSWORD}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

## Troubleshooting

### Connection Issues

```java
// Enable debug logging
logging.level.org.springframework.amqp: DEBUG
```

Tjek health endpoint:
```bash
curl http://localhost:8080/actuator/health/rabbit
```

### Message Not Received

1. Verificer queue binding i Management UI
2. Tjek routing key matcher pattern
3. Verificer consumer er startet (logs)
4. Tjek dead letter queue for fejlede beskeder

### Performance Issues

1. Øg prefetch count for consumers:
   ```yaml
   spring.rabbitmq.listener.simple.prefetch: 50
   ```

2. Brug flere concurrent consumers:
   ```yaml
   spring.rabbitmq.listener.simple.concurrency: 5
   spring.rabbitmq.listener.simple.max-concurrency: 10
   ```

3. Overvej batch processing for bulk messages

## Common Patterns

### Request-Reply Pattern

```java
// Request
Order response = rabbitTemplate.convertSendAndReceive(
    "orders.exchange",
    "orders.process",
    order,
    Order.class
);

// Reply
@RabbitListener(queues = "orders.process.queue")
public Order processOrder(Order order) {
    // Process and return result
    return processedOrder;
}
```

### Publish-Subscribe Pattern

```java
// Publisher
rabbitTemplate.convertAndSend("notifications.fanout", "", notification);

// Subscribers (multiple services)
@RabbitListener(queues = "email.notifications.queue")
public void sendEmail(Notification notification) { }

@RabbitListener(queues = "sms.notifications.queue")
public void sendSMS(Notification notification) { }
```

### Work Queue Pattern

```java
// Multiple workers consuming from same queue
@RabbitListener(queues = "tasks.queue", concurrency = "3")
public void processTask(Task task) {
    // Heavy processing
}
```

## Environments

### Stage
- URL: `rabbitmq.rabbitmq.svc.cluster.local:5672`
- Management: https://rabbitmq.stage.saoad.dk
- Resources: 1 replica, 512Mi-1Gi RAM

### Production
- URL: `rabbitmq.rabbitmq.svc.cluster.local:5672`
- Management: https://rabbitmq.prod.saoad.dk
- Resources: 3 replicas (HA), 2-4Gi RAM per node


