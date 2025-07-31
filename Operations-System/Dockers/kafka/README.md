# Kafka Cluster with Podman Compose

A production-ready Apache Kafka cluster setup using Podman Compose with KRaft mode, Schema Registry, and comprehensive monitoring.

## 🏗️ Architecture

- **3-Node Kafka Cluster** with KRaft mode (no Zookeeper required)
- **Schema Registry** for data governance
- **Prometheus + Grafana** for monitoring and visualization
- **SASL/PLAIN Authentication** for basic security
- **External Access** support for client connections

## 📋 Prerequisites

- Podman and Podman Compose installed
- At least 8GB RAM available
- Ports 3000, 8081, 9090, 9092-9094 available

## 🚀 Quick Start

### 1. Start the Cluster
```bash
cd solution
./scripts/start.sh
```

### 2. Create Topics
```bash
./scripts/create-topics.sh
```

### 3. Access Services
- **Kafka Brokers**: `localhost:9092`, `localhost:9093`, `localhost:9094`
- **Schema Registry**: http://localhost:8081
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 (admin/grafana-secret)

### 4. Stop the Cluster
```bash
./scripts/stop.sh
```

## 📊 Services Overview

| Service | Port | Description |
|---------|------|-------------|
| Kafka Broker 1 | 9092 | Primary Kafka broker |
| Kafka Broker 2 | 9093 | Secondary Kafka broker |
| Kafka Broker 3 | 9094 | Tertiary Kafka broker |
| Schema Registry | 8081 | Confluent Schema Registry |
| Prometheus | 9090 | Metrics collection |
| Grafana | 3000 | Monitoring dashboards |

## 🔐 Authentication

The cluster uses SASL/PLAIN authentication:
- **Admin User**: `admin` / `admin-secret`
- **Client User**: `client` / `client-secret`

### Client Configuration Example
```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="client" password="client-secret";
```

## 📝 Pre-configured Topics

The following topics are created by default:
- `account` - Business account events (6 partitions, RF=3)
- `fund-transfer` - Financial transactions (6 partitions, RF=3)
- `notification` - User notifications (6 partitions, RF=3)
- `notification-dlq` - Dead letter queue (6 partitions, RF=3)

## 🛠️ Management Scripts

### Start Cluster
```bash
./scripts/start.sh
```
Starts all services and performs health checks.

### Stop Cluster
```bash
./scripts/stop.sh
```
Gracefully stops all services while preserving data.

### Create Topics
```bash
./scripts/create-topics.sh
```
Creates the required business topics with optimal configurations.

### Backup Configuration
```bash
./scripts/backup.sh
```
Creates a backup of cluster configuration and metadata.

## 🔧 Configuration

### Environment Variables (.env)
- `KAFKA_CLUSTER_ID`: Unique cluster identifier
- `KAFKA_ADMIN_USER/PASSWORD`: Admin credentials
- `GRAFANA_ADMIN_USER/PASSWORD`: Grafana credentials

### Resource Limits
- **Kafka Brokers**: 2GB RAM, 1 CPU each
- **Schema Registry**: 1GB RAM, 0.5 CPU
- **Prometheus**: 1GB RAM, 0.5 CPU
- **Grafana**: 512MB RAM, 0.5 CPU

## 📈 Monitoring

### Prometheus Metrics
- Kafka broker metrics via JMX
- Schema Registry metrics
- System metrics

### Grafana Dashboards
- Kafka cluster overview
- Broker performance metrics
- Topic and partition statistics

Access Grafana at http://localhost:3000 with credentials `admin/grafana-secret`.

## 🧪 Testing the Cluster

### Produce Messages
```bash
podman exec -it kafka-1 kafka-console-producer.sh \
  --bootstrap-server localhost:19092 \
  --topic account
```

### Consume Messages
```bash
podman exec -it kafka-1 kafka-console-consumer.sh \
  --bootstrap-server localhost:19092 \
  --topic account \
  --from-beginning
```

### List Topics
```bash
podman exec kafka-1 kafka-topics.sh \
  --bootstrap-server localhost:19092 \
  --list
```

## 🔍 Troubleshooting

### Check Service Status
```bash
podman-compose ps
```

### View Logs
```bash
podman-compose logs kafka-1
podman-compose logs schema-registry
```

### Health Checks
```bash
# Kafka broker health
podman exec kafka-1 kafka-broker-api-versions.sh --bootstrap-server localhost:19092

# Schema Registry health
curl http://localhost:8081/subjects

# Prometheus health
curl http://localhost:9090/-/healthy
```

### Common Issues

1. **Port Conflicts**: Ensure ports 3000, 8081, 9090, 9092-9094 are available
2. **Memory Issues**: Ensure at least 8GB RAM is available
3. **Authentication Errors**: Verify SASL credentials in client configuration
4. **Network Issues**: Check if containers can communicate via the kafka-network

## 📁 Directory Structure

```
solution/
├── podman-compose.yml      # Main compose file
├── .env                    # Environment variables
├── README.md              # This file
├── config/                # Configuration files
│   ├── kafka/            # Kafka configurations
│   ├── prometheus/       # Prometheus config
│   └── grafana/          # Grafana configs
├── scripts/              # Management scripts
│   ├── start.sh         # Start cluster
│   ├── stop.sh          # Stop cluster
│   ├── create-topics.sh # Create topics
│   └── backup.sh        # Backup script
└── monitoring/           # Monitoring configs
    └── grafana-dashboard.json
```

## 🔒 Security Considerations

This setup includes basic security measures suitable for development and testing:
- SASL/PLAIN authentication
- Internal network isolation
- Resource limits

For production environments, consider:
- SSL/TLS encryption
- More robust authentication (SASL/SCRAM, OAuth)
- Network policies and firewalls
- Regular security updates
- Audit logging

## 🚀 Production Deployment

For production deployment, consider:
- Using persistent storage with proper backup strategies
- Implementing SSL/TLS encryption
- Setting up proper monitoring and alerting
- Configuring log aggregation
- Implementing disaster recovery procedures
- Using secrets management for credentials

## 📚 Additional Resources

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Kafka KRaft Mode](https://kafka.apache.org/documentation/#kraft)
- [Prometheus Monitoring](https://prometheus.io/docs/)
- [Grafana Dashboards](https://grafana.com/docs/)

## 🤝 Support

For issues and questions:
1. Check the troubleshooting section
2. Review service logs
3. Verify configuration files
4. Ensure all prerequisites are met
