# Kafka 4.2 AMI Guide

This guide explains how the Kafka 4.2 AMI is built, how it configures itself at first boot, and how the repository is organised.

## Architecture

Kafka 4.2 runs in KRaft mode; a single process acts as both broker and controller, so there is no ZooKeeper.

This single-node build runs Kafka and Kafdrop on one instance, with data on a dedicated LVM/XFS volume at `/data/kafka`.

| Port | Service | Exposure |
|------|---------|----------|
| 9092 | Kafka broker (PLAINTEXT) | instance-local / VPC |
| 9093 | KRaft controller | internal to the instance |
| 8082 | Kafdrop UI | SSM tunnel only |

## What Changed From Older Estate Patterns

### Removed or Replaced

- `broker.id` replaced by `node.id`
- `zookeeper.connect` removed
- `zookeeper.connection.timeout.ms` removed
- `advertised.host.name` removed

### Added for Kafka 4.2 / KRaft

- `process.roles=broker,controller`
- `controller.listener.names=CONTROLLER`
- `controller.quorum.bootstrap.servers=...`
- `listener.security.protocol.map=CONTROLLER:PLAINTEXT,...`
- `listeners=PLAINTEXT://:9092,CONTROLLER://127.0.0.1:9093`
- `share.coordinator.state.topic.*` settings (Kafka 4.x features)

## First Boot

`kafka-bootstrap.service` runs once before `kafka.service`:

1. Reads the instance private IP from EC2 metadata (IMDSv2).
2. Sets `advertised.listeners` to that IP for the single-node build.
3. Generates a cluster ID once and caches it in `/etc/kafka-clusterid`.
4. Formats the KRaft metadata log with `kafka-storage.sh`.

When the `kafka-streaming` Terraform module supplies `server.properties` with a controller voter list and cluster ID, the bootstrap keeps that config and formats the shared quorum instead of a standalone node.

Key files:

- [kafka-bootstrap.sh.j2](ansible/roles/kraft/templates/usr/local/bin/kafka-bootstrap.sh.j2)
- [kafka-bootstrap.service.j2](ansible/roles/kraft/templates/etc/systemd/system/kafka-bootstrap.service.j2)
- [kraft/tasks/main.yml](ansible/roles/kraft/tasks/main.yml)

## Folder Structure (Simple View)

```text
kafka-ami/
|- README.md
|- KAFKA-4.2-README.md
|- version
|- packer/
|  |- variables.pkr.hcl
|  |- sources.pkr.hcl
|  |- build.pkr.hcl
|- ansible/
   |- playbook.yml
   |- group_vars/kafka/vars
   |- roles/
      |- kafka/
      |  |- defaults/main.yml
      |  |- tasks/install_kafka.yml
      |  |- templates/opt/kafka/config/conf.d/10-common.properties.j2
      |  |- templates/opt/kafka/config/conf.d/30-durability.properties.j2
      |  |- templates/etc/systemd/system/kafka.service.j2
      |- kraft/
      |  |- defaults/main.yml
      |  |- tasks/main.yml
      |  |- templates/opt/kafka/config/conf.d/20-kraft.properties.j2
      |  |- templates/usr/local/bin/kafka-bootstrap.sh.j2
      |  |- templates/etc/systemd/system/kafka-bootstrap.service.j2
      |- kafka-ui/
         |- tasks/
         |- templates/
```

## What Each Main Folder Does

- `packer/`: builds the AMI image
- `ansible/`: installs Kafka and system services into the AMI
- `ansible/roles/kafka/`: shared broker setup
- `ansible/roles/kraft/`: KRaft-specific config and first-boot logic
- `ansible/roles/kafka-ui/`: Kafdrop UI service

Kafka application logs are written to `/var/log/kafka` and rotated by the
logrotate role. The AMI ships with a 1 GiB heap default. A launch process can
override it by writing `KAFKA_HEAP_OPTS` to `/etc/sysconfig/kafka.override`
before starting or restarting `kafka.service`.

The hardened source AMI filter and owner are supplied by the build environment
because they are account-specific. Terraform or cloud-init launch wiring is
outside this repository.

## Deploy & Test Lifecycle

This is for validating a single AMI instance directly. In a real deployment the `kafka-streaming` Terraform module gives each broker a stable DNS name (Route53, or `nsupdate` where Route53 is unavailable) and clients connect to those names on 9092 — no port forwarding. The SSM tunnel below is only to reach the loopback Kafdrop UI on a standalone test instance.

The full loop — launch, verify, access Kafdrop, tear down. Set these once per
session; everything below uses them:

```bash
export AWS_PROFILE=development-eu-west-2
export AWS_REGION=eu-west-2
AMI=ami-XXXXXXXXXXXXXXXXX        # current AMI from the pipeline build log
NAME=kafka-4.2-test              # instance Name tag
```

### 1. Launch

```bash
aws ec2 run-instances \
  --image-id $AMI \
  --instance-type t3.medium \
  --subnet-id subnet-0bc087c8ba49b0c48 \
  --iam-instance-profile Name=ci-devops-web \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$NAME}]" \
  --query 'Instances[0].{ID:InstanceId,IP:PrivateIpAddress}' \
  --output table
```

Note the instance ID, then:

```bash
ID=i-XXXXXXXXXXXXXXXXX
```

Wait 3–4 minutes for first boot (bootstrap, Kafka, and Kafdrop).

### 2. Connect (SSM — no SSH keys needed)

```bash
eval "$(aws configure export-credentials --profile $AWS_PROFILE --format env)"
aws ssm start-session --target $ID --region $AWS_REGION
```

### 3. Verify (run inside the SSM session)

Stops at the first failing check; ends with `ALL CHECKS PASSED` when green:

```bash
echo "=== 1. Services ===" && \
systemctl is-active kafka kafka-bootstrap kafdrop && \
echo "=== 2. Kafka version ===" && \
ls /opt/kafka/libs/kafka_2.13-*.jar | head -1 && \
echo "=== 3. Java in use ===" && \
for svc in kafka kafdrop; do pid=$(systemctl show -p MainPID --value $svc); echo "$svc -> $(sudo readlink -f /proc/$pid/exe)"; done && \
echo "=== 4. KRaft quorum ===" && \
/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status && \
echo "=== 5. Advertised listener (instance IP, not localhost) ===" && \
grep ^advertised.listeners /opt/kafka/config/server.properties && \
echo "=== 6. Data volume ===" && \
df -h /data/kafka | tail -1 && \
echo "=== 7. Broker answers ===" && \
/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list && \
echo "=== 8. Kafdrop ===" && \
curl -s -o /dev/null -w "Kafdrop (8082): %{http_code}\n" localhost:8082 && \
echo "=== ALL CHECKS PASSED ==="
```

Functional round-trip (creates, produces, consumes, deletes a topic — safe to
re-run):

```bash
T=verify-$(date +%s) && \
/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic $T --partitions 1 --replication-factor 1 && \
echo "roundtrip-ok" | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic $T && \
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic $T --from-beginning --max-messages 1 && \
/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic $T && \
echo "=== ROUND-TRIP PASSED (topic cleaned up) ==="
```

A fresh instance should pass everything with no manual intervention.

### 4. Access Kafdrop (SSM port forwarding)

Kafdrop is not exposed on the network; access it through an SSM tunnel.

```bash
# Kafdrop -> http://localhost:8082
eval "$(aws configure export-credentials --profile $AWS_PROFILE --format env)"
aws ssm start-session --target $ID \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8082"],"localPortNumber":["8082"]}' \
  --region $AWS_REGION
```

### 5. Terminate when done

Volumes are delete-on-termination — nothing is left behind:

```bash
aws ec2 terminate-instances --instance-ids $ID
```

#### Compatibility 

- Application-level producer and consumer flows should be familiar to Kafka 2.x/3.x users
- This configuration is single-node (`replication_factor=1`, `min_insync_replicas=1`) and does not provide production resilience
- A production deployment requires a multi-node quorum, secured listeners, and stricter durability controls