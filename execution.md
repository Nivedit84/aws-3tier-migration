Migration Execution Steps

The migration will be executed in controlled phases, with the source
environment remaining operational until the target environment has
been validated and accepted.

### Phase 1 — Source Discovery & Assessment

1. Inventory the existing AWS resources and dependencies.
2. Document the current VPC, subnets, route tables, security groups,
   ALB, EC2 instances and RDS configuration.
3. Identify application dependencies, ports, DNS records, certificates,
   secrets and external integrations.
4. Review the existing deployment mechanism and application artifacts.
5. Identify the source RDS engine, version, configuration and database
   size.
6. Identify any existing infrastructure-as-code.
7. Use tools such as **Former2** where appropriate to help capture the
   existing AWS resource configuration and establish a baseline.

The output of this phase is a validated source-environment inventory
and migration dependency map.

---

### Phase 2 — Target Account Foundation

1. Establish the security baseline in Account B.
2. Configure appropriate IAM roles and least-privilege access.
3. Enable required logging, auditing and security monitoring.
4. Configure account-level governance and required security controls.
5. Establish the target AWS region and Availability Zones.
6. Configure Terraform state management and the target deployment
   workflow.

No production application traffic is introduced at this stage.

---

### Phase 3 — Target Infrastructure

1. Create the target VPC using a non-overlapping CIDR.
2. Create public, private application and private database subnets
   across multiple AZs.
3. Configure route tables, Internet Gateway and NAT Gateways.
4. Configure security groups according to the application flow.
5. Create the target Multi-AZ RDS instance.
6. Create the target ALB and target groups.
7. Provision frontend and backend EC2 instances.
8. Configure required IAM instance roles and application access.
9. Configure Secrets Manager/Parameter Store for application
   credentials and secrets.

Terraform will be used to make the target infrastructure
repeatable and auditable.

---

### Phase 4 — Application Deployment

1. Deploy the same approved application version used in the source
   environment.
2. Reproduce required application configuration and runtime
   dependencies.
3. Configure the application to use the target RDS.
4. Register frontend instances with the target ALB.
5. Validate frontend → backend → database connectivity.
6. Perform basic application functional testing.

Production traffic remains on the source environment.

---

### Phase 5 — Cross-Account Database Migration

1. Establish temporary VPC Peering between the source and target
   VPCs.
2. Configure the required routes and security controls.
3. Provision DMS replication infrastructure in the target environment.
4. Configure source and target database endpoints.
5. Validate CDC prerequisites and database compatibility.
6. Start DMS Full Load.
7. Validate the migrated data.
8. Enable/continue CDC.
9. Monitor replication health and lag while the source database
   continues serving production traffic.

The target database is continuously synchronized while the source
remains authoritative.

---

### Phase 6 — Target Environment Validation

Validate the target environment before production cutover.

Validation includes:

* Application functional testing
* End-to-end transaction testing
* Database consistency checks
* Frontend → Backend → RDS connectivity
* ALB health checks
* Performance verification
* Security and access validation
* Application logging and monitoring
* Critical business transaction testing

Migration proceeds only after the target environment meets the
agreed validation criteria.

---

### Phase 7 — Production Cutover

1. Confirm the migration window and stakeholder approval.
2. Freeze or control application writes on the source environment.
3. Take and verify a final source RDS snapshot.
4. Allow DMS CDC to catch up to the agreed near-zero lag threshold.
5. Validate the final target database state.
6. Update application configuration to use the target RDS and verify successful connectivity.
7. Perform application smoke tests.
8. Update Route 53 to point to the target ALB.
9. Validate production traffic.
10. Resume normal application writes.
11. Monitor the target environment during the stabilization period.

The source environment remains available throughout this period.

---

### Phase 8 — Stabilization & Decommissioning

1. Monitor application, ALB, EC2 and RDS health.
2. Monitor error rates, latency and critical business transactions.
3. Confirm database consistency after production traffic is enabled.
4. Retain the source environment for the agreed rollback period.
5. Remove temporary DMS infrastructure after migration stabilization.
6. Remove temporary VPC Peering.
7. Restore normal DNS TTL.
8. Obtain customer/business approval before source decommissioning.
9. Retain required source backups according to the agreed retention
   policy.
10. Decommission the source environment.

The migration is considered complete only after the target
environment has been accepted and the rollback period has expired.
