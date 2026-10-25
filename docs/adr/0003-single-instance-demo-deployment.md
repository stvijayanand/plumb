---
status: accepted
---

# Single-instance demo deployment instead of managed AWS services

Plumb runs as one Docker Compose stack: on the laptop for development and screen-share interviews, and on one on-demand t4g.xlarge EC2 instance (4 vCPU, 16 GB, ARM) in us-west-2 behind a Cloudflare Tunnel only for scheduled demos and load tests. `make demo-up` creates it with Terraform and `make demo-down` destroys it. We chose this because the expensive parts of AWS are managed services that bill by the hour whether or not anyone is using them, and a portfolio project is idle almost all the time. Running Kafka, Postgres, Redis and Prometheus/Grafana as containers on one instance keeps idle cost under $1 a month and a heavy month around $7.50, while still showing the Terraform and a live link. The outline's ECS or EKS deployment is replaced.

## Considered Options

Hourly prices are us-west-2 on-demand, taken from the map's *Cost plan*, which holds the full table.

- **EKS or ECS:** EKS is $0.10 per cluster-hour (about $73 a month) before any workload runs. Rejected: orchestration isn't the signal this project is built to show.
- **MSK:** MSK Serverless is $0.75 per cluster-hour plus partition-hours. Replaced by a single-broker Kafka container in KRaft mode.
- **RDS:** bills hourly while running, and a stopped instance restarts itself after 7 days. Replaced by a Postgres container.
- **ElastiCache:** replaced by a Redis container.
- **ALB:** billed hourly. Replaced by Cloudflare Tunnel, which gives TLS and a stable hostname with no inbound ports.
- **NAT gateway:** $0.045 an hour plus $0.045 per GB processed. Confirmed on 2026-10-24 against AWS's official Price List API (AmazonEC2 offer for us-west-2, version 20261023221355, usage types `USW2-NatGateway-Hours` and `USW2-NatGateway-Bytes`). Avoided: there are no private subnets, and the tunnel is outbound-only.
- **CloudWatch:** replaced by Prometheus and Grafana containers.

Plumb's own costs are the instance at $0.1344 an hour, its auto-assigned public IPv4 address at $0.005 an hour (no Elastic IP), and, during load tests only, a c7g.large k6 generator at $0.0725 an hour plus its own IP: about $0.14 an hour for a demo.

## Guardrails

- The boot script schedules a shutdown after 3 hours, with shutdown behaviour set to terminate. The load generator gets the same timer.
- A nightly GitHub Actions workflow runs `demo-down`, which destroys both instances, lists anything still tagged `project=plumb`, and fails loudly if something is left.
- AWS Budgets alert at $10 and $25 a month. The bill is checked by the `project=plumb` tag once a month.
- A separate AWS account for Plumb, with a region-deny policy (an SCP or an `aws:RequestedRegion` condition) blocking every region except us-west-2.
- The security group has no inbound rules. The one exception: `make loadtest-up` adds a single rule that allows only the generator's security group, over private IPs in the same availability zone.
- **Default deny list:** no EKS, MSK, RDS, NAT gateway, load balancer or Elastic IP. Each exception needs its own ADR.
- Destroy rather than stop. Everything is rebuilt from code and seed data, and a stopped instance still bills for its disk.

## Consequences

- No managed HA or failover. One instance, one broker and one Postgres with no replication, so an instance failure ends the demo. Failure drills exercise Plumb's own handling (PSP chaos, risk service outages), not AWS's.
- Load-test numbers describe one shared 4 vCPU box, not the architecture. The load-test report presents them as relative measurements in a stated reference environment (see *Load-test methodology* on the map). Check the t4g credit mode (standard or unlimited) before every run. Run the generator off-box. Rely on same-box comparisons. An optional one-off run on a non-burstable instance can give a production-like number, priced first.
- The SLO latency targets apply to this reference environment, not to production.
