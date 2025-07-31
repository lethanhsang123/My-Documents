# Kafka Cluster Troubleshooting Guide

## 🔍 Common Issues and Solutions

### 1. Services Won't Start

#### Symptoms
- Services fail to start or immediately exit
- Error messages in logs about port conflicts

#### Solutions
```bash
# Check port availability
netstat -tulpn | grep -E ':(3000|8081|9090|909[2-4])'

# Check if services are already running
podman ps -a

# Clean up existing containers
podman-compose down
podman container prune -f
```

### 2. Kafka Brokers Can't Form Cluster

#### Symptoms
- Brokers start but can't communicate
- Controller election failures
- Metadata synchronization issues

#### Solutions
```bash
# Check broker logs
podman-compose logs kafka-1
podman-compose logs kafka-2
podman-compose logs kafka-3

# Verify cluster ID consistency
grep KAFKA_CLUSTER_ID .env

# Check network connectivity
podman exec kafka-1 ping kafka-2
podman exec kafka-1 ping kafka-3
```

### 3. Authentication Issues

#### Symptoms
- Client connections rejected
- SASL authentication failures

#### Solutions
```bash
# Verify SASL configuration
podman exec kafka-1 cat /etc/kafka/config/server.properties | grep -i sasl

# Test with correct credentials
podman exec kafka-1 kafka-console-producer.sh \
  --bootstrap-server localhost:19092 \
  --topic test \
  --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";')
```

### 4. Schema Registry Connection Issues

#### Symptoms
- Schema Registry can't connect to Kafka
- 500 errors from Schema Registry API

#### Solutions
```bash
# Check Schema Registry logs
podman-compose logs schema-registry

# Verify Kafka connectivity from Schema Registry
podman exec schema-registry curl -s kafka-1:19092

# Test Schema Registry API
curl -s http://localhost:8081/subjects
```

### 5. Monitoring Stack Issues

#### Symptoms
- Prometheus can't scrape metrics
- Grafana dashboards show no data

#### Solutions
```bash
# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets

# Verify JMX metrics from Kafka
curl -s http://localhost:8080/metrics

# Check Grafana data sources
curl -s http://admin:grafana-secret@localhost:3000/api/datasources
```

### 6. Topic Creation Failures

#### Symptoms
- Topics can't be created
- Replication factor errors

#### Solutions
```bash
# Check cluster health
podman exec kafka-1 kafka-topics.sh --bootstrap-server localhost:19092 --list

# Verify broker count
podman exec kafka-1 kafka-broker-api-versions.sh --bootstrap-server localhost:19092

# Create topic with lower replication factor if needed
podman exec kafka-1 kafka-topics.sh \
  --bootstrap-server localhost:19092 \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 1
```

## 🛠️ Diagnostic Commands

### Check Service Health
```bash
# Overall service status
podman-compose ps

# Individual service health
podman exec kafka-1 kafka-broker-api-versions.sh --bootstrap-server localhost:19092
curl -f http://localhost:8081/subjects
curl -f http://localhost:9090/-/healthy
curl -f http://localhost:3000/api/health
```

### View Detailed Logs
```bash
# All services
podman-compose logs

# Specific service with timestamps
podman-compose logs -t kafka-1

# Follow logs in real-time
podman-compose logs -f schema-registry

# Last 100 lines
podman-compose logs --tail=100 prometheus
```

### Network Diagnostics
```bash
# Check network connectivity
podman network ls
podman network inspect solution_kafka-network

# Test inter-service communication
podman exec kafka-1 ping kafka-2
podman exec kafka-1 telnet kafka-2 19092
```

### Resource Usage
```bash
# Check container resource usage
podman stats

# Check system resources
free -h
df -h
```

## 🔧 Configuration Validation

### Verify Environment Variables
```bash
# Check .env file
cat .env

# Verify variables are loaded
podman exec kafka-1 env | grep KAFKA
```

### Validate Compose File
```bash
# Check compose file syntax
podman-compose config

# Validate service definitions
podman-compose config --services
```

### Check File Permissions
```bash
# Verify script permissions
ls -la scripts/

# Fix permissions if needed
chmod +x scripts/*.sh
```

## 🚨 Emergency Procedures

### Complete Cluster Reset
```bash
# Stop all services
./scripts/stop.sh

# Remove all containers and volumes
podman-compose down -v
podman system prune -a -f

# Restart from scratch
./scripts/start.sh
```

### Data Recovery
```bash
# Check volume status
podman volume ls

# Backup volumes before reset
podman run --rm -v kafka-1-data:/data -v $(pwd):/backup alpine tar czf /backup/kafka-1-backup.tar.gz /data

# Restore from backup
podman run --rm -v kafka-1-data:/data -v $(pwd):/backup alpine tar xzf /backup/kafka-1-backup.tar.gz -C /
```

### Performance Issues
```bash
# Check JVM heap usage
podman exec kafka-1 jstat -gc 1

# Monitor disk I/O
podman exec kafka-1 iostat -x 1

# Check network throughput
podman exec kafka-1 iftop
```

## 📊 Monitoring and Alerting

### Key Metrics to Monitor
- Broker availability
- Under-replicated partitions
- Consumer lag
- Disk usage
- Memory usage
- Network throughput

### Log Analysis
```bash
# Search for errors
podman-compose logs | grep -i error

# Monitor specific patterns
podman-compose logs -f | grep -E "(WARN|ERROR|FATAL)"

# Analyze performance issues
podman-compose logs kafka-1 | grep -i "slow"
```

## 🔄 Maintenance Procedures

### Regular Health Checks
```bash
# Create a health check script
cat > health-check.sh << 'EOF'
#!/bin/bash
echo "Checking Kafka cluster health..."
podman exec kafka-1 kafka-broker-api-versions.sh --bootstrap-server localhost:19092 > /dev/null && echo "✅ Kafka OK" || echo "❌ Kafka FAIL"
curl -sf http://localhost:8081/subjects > /dev/null && echo "✅ Schema Registry OK" || echo "❌ Schema Registry FAIL"
curl -sf http://localhost:9090/-/healthy > /dev/null && echo "✅ Prometheus OK" || echo "❌ Prometheus FAIL"
curl -sf http://localhost:3000/api/health > /dev/null && echo "✅ Grafana OK" || echo "❌ Grafana FAIL"
EOF
chmod +x health-check.sh
```

### Log Rotation
```bash
# Check log sizes
podman exec kafka-1 du -sh /var/log/kafka/

# Rotate logs manually if needed
podman exec kafka-1 find /var/log/kafka/ -name "*.log" -mtime +7 -delete
```

## 📞 Getting Help

1. **Check this troubleshooting guide first**
2. **Review service logs for specific error messages**
3. **Verify your configuration matches the examples**
4. **Test with minimal configurations to isolate issues**
5. **Check Apache Kafka and Confluent documentation**

## 🔗 Useful Resources

- [Kafka Troubleshooting Guide](https://kafka.apache.org/documentation/#troubleshooting)
- [Confluent Platform Troubleshooting](https://docs.confluent.io/platform/current/troubleshooting/index.html)
- [Podman Compose Documentation](https://github.com/containers/podman-compose)
- [Prometheus Troubleshooting](https://prometheus.io/docs/prometheus/latest/troubleshooting/)
- [Grafana Troubleshooting](https://grafana.com/docs/grafana/latest/troubleshooting/)
