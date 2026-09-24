# AWS 3-Tier Application Migration

## 1. Objective

Migrate the production 3-tier application from AWS Account A to an independent AWS Account B with near-zero downtime and zero data loss as the target design objectives.

**Target:** RPO ≈ 0 | Minimal cutover downtime

The source environment will remain operational during migration and will be retained through the defined stabilization and rollback period.

---

## 2. Current Architecture

Route 53
   ↓
Public ALB
   ↓
Frontend EC2
   ↓
Backend EC2
   ↓
RDS

---

## 3. Migration Strategy

The target environment will be built in parallel with the existing production environment.

The migration will follow these stages:

1. Build the target network and security foundation.
2. Establish temporary connectivity between source and target.
3. Provision the target compute and database environment.
4. Migrate the database using AWS DMS.
5. Validate the target application and database.
6. Perform a controlled production cutover.
7. Monitor the target during the stabilization period.
8. Decommission the source environment after successful validation.

---

## 4. Target VPC

The target VPC will replicate the source network's logical structure while using a non-overlapping CIDR range.

The VPC will span at least two Availability Zones with separate public, private application, and private database subnets.

* **Public subnets:** Internet-facing ALB
* **Private application subnets:** Frontend, Backend and migration components such as DMS
* **Private database subnets:** RDS
* **Internet Gateway:** Public subnet connectivity
* **NAT Gateway:** Controlled outbound connectivity from private subnets

The target CIDR and subnet ranges will be validated against existing source, corporate and connected networks before deployment.

### NAT Gateway

A NAT Gateway will be deployed in each Availability Zone to provide highly available outbound internet connectivity for private application subnets.

Each private subnet will route internet-bound traffic through the NAT Gateway in its own AZ.

This avoids a single-AZ NAT dependency. The additional NAT Gateway cost is accepted to meet the availability requirement.

---

## 5. Cross-Account Connectivity

Temporary VPC Peering will be used to connect the source and target VPCs for database migration using AWS DMS.

The source and target VPCs will use non-overlapping CIDR ranges. Routes will be added only for the required network ranges, and security controls will restrict database connectivity to the DMS migration traffic.

VPC Peering is preferred because the migration involves a single source and target VPC. If existing corporate, Transit Gateway or shared-services routing requirements exceed VPC Peering limitations, Transit Gateway or VPN-based connectivity will be evaluated.

The peering connection will be removed after migration and the rollback/stabilization period if it is no longer required.

---

## 6. Security Groups

Security groups will enforce tier-to-tier access using least privilege.

* **ALB-SG:** Internet → ALB on HTTPS.
* **Frontend-SG:** Traffic accepted only from ALB-SG.
* **Backend-SG:** Traffic accepted only from Frontend-SG.
* **DB-SG:** Database traffic accepted only from Backend-SG.
* **DMS-SG:** Temporary database access to source and target RDS
  during migration.

No application or database instance will require direct public inbound access.

---

## 7. Secrets Management

Application secrets and DB credentials will not be copied as-is. New credentials will be provisioned in Account B's Secrets Manager (or Parameter Store, matching current tooling) and referenced by the application via the same mechanism used in Account A.

Database credentials used by DMS (source read, target write) will be scoped to migration-only permissions and rotated/revoked after cutover.

Any hardcoded secrets or credentials discovered during migration (common in older EC2-based deployments) will be flagged and moved to Secrets Manager as part of this migration rather than carried forward as-is.

---

## 8. Compute Migration

Frontend and backend EC2 instances will be recreated in the target account rather than directly moved between accounts.

The preferred approach is to provision the target instances using the existing infrastructure and deployment automation, where available, and deploy the same application version and configuration used in production.

If an existing CI/CD or deployment mechanism is not available, the application can be deployed using approved application artifacts or validated AMIs from the source environment. The selected approach will preserve the application version, configuration and runtime dependencies of the source environment.

Frontend and backend EC2 instances will be managed using Auto Scaling Groups distributed across multiple Availability Zones. The frontend
ASG will be registered with the ALB target group, while the backend ASG will preserve the existing backend traffic and service-discovery mechanism.

Frontend instances will be distributed across Availability Zones behind the ALB.

The backend traffic model and service-discovery mechanism will initially be preserved from the source architecture. An internal ALB and independent backend scaling can be introduced as a future improvement.

---

## 9. ALB Migration

The application load balancer will be recreated in the target account using equivalent listener, security group and health-check configuration.

The new ALB will target the newly provisioned frontend EC2 instances and will be fully validated before production traffic is redirected.

If HTTPS is used, a corresponding ACM certificate will be provisioned and validated in the target account.

The source ALB will remain available during the migration and stabilization period to support rollback.

---

## 10. Database Migration

Before enabling CDC, source database CDC prerequisites, logging requirements and engine compatibility will be validated.

AWS DMS will perform an initial Full Load from the source RDS followed by Change Data Capture (CDC) to continuously replicate ongoing changes to the target RDS.

The target RDS engine version, parameter configuration and required database features will be verified for compatibility with the application before migration.

The target database will be validated while replication continues.

For cutover, application writes will be temporarily controlled, DMS replication will be allowed to reach near-zero lag, and the target database will be validated before the application is switched to the target environment.

**Goal:** RPO ≈ 0 with minimal cutover downtime.

The final achievable RPO/RTO will depend on database engine capabilities, replication performance, application behavior and the agreed migration procedure.

---

## 11. Validation

Before production cutover, the target environment will be validated against the source application's expected behavior.

Validation will include:

- Application functional testing
- End-to-end transaction testing
- Database consistency checks
- Performance verification
- Security and connectivity validation
- Smoke testing through the target ALB

Migration will proceed to cutover only after the defined validation criteria are met.

---

## 12. Cutover & Rollback

Before cutover, the target environment and database will be fully validated while AWS DMS continuously replicates source changes.

During the migration window:

1. Freeze application writes.
2. Take and verify a final source RDS snapshot.
3. Allow DMS CDC to reach near-zero replication lag.
4. Validate the target database.
5. Switch the application to the target RDS.
6. Perform application smoke tests.
7. Redirect Route 53 traffic to the target ALB.
8. Resume writes after validation.
9. Monitor the target during a defined stabilization window.

The source environment and database will remain available during the rollback window.

Once application writes begin on the target database, rollback to the source database becomes a manual recovery activity requiring reconciliation of target-side changes.

Application rollback and database rollback should therefore be treated as independent operations.

---

## 13. Route 53 Cutover

The existing Route 53 hosted zone will remain unchanged during the initial application migration.

DNS TTL will be reduced before the migration window to minimize cache duration during cutover.

After target database and application validation is complete, the existing DNS record will be updated to point to the target ALB.

The source ALB and application environment will remain available during the stabilization and rollback period.

The Route 53 hosted zone may be migrated to the target account as a separate post-migration activity if required.

---

## 14. Post-Migration

After the stabilization and rollback period:

* Confirm application and database stability.
* Remove temporary VPC Peering.
* Remove DMS migration infrastructure.
* Decommission source resources after customer approval.
* Restore normal DNS TTL.
* Review resource sizing and cost optimization opportunities.

---

## 15. Well-Architected Considerations

### Security

* Private application and database tiers.
* Least-privilege security groups.
* No direct public access to application or database instances.

### Reliability

* Multi-AZ target architecture.
* Continuous database replication.
* Controlled cutover.
* Source environment retained for rollback.

### Operational Excellence

* Parallel migration approach.
* Controlled validation and cutover.
* Reuse of existing deployment/automation mechanisms where possible.

### Performance Efficiency

* Target architecture maintains the existing application flow.
* Frontend instances distributed across Availability Zones.

### Cost Optimization

* Avoid unnecessary architectural changes during migration.
* Temporary migration resources will be removed after stabilization.
* NAT Gateway redundancy is accepted to meet availability
  requirements.

### Sustainability

* Reuse the existing application architecture where appropriate.
* Right-size target resources after migration based on observed
  usage.
