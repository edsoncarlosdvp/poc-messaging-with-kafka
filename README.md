# Prática de Mensageria com Kafka

Projeto demonstrativo de arquitetura de mensageria assíncrona utilizando **Apache Kafka** com **Spring Boot**.

## 🎯 Visão Geral

Implementação de um sistema de processamento de pedidos com:
- **1 Produtor**: Recebe pedidos via REST API e os publica no Kafka
- **2 Consumidores**: Processam pedidos de forma independente e assíncrona
- **Zookeeper + Kafka**: Orquestração de fila de mensagens

## 📋 Pré-requisitos

- **Docker** e **Docker Compose**
- **Java 21+**
- **Maven 3.8+** (opcional, já incluído via `mvnw`)

## 🚀 Execução Rápida

### 1. Iniciar os serviços com Docker Compose

```bash
docker-compose up --build
```

Isto irá iniciar:
- Zookeeper (porta 2181)
- Kafka (porta 9092)
- Producer (porta 8084 → 8081)
- Consumer 01 (porta 8082)
- Consumer 02 (porta 8083)

### 2. Testar a API do Produtor

**Criar um pedido:**

```bash
curl -X POST http://localhost:8084/orders \
  -H "Content-Type: application/json" \
  -d '{"id":"1","product":"Notebook","quantity":2,"price":3000.00}'
```

Os consumidores processarão a mensagem automaticamente.

### 3. Parar os serviços

```bash
docker-compose down
```

## 🏗️ Arquitetura do Projeto

```
┌───────────────────────────────────────────────────────────────────────────┐
│                         CLIENTE HTTP                                      │
│                    (curl, Postman, etc)                                   │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ POST /orders
                                    ▼
                         ┌──────────────────────┐
                         │  PRODUCER (8084)     │
                         │ ┌──────────────────┐ │
                         │ │ OrderController  │ │
                         │ └────────┬─────────┘ │
                         │          │           │
                         │ ┌────────▼─────────┐ │
                         │ │  OrderService    │ │
                         │ └────────┬─────────┘ │
                         └──────────┼───────────┘
                                    │ Publica mensagem
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ╔═══════════════════════════════════════════════════════════════════════╗ │
│  ║         APACHE KAFKA (porta 9092)                                    ║ │
│  ║    ┌──────────────────────────────────────────────────────────────┐  ║ │
│  ║    │  Topic: "orders"                                             │  ║ │
│  ║    │  (Partition 0)                                               │  ║ │
│  ║    └──────────────────────────────────────────────────────────────┘  ║ │
│  ║                                                                       ║ │
│  ║  (Zookeeper: 2181)                                                   ║ │
│  ╚═══════════════════════════════════════════════════════════════════════╝ │
│                       ▲                        ▲                           │
│                       │                        │                           │
│  (Consome mensagem)   │                        │   (Consome mensagem)      │
└───────────────────────┼────────────────────────┼──────────────────────────┘
                        │                        │
           ┌────────────▼──────────┐  ┌──────────▼──────────────┐
           │ CONSUMER 01 (8082)    │  │ CONSUMER 02 (8083)      │
           │ ┌────────────────────┐│  │ ┌────────────────────┐  │
           │ │  OrderService      ││  │ │  OrderService      │  │
           │ │ @KafkaListener     ││  │ │ @KafkaListener     │  │
           │ └────────────────────┘│  │ └────────────────────┘  │
           │                        │  │                        │
           │ (Processa pedidos)     │  │ (Processa pedidos)     │
           └────────────────────────┘  └────────────────────────┘
```

## 📁 Estrutura de Diretórios

```
pratice-messaging-kafka/
│
├── 📦 producer-kafka/
│   ├── src/main/java/com/inovationtech/example_kafka/
│   │   ├── ProducerKafkaApplication.java     (entry point)
│   │   ├── controller/
│   │   │   └── OrderController.java          (REST API)
│   │   ├── service/
│   │   │   └── OrderService.java             (envio de mensagens)
│   │   ├── config/
│   │   │   ├── KafkaProducerConfig.java      (config producer)
│   │   │   └── KafkaTopicConfig.java         (criação de tópicos)
│   │   └── record/
│   │       └── OrderRecord.java              (DTO)
│   ├── src/main/resources/
│   │   └── application.yaml
│   ├── Dockerfile
│   └── pom.xml
│
├── 📦 consumer01-kafka/
│   ├── src/main/java/com/inovationtech/consumer_kafka/
│   │   ├── ConsumerKafkaApplication.java     (entry point)+
│   │   ├── config/
│   │   │   └── KafkaConsumerConfig.java      (config consumer)
│   │   ├── service/
│   │   │   └── OrderService.java             (processamento)
│   │   └── record/
│   │       └── OrderRecord.java              (DTO)
│   ├── src/main/resources/
│   │   └── application.yaml
│   ├── Dockerfile
│   └── pom.xml
│
├── 📦 consumer02-kafka/
│   ├── (estrutura idêntica ao consumer01)
│   ├── Dockerfile
│   └── pom.xml
│
├── 🐳 docker-compose.yml                   (orquestração)
└── 📄 README.md                             (documentação)
```

##  API do Produtor

### Criar Pedido
- **Método:** `POST`
- **Endpoint:** `/orders`
- **Porta:** `8084` (mapeado para 8081)
- **Body (JSON):**

```json
{
  "id": "12345",
  "product": "Notebook Dell",
  "quantity": 1,
  "price": 3500.00
}
```

**Resposta:** `202 Accepted`

A mensagem será publicada no tópico `orders` do Kafka e processada pelos consumidores.

## 🔄 Fluxo de Processamento

```
REST API (8084)
    │
    ├──→ OrderController.createOrder()
    │
    ├──→ OrderService.sendMessageOrder()
    │
    └──→ Kafka Topic: "orders"
         │
         ├──→ Consumer 01 (8082) - Processa a mensagem
         │
         └──→ Consumer 02 (8083) - Processa a mensagem
```

## 🛠️ Desenvolvimento Local (sem Docker)

### Opção 1: Iniciar manualmente

**Terminal 1 - Zookeeper + Kafka:**
```bash
# Requer Kafka instalado localmente
$KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties
$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties
```

**Terminal 2 - Producer:**
```bash
cd producer-kafka
./mvnw spring-boot:run
```

**Terminal 3 - Consumer 01:**
```bash
cd consumer01-kafka
./mvnw spring-boot:run
```

**Terminal 4 - Consumer 02:**
```bash
cd consumer02-kafka
./mvnw spring-boot:run
```

### Opção 2: Build local

```bash
# Producer
cd producer-kafka
./mvnw clean package

# Consumers
cd consumer01-kafka
./mvnw clean package

cd consumer02-kafka
./mvnw clean package
```

## 📦 Dependências Principais

- **Spring Boot** 4.0.1
- **Spring Kafka**
- **Spring Web**
- **Java** 21

## 🐳 Imagens Docker

Cada serviço possui seu próprio `Dockerfile`:
- `producer-kafka/Dockerfile`
- `consumer01-kafka/Dockerfile`
- `consumer02-kafka/Dockerfile`

As imagens do Kafka e Zookeeper são obtidas do Confluent Docker Hub.

## 📊 Monitoramento

### Ver logs do Producer:
```bash
docker-compose logs -f producer-kafka
```

### Ver logs do Consumer 01:
```bash
docker-compose logs -f consumer01-kafka
```

### Ver logs do Consumer 02:
```bash
docker-compose logs -f consumer02-kafka
```

### Ver logs do Kafka:
```bash
docker-compose logs -f kafka
```

## 🔍 Debugging

### Conectar ao Kafka (dentro do container):
```bash
docker-compose exec kafka kafka-topics --list --bootstrap-server localhost:9092
```

### Ver mensagens do tópico:
```bash
docker-compose exec kafka kafka-console-consumer --bootstrap-server localhost:9092 --topic orders --from-beginning
```

## ✅ Exemplo de Fluxo Completo

1. **Iniciar os serviços:**
   ```bash
   docker-compose up --build
   ```

2. **Enviar um pedido:**
   ```bash
   curl -X POST http://localhost:8084/orders \
     -H "Content-Type: application/json" \
     -d '{"id":"001","product":"Mouse Wireless","quantity":5,"price":50.00}'
   ```

3. **Verificar os logs dos consumidores:**
   ```bash
   docker-compose logs consumer01-kafka consumer02-kafka
   ```

## 📝 Notas Importantes

- O tópico `orders` é criado automaticamente pelo `KafkaTopicConfig`
- Ambos os consumidores processam a **mesma mensagem** (padrão publish-subscribe)
- As mensagens são persistidas no Kafka (replication factor = 1)
- Para ambientes de produção, aumentar o `replication-factor`

## 🤝 Contribuições

Este é um projeto de aprendizado. Sinta-se livre para:
- Adicionar novos consumidores
- Implementar diferentes estratégias de particionamento
- Expandir a lógica de negócio

## 📄 Licença

Projeto educacional - Desenvolvido com base em um vídeo no YouTube para entender na prática, o conceito de menssageria utilizando o Kafka. Logo, pode ser utilizado  livremente para fins de aprendizado.

---

**Desenvolvido para praticar padrões de mensageria com Kafka e Spring Boot**
