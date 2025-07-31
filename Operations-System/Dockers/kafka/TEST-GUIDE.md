# Kafka Cluster Testing Guide

## 🧪 Comprehensive Testing Procedures

This guide provides step-by-step testing procedures to validate your Kafka cluster deployment.

## 🚀 Pre-Test Setup

### 1. Start the Cluster
```bash
cd solution
./scripts/start.sh
```

### 2. Wait for Services to be Ready
```bash
# Wait approximately 2-3 minutes for all services to fully initialize
sleep 180
```

## ✅ Basic Functionality Tests

### Test 1: Cluster Health Check
```bash
echo "=== Test 1: Cluster Health Check ==="

# Check all brokers are responsive
for i in {1..3}; do
    echo "Testing Kafka Broker $i..."
    podman exec kafka-$i kafka-broker-api-versions.sh --bootstrap-server localhost:19092
    if [ $? -eq 0 ]; then
        echo "✅ Broker $i is healthy"
    else
        echo "❌ Broker $i is not responding"
    fi
done
```

### Test 2: Schema Registry Connectivity
```bash
echo "=== Test 2: Schema Registry Test ==="

# Test Schema Registry API
curl -s http://localhost:8081/subjects
if [ $? -eq 0 ]; then
    echo "✅ Schema Registry is accessible"
else
    echo "❌ Schema Registry is not responding"
fi

# Test Schema Registry health
curl -s http://localhost:8081/subjects | jq . > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ Schema Registry returns valid JSON"
else
    echo "⚠️  Schema Registry response may not be valid JSON"
fi
```

### Test 3: Monitoring Stack
```bash
echo "=== Test 3: Monitoring Stack Test ==="

# Test Prometheus
curl -s http://localhost:9090/-/healthy
if [ $? -eq 0 ]; then
    echo "✅ Prometheus is healthy"
else
    echo "❌ Prometheus is not responding"
fi

# Test Grafana
curl -s http://localhost:3000/api/health
if [ $? -eq 0 ]; then
    echo "✅ Grafana is accessible"
else
    echo "❌ Grafana is not responding"
fi
```

## 📝 Topic Management Tests

### Test 4: Create Test Topics
```bash
echo "=== Test 4: Topic Creation Test ==="

# Create the required topics
./scripts/create-topics.sh

# Verify topics were created
EXPECTED_TOPICS=("account" "fund-transfer" "notification" "notification-dlq")
ACTUAL_TOPICS=$(podman exec kafka-1 kafka-topics.sh --bootstrap-server localhost:19092 --list)

for topic in "${EXPECTED_TOPICS[@]}"; do
    if echo "$ACTUAL_TOPICS" | grep -q "^$topic$"; then
        echo "✅ Topic '$topic' exists"
    else
        echo "❌ Topic '$topic' is missing"
    fi
done
```

### Test 5: Topic Configuration Validation
```bash
echo "=== Test 5: Topic Configuration Test ==="

# Check topic configurations
for topic in account fund-transfer notification notification-dlq; do
    echo "Checking topic: $topic"
    CONFIG=$(podman exec kafka-1 kafka-topics.sh --bootstrap-server localhost:19092 --describe --topic $topic)
    
    # Check partition count
    PARTITIONS=$(echo "$CONFIG" | grep "PartitionCount" | awk '{print $2}')
    if [ "$PARTITIONS" = "6" ]; then
        echo "✅ Topic '$topic' has correct partition count (6)"
    else
        echo "❌ Topic '$topic' has incorrect partition count ($PARTITIONS)"
    fi
    
    # Check replication factor
    REPLICATION=$(echo "$CONFIG" | grep "ReplicationFactor" | awk '{print $2}')
    if [ "$REPLICATION" = "3" ]; then
        echo "✅ Topic '$topic' has correct replication factor (3)"
    else
        echo "❌ Topic '$topic' has incorrect replication factor ($REPLICATION)"
    fi
done
```

## 🔐 Authentication Tests

### Test 6: SASL Authentication
```bash
echo "=== Test 6: Authentication Test ==="

# Test with correct credentials
echo "Testing with admin credentials..."
podman exec kafka-1 kafka-console-producer.sh \
    --bootstrap-server localhost:19092 \
    --topic account \
    --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";') \
    --timeout-ms 5000 < /dev/null

if [ $? -eq 0 ]; then
    echo "✅ Admin authentication successful"
else
    echo "❌ Admin authentication failed"
fi

# Test with client credentials
echo "Testing with client credentials..."
podman exec kafka-1 kafka-console-producer.sh \
    --bootstrap-server localhost:19092 \
    --topic account \
    --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="client" password="client-secret";') \
    --timeout-ms 5000 < /dev/null

if [ $? -eq 0 ]; then
    echo "✅ Client authentication successful"
else
    echo "❌ Client authentication failed"
fi
```

## 📨 Message Production and Consumption Tests

### Test 7: Message Flow Test
```bash
echo "=== Test 7: Message Production and Consumption Test ==="

# Create test messages
TEST_MESSAGES=("Test message 1" "Test message 2" "Test message 3")

# Produce messages
echo "Producing test messages..."
for msg in "${TEST_MESSAGES[@]}"; do
    echo "$msg" | podman exec -i kafka-1 kafka-console-producer.sh \
        --bootstrap-server localhost:19092 \
        --topic account \
        --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";')
done

if [ $? -eq 0 ]; then
    echo "✅ Messages produced successfully"
else
    echo "❌ Message production failed"
fi

# Consume messages
echo "Consuming test messages..."
CONSUMED_MESSAGES=$(timeout 10s podman exec kafka-1 kafka-console-consumer.sh \
    --bootstrap-server localhost:19092 \
    --topic account \
    --from-beginning \
    --max-messages 3 \
    --consumer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";') 2>/dev/null)

MESSAGE_COUNT=$(echo "$CONSUMED_MESSAGES" | wc -l)
if [ "$MESSAGE_COUNT" -ge 3 ]; then
    echo "✅ Messages consumed successfully ($MESSAGE_COUNT messages)"
else
    echo "❌ Message consumption failed (only $MESSAGE_COUNT messages)"
fi
```

### Test 8: Multi-Topic Message Test
```bash
echo "=== Test 8: Multi-Topic Message Test ==="

TOPICS=("account" "fund-transfer" "notification")

for topic in "${TOPICS[@]}"; do
    echo "Testing topic: $topic"
    
    # Produce a message
    echo "Test message for $topic" | podman exec -i kafka-1 kafka-console-producer.sh \
        --bootstrap-server localhost:19092 \
        --topic $topic \
        --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";')
    
    # Consume the message
    CONSUMED=$(timeout 5s podman exec kafka-1 kafka-console-consumer.sh \
        --bootstrap-server localhost:19092 \
        --topic $topic \
        --from-beginning \
        --max-messages 1 \
        --consumer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";') 2>/dev/null)
    
    if [ -n "$CONSUMED" ]; then
        echo "✅ Topic '$topic' message flow working"
    else
        echo "❌ Topic '$topic' message flow failed"
    fi
done
```

## 📊 Performance Tests

### Test 9: Basic Performance Test
```bash
echo "=== Test 9: Basic Performance Test ==="

# Performance test with kafka-producer-perf-test
echo "Running producer performance test..."
podman exec kafka-1 kafka-producer-perf-test.sh \
    --topic account \
    --num-records 1000 \
    --record-size 100 \
    --throughput 100 \
    --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";')

if [ $? -eq 0 ]; then
    echo "✅ Producer performance test completed"
else
    echo "❌ Producer performance test failed"
fi

# Performance test with kafka-consumer-perf-test
echo "Running consumer performance test..."
podman exec kafka-1 kafka-consumer-perf-test.sh \
    --topic account \
    --messages 1000 \
    --consumer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";')

if [ $? -eq 0 ]; then
    echo "✅ Consumer performance test completed"
else
    echo "❌ Consumer performance test failed"
fi
```

## 🔄 High Availability Tests

### Test 10: Broker Failover Test
```bash
echo "=== Test 10: Broker Failover Test ==="

# Stop one broker
echo "Stopping kafka-2 to test failover..."
podman stop kafka-2

# Wait for cluster to adjust
sleep 30

# Test if cluster is still functional
echo "Testing cluster functionality with one broker down..."
echo "Failover test message" | podman exec -i kafka-1 kafka-console-producer.sh \
    --bootstrap-server localhost:19092 \
    --topic account \
    --producer.config <(echo 'security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret";')

if [ $? -eq 0 ]; then
    echo "✅ Cluster remains functional with one broker down"
else
    echo "❌ Cluster failed with one broker down"
fi

# Restart the broker
echo "Restarting kafka-2..."
podman start kafka-2

# Wait for broker to rejoin
sleep 30

echo "✅ Failover test completed"
```

## 📈 Monitoring Tests

### Test 11: Metrics Collection Test
```bash
echo "=== Test 11: Metrics Collection Test ==="

# Test Prometheus metrics collection
echo "Testing Prometheus metrics..."
METRICS=$(curl -s http://localhost:9090/api/v1/query?query=up)
if echo "$METRICS" | grep -q '"status":"success"'; then
    echo "✅ Prometheus is collecting metrics"
else
    echo "❌ Prometheus metrics collection failed"
fi

# Test JMX metrics from Kafka
echo "Testing Kafka JMX metrics..."
JMX_METRICS=$(curl -s http://localhost:8080/metrics)
if echo "$JMX_METRICS" | grep -q "kafka_"; then
    echo "✅ Kafka JMX metrics are available"
else
    echo "❌ Kafka JMX metrics are not available"
fi
```

## 🧹 Test Cleanup

### Test 12: Cleanup Test Topics
```bash
echo "=== Test 12: Cleanup Test ==="

# Clean up test data (optional)
echo "Cleaning up test topics..."
TEST_TOPICS=$(podman exec kafka-1 kafka-topics.sh --bootstrap-server localhost:19092 --list | grep -E "^test-")

for topic in $TEST_TOPICS; do
    podman exec kafka-1 kafka-topics.sh --bootstrap-server localhost:19092 --delete --topic $topic
    echo "Deleted test topic: $topic"
done

echo "✅ Cleanup completed"
```

## 📋 Test Summary Script

Create a comprehensive test script:

```bash
cat > run-all-tests.sh << 'EOF'
#!/bin/bash

echo "🧪 Kafka Cluster Comprehensive Test Suite"
echo "=========================================="

TESTS_PASSED=0
TESTS_FAILED=0

run_test() {
    local test_name="$1"
    local test_command="$2"
    
    echo ""
    echo "Running: $test_name"
    echo "----------------------------------------"
    
    if eval "$test_command"; then
        echo "✅ PASSED: $test_name"
        ((TESTS_PASSED++))
    else
        echo "❌ FAILED: $test_name"
        ((TESTS_FAILED++))
    fi
}

# Run all tests
run_test "Cluster Health Check" "podman exec kafka-1 kafka-broker-api-versions.sh --bootstrap-server localhost:19092 > /dev/null"
run_test "Schema Registry" "curl -sf http://localhost:8081/subjects > /dev/null"
run_test "Prometheus Health" "curl -sf http://localhost:9090/-/healthy > /dev/null"
run_test "Grafana Health" "curl -sf http://localhost:3000/api/health > /dev/null"

echo ""
echo "=========================================="
echo "Test Results Summary:"
echo "✅ Passed: $TESTS_PASSED"
echo "❌ Failed: $TESTS_FAILED"
echo "Total: $((TESTS_PASSED + TESTS_FAILED))"
echo "=========================================="

if [ $TESTS_FAILED -eq 0 ]; then
    echo "🎉 All tests passed! Your Kafka cluster is ready for use."
    exit 0
else
    echo "⚠️  Some tests failed. Please check the logs and troubleshoot."
    exit 1
fi
EOF

chmod +x run-all-tests.sh
```

## 🎯 Test Execution

Run the complete test suite:
```bash
./run-all-tests.sh
```

Or run individual test sections as needed.

## 📝 Test Results Documentation

Document your test results:
```bash
# Create test results file
echo "Kafka Cluster Test Results - $(date)" > test-results.txt
echo "======================================" >> test-results.txt
./run-all-tests.sh >> test-results.txt 2>&1
```

This comprehensive testing guide ensures your Kafka cluster is properly configured and functioning correctly across all components.
