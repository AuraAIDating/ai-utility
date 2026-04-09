---
name: kafka-patterns
description: Apache Kafka producer and consumer patterns for Spring Boot services. Covers KafkaTopics constants, idempotent consumers, DefaultErrorHandler with DLT, producer with completion callback, @EmbeddedKafka unit tests, and Awaitility assertions.
origin: polaris-hha
---

# Kafka Patterns Skill

## HHA Topics Reference

```java
public final class KafkaTopics {
    public static final String HHA_ORDER_RECEIVED       = "hha-order-received";
    public static final String DOCUMENT_SCAN_RESULTS    = "document-scan-results";
    public static final String HHA_ORDER_STATE_CHANGES  = "hha-order-state-changes";
    public static final String HHA_ORDER_RECEIVED_DLT   = "hha-order-received.DLT";
    public static final String DOCUMENT_SCAN_RESULTS_DLT = "document-scan-results.DLT";
    private KafkaTopics() {}
}
```

---

## 1. Topic Configuration

```java
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic hhaOrderReceivedTopic() {
        return TopicBuilder.name(KafkaTopics.HHA_ORDER_RECEIVED)
            .partitions(6).replicas(1).build();
    }

    @Bean
    public NewTopic documentScanResultsTopic() {
        return TopicBuilder.name(KafkaTopics.DOCUMENT_SCAN_RESULTS)
            .partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic hhaOrderStateChangesTopic() {
        return TopicBuilder.name(KafkaTopics.HHA_ORDER_STATE_CHANGES)
            .partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic hhaOrderReceivedDlt() {
        return TopicBuilder.name(KafkaTopics.HHA_ORDER_RECEIVED_DLT)
            .partitions(1).replicas(1).build();
    }
}
```

---

## 2. Producer Pattern — With Completion Callback

```java
@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, HhaOrderReceivedEvent> kafkaTemplate;

    public void publishOrderReceived(HhaOrderReceivedEvent event) {
        kafkaTemplate.send(
                KafkaTopics.HHA_ORDER_RECEIVED,
                event.orderId().toString(),   // key = orderId for partition locality
                event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("order_received_publish_failed orderId={} error={}",
                        event.orderId(), ex.getMessage());
                } else {
                    log.info("order_received_published orderId={} partition={} offset={}",
                        event.orderId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

---

## 3. Idempotent Consumer Pattern

```java
@Component
public class HhaOrderReceivedConsumer {

    private final HhaOrderPersistenceService persistenceService;
    private final TemporalWorkflowStarter workflowStarter;

    @KafkaListener(
        topics = KafkaTopics.HHA_ORDER_RECEIVED,
        groupId = "polaris-order-consumer-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void onOrderReceived(
            @Payload HhaOrderReceivedEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("order_received orderId={} partition={} offset={}",
            event.orderId(), partition, offset);

        try {
            // IDEMPOTENCY: check if already processed
            if (persistenceService.existsByOrderId(event.orderId())) {
                log.info("order_already_exists orderId={} skipping", event.orderId());
                ack.acknowledge();
                return;
            }

            persistenceService.persist(event);
            workflowStarter.startWorkflow(event.orderId());
            ack.acknowledge();

        } catch (DuplicateKeyException e) {
            // Race condition — idempotent, just ack
            log.info("order_duplicate_key orderId={}", event.orderId());
            ack.acknowledge();
        } catch (Exception e) {
            log.error("order_processing_failed orderId={}", event.orderId(), e);
            // Do NOT ack — error handler will retry then DLT
            throw e;
        }
    }
}
```

---

## 4. Error Handler with Dead Letter Topic

```java
@Configuration
public class KafkaErrorHandlerConfig {

    @Bean
    public DefaultErrorHandler kafkaErrorHandler(
            KafkaTemplate<String, Object> kafkaTemplate) {

        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
        );

        ExponentialBackOff backOff = new ExponentialBackOff(1000L, 2.0);
        backOff.setMaxElapsedTime(30_000L); // 30 seconds max

        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

        // Don't retry these — publish straight to DLT
        handler.addNotRetryableExceptions(
            InvalidOrderException.class,
            JsonMappingException.class,
            SerializationException.class
        );

        return handler;
    }
}
```

---

## 5. Consumer Group IDs (Convention)

| Service                | Topic                   | Group ID                     |
|------------------------|-------------------------|------------------------------|
| polaris-order-consumer | hha-order-received      | polaris-order-consumer-group |
| polaris-order-consumer | document-scan-results   | polaris-order-consumer-group |
| polaris-order-agent    | hha-order-state-changes | polaris-order-agent-group    |

---

## 6. application.yaml

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        spring.json.add.type.headers: false
        linger.ms: 10
        delivery.timeout.ms: 120000
    consumer:
      group-id: ${spring.application.name}-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "com.waveum.polaris.*"
```

---

## 7. Kafka Event Schema Rules

ALL Kafka messages MUST include:

```java
// Every event record must have these three fields:
UUID eventId           // unique per event instance
Instant eventTimestamp // when the event was created
UUID orderId           // the order this event relates to
```

Example:

```java
public record HhaOrderReceivedEvent(
    UUID eventId,           // REQUIRED
    Instant eventTimestamp, // REQUIRED
    UUID orderId,           // REQUIRED — also used as message key
    String orderOrigin,
    String physicianNpi,
    String healthcareProviderNpi,
    int productCount,
    String status
) {}
```

---

## 8. @EmbeddedKafka Unit Test

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {KafkaTopics.HHA_ORDER_RECEIVED, KafkaTopics.HHA_ORDER_RECEIVED_DLT}
)
class HhaOrderReceivedConsumerTest {

    @Autowired
    private KafkaTemplate<String, HhaOrderReceivedEvent> kafkaTemplate;

    @MockitoBean
    private HhaOrderPersistenceService persistenceService;

    @MockitoBean
    private TemporalWorkflowStarter workflowStarter;

    @Test
    void onOrderReceived_newOrder_persistsAndStartsWorkflow() throws Exception {
        HhaOrderReceivedEvent event = buildTestEvent();

        kafkaTemplate.send(KafkaTopics.HHA_ORDER_RECEIVED, event.orderId().toString(), event);

        await().atMost(5, SECONDS).untilAsserted(() -> {
            verify(persistenceService).persist(event);
            verify(workflowStarter).startWorkflow(event.orderId());
        });
    }

    @Test
    void onOrderReceived_duplicate_skipsProcessing() throws Exception {
        when(persistenceService.existsByOrderId(any())).thenReturn(true);

        kafkaTemplate.send(KafkaTopics.HHA_ORDER_RECEIVED,
            UUID.randomUUID().toString(), buildTestEvent());

        await().atMost(5, SECONDS).untilAsserted(() -> {
            verify(persistenceService, never()).persist(any());
            verify(workflowStarter, never()).startWorkflow(any());
        });
    }
}
```
