---
agent: 'agent'
description: 'Generate comprehensive rollout plans with preflight checks, step-by-step deployment, verification signals, rollback procedures, and communication plans for infrastructure and application changes'
tools: ['codebase', 'terminalCommand', 'search', 'githubRepo']
---

# DevOps Rollout Plan Generator

Your goal is to create a comprehensive, production-ready rollout plan for infrastructure or application changes.

## Input Requirements

Before generating the plan, gather these critical details:

### **1. Change Description**
- What is being changed? (Infrastructure, application, configuration, etc.)
- What version or state are we moving from/to?
- What problem does this solve or feature does it add?

### **2. Environment Details**
- Target environment (dev, staging, production, all)
- Infrastructure type (Kubernetes, VMs, serverless, containers, etc.)
- Affected services and dependencies
- Current capacity and scale

### **3. Constraints & Requirements**
- Acceptable downtime window (zero-downtime, scheduled maintenance, etc.)
- Change window restrictions (business hours, after-hours, weekends)
- Approval requirements (team lead, security, compliance)
- Regulatory or compliance considerations
- Budget constraints

### **4. Risk Assessment**
- What's the blast radius of this change?
- Are there data migrations or schema changes?
- Can this change be rolled back safely?
- What are the known risks?

## Output Format: Comprehensive Rollout Plan

Generate a structured rollout plan with these sections:

---

## Rollout Plan: [Change Name]

### Executive Summary

**Change Overview:**
- **What:** [Brief description of the change]
- **Why:** [Business justification]
- **When:** [Proposed date/time]
- **Duration:** [Estimated time]
- **Risk Level:** [Low/Medium/High]
- **Rollback Time:** [Estimated time to rollback]

**Impact:**
- **Affected Systems:** [List of systems/services]
- **User Impact:** [Expected user experience during rollout]
- **Downtime:** [Expected downtime, if any]

---

### 1. Prerequisites & Approvals

**Required Approvals:**
- [ ] Technical Lead: [Name]
- [ ] Security Team: [If security-related]
- [ ] Compliance Team: [If compliance-related]
- [ ] Change Advisory Board: [If required]
- [ ] Business Owner: [For production changes]

**Required Resources:**
- [ ] Infrastructure capacity verified
- [ ] Backup/snapshot capability confirmed
- [ ] Monitoring and alerting configured
- [ ] Rollback automation tested
- [ ] On-call team notified
- [ ] Communication plan distributed

**Pre-Deployment Backups:**
- [ ] Database backup completed (if applicable)
- [ ] Configuration backup taken
- [ ] State file backup (for IaC changes)
- [ ] Container image tags recorded
- [ ] Current infrastructure state documented

---

### 2. Preflight Checks

**Infrastructure Validation:**
```bash
# Verify cluster/environment health
[Specific commands for the infrastructure type]

# Example for Kubernetes:
kubectl get nodes
kubectl top nodes
kubectl get pods --all-namespaces | grep -v Running

# Example for AWS:
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxxxx \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --period 3600 \
  --statistics Average
```

**Application Health:**
```bash
# Verify application is healthy before change
[Health check commands specific to application]

# Example:
curl -f https://api.example.com/health
# Check error rates in last hour
# Check latency p95/p99
```

**Dependency Check:**
```bash
# Verify all dependencies are available
[Dependency verification commands]

# Examples:
nc -zv database.example.com 5432
curl -f https://external-api.example.com/health
```

**Monitoring Baseline:**
```bash
# Record baseline metrics
[Commands to capture current state]

# Examples:
# - Current request rate
# - Current error rate
# - Current latency percentiles
# - Current resource utilization
```

**Go/No-Go Decision Checklist:**
- [ ] All preflight checks passed
- [ ] Baseline metrics recorded
- [ ] Team ready and on standby
- [ ] Communication sent to stakeholders
- [ ] Rollback plan tested and ready

---

### 3. Step-by-Step Rollout Procedure

**Phase 1: Pre-Deployment** (Duration: [X minutes])

```bash
# Step 1: Notification
[Send deployment start notification]

# Step 2: Enable maintenance mode (if applicable)
[Commands to enable maintenance mode]

# Step 3: Final verification
[Last-minute checks before deployment]
```

**Validation:**
- [ ] Maintenance mode active (if applicable)
- [ ] Stakeholders notified
- [ ] Team ready

---

**Phase 2: Deployment** (Duration: [X minutes])

```bash
# Step 1: Deploy changes
[Specific deployment commands for the change]

# Examples for different scenarios:

# Kubernetes Deployment:
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp -n production --timeout=5m

# Terraform:
terraform plan -out=tfplan
terraform apply tfplan

# Docker Container:
docker pull myapp:1.2.3
docker stop myapp-old
docker run -d --name myapp myapp:1.2.3

# Database Migration:
./migrate.sh up
```

**Validation After Each Step:**
- [ ] Command executed successfully
- [ ] No errors in logs
- [ ] Resources created/updated as expected

---

**Phase 3: Progressive Verification** (Duration: [X minutes])

```bash
# Immediately after deployment (0-2 minutes)
[Quick smoke tests]

# Short-term monitoring (2-5 minutes)
[Monitor key metrics]

# Medium-term monitoring (5-15 minutes)
[Extended verification]
```

---

### 4. Verification Signals

**Critical Success Indicators:**

**Immediate (0-2 minutes):**
- [ ] Deployment command completed successfully
- [ ] Pods/containers started successfully
- [ ] Health checks passing
- [ ] No crash loops or restart spikes

```bash
# Commands to verify immediate success
[Specific commands]

# Examples:
kubectl get pods -n production -l app=myapp
kubectl logs -n production -l app=myapp --tail=50 | grep -i error
```

**Short-term (2-5 minutes):**
- [ ] Application responding to requests
- [ ] Error rate within acceptable threshold (< X%)
- [ ] Latency within acceptable range (p95 < Xms)
- [ ] No critical alerts fired

```bash
# Monitor key metrics
[Monitoring commands or dashboard links]

# Examples:
curl -f https://api.example.com/health
# Check Prometheus/Grafana dashboard
# Check error rate: should be < 0.1%
# Check latency p95: should be < 200ms
```

**Medium-term (5-15 minutes):**
- [ ] Sustained healthy metrics
- [ ] Database connections stable
- [ ] No memory leaks detected
- [ ] External integrations working
- [ ] User transactions completing successfully

```bash
# Extended verification
[Extended monitoring commands]

# Examples:
# Monitor memory usage trends
# Check for database connection pool exhaustion
# Verify external API calls succeeding
# Sample user transactions
```

**Long-term (15+ minutes):**
- [ ] No degradation over time
- [ ] Capacity metrics healthy
- [ ] No unusual patterns in logs
- [ ] Business metrics normal

**Automated Monitoring:**
```bash
# Set up automated monitoring during rollout
[Automated monitoring setup]

# Example:
watch -n 10 'kubectl top pods -n production -l app=myapp'
# Monitor Datadog/New Relic dashboard
# Set up temporary high-sensitivity alerts
```

---

### 5. Rollback Procedure

**Rollback Decision Criteria:**

Initiate rollback immediately if ANY of these occur:
- [ ] Error rate > [X%] for > [Y minutes]
- [ ] Latency p95 > [X ms] for > [Y minutes]
- [ ] Critical functionality broken
- [ ] Data corruption detected
- [ ] Security vulnerability exposed
- [ ] More than [X%] of pods/containers failing

**Rollback Steps:**

**Option 1: Automated Rollback** (Fastest - [X minutes])

```bash
# Kubernetes rollback
kubectl rollout undo deployment/myapp -n production
kubectl rollout status deployment/myapp -n production --timeout=5m

# Verify rollback
kubectl get pods -n production -l app=myapp
kubectl logs -n production -l app=myapp --tail=50
```

**Option 2: Revert Infrastructure Changes** ([X minutes])

```bash
# Terraform rollback
git revert HEAD
terraform plan -out=rollback-plan
terraform apply rollback-plan

# Docker rollback
docker stop myapp
docker start myapp-old

# Database rollback (if applicable)
./migrate.sh down
# OR restore from backup
```

**Option 3: Full Restore from Backup** (Slowest - [X minutes])

```bash
# Restore database
[Database restore commands]

# Restore configuration
[Configuration restore commands]

# Redeploy previous version
[Redeployment commands]
```

**Post-Rollback Verification:**
```bash
# Verify system is back to healthy state
[Verification commands matching original preflight checks]

- [ ] Application health restored
- [ ] Error rates back to baseline
- [ ] Latency back to baseline
- [ ] All services responding
```

**Rollback Communication:**
```text
Subject: ROLLBACK INITIATED - [Change Name]

Team,

We have initiated a rollback of [Change Name] due to [Reason].

Status: [In Progress / Complete]
Expected Completion: [Time]
Impact: [Description]

Next Steps:
1. [Action item]
2. [Action item]

Post-mortem scheduled for: [Date/Time]
```

---

### 6. Communication Plan

**Pre-Deployment Communication** (T-24 hours):

```text
Subject: Scheduled Deployment - [Change Name] - [Date/Time]

Team/Stakeholders,

We will be deploying [Change Name] on [Date] at [Time].

What's Changing:
- [Brief description]

Expected Impact:
- [User impact, if any]
- [Downtime, if any]

Duration: [Estimated time]

Rollback Plan: [Brief description]

Contact: [On-call engineer] ([Contact info])
```

**Deployment Start Communication**:

```text
Subject: DEPLOYMENT STARTED - [Change Name]

Deployment has commenced. Expected completion: [Time]

Status updates will be sent every [X] minutes.

Current Status: [Phase name]
```

**Progress Updates** (Every X minutes):

```text
Subject: DEPLOYMENT IN PROGRESS - [Change Name] - [% Complete]

Current Phase: [Phase name]
Status: [On track / Delayed / Issues]

Metrics:
- Error Rate: [X%] (Target: <[Y%])
- Latency p95: [X ms] (Target: <[Y ms])
- Pods Healthy: [X/Y]

Next Phase: [Phase name] (ETA: [X] minutes)
```

**Deployment Success Communication**:

```text
Subject: DEPLOYMENT COMPLETE - [Change Name] ✅

The deployment has completed successfully.

Completion Time: [Time]
Final Metrics:
- Error Rate: [X%]
- Latency p95: [X ms]
- All health checks: PASSING

Monitoring will continue for the next [X] hours.

Thank you for your patience.
```

**Rollback Communication** (If needed):

```text
Subject: ROLLBACK INITIATED - [Change Name] ⚠️

We have initiated a rollback of the deployment due to [Reason].

Rollback Status: [In Progress / Complete]
Estimated Completion: [Time]
Impact: [Description]

Root Cause Analysis: [Date/Time]

We apologize for any inconvenience.
```

**Stakeholder Communication Matrix:**

| Audience | When | Method | Content |
|----------|------|--------|---------|
| Engineering Team | T-24h, Start, Every 15min, Complete | Slack #deployments | Technical details |
| Support Team | T-24h, Start, Complete | Email + Slack | User impact info |
| Management | T-24h, Complete | Email | Executive summary |
| Customers | If user-facing impact | Status page | Non-technical impact |

---

### 7. Post-Deployment Tasks

**Immediate (Within 1 hour):**
- [ ] Verify all success criteria met
- [ ] Review deployment logs for warnings
- [ ] Confirm all tests passing
- [ ] Update documentation
- [ ] Send completion communication

**Short-term (Within 24 hours):**
- [ ] Monitor metrics for degradation
- [ ] Review error logs for new patterns
- [ ] Check capacity and scaling
- [ ] Verify backup processes working
- [ ] Update runbooks if needed

**Medium-term (Within 1 week):**
- [ ] Conduct post-deployment review
- [ ] Document lessons learned
- [ ] Update rollout procedures
- [ ] Share knowledge with team
- [ ] Close deployment ticket

**Post-Deployment Review Questions:**
1. Did the deployment go as planned?
2. Were estimates accurate?
3. Were there any surprises?
4. What went well?
5. What could be improved?
6. Any action items for future deployments?

---

### 8. Contingency Plans

**Scenario 1: Partial Failure**
- Symptoms: [Describe]
- Response: [Action]
- Timeline: [Duration]

**Scenario 2: Performance Degradation**
- Symptoms: [Describe]
- Response: [Action]
- Timeline: [Duration]

**Scenario 3: Data Inconsistency**
- Symptoms: [Describe]
- Response: [Action]
- Timeline: [Duration]

**Scenario 4: External Dependency Failure**
- Symptoms: [Describe]
- Response: [Action]
- Timeline: [Duration]

---

### 9. Contact Information

**Primary On-Call:**
- Name: [Name]
- Phone: [Number]
- Email: [Email]
- Slack: @[handle]

**Secondary On-Call:**
- Name: [Name]
- Phone: [Number]
- Email: [Email]
- Slack: @[handle]

**Escalation Path:**
1. On-call engineer
2. Team Lead: [Name]
3. Engineering Manager: [Name]
4. VP Engineering: [Name]

**Emergency Contacts:**
- Infrastructure: [Contact]
- Security: [Contact]
- Database: [Contact]
- Networking: [Contact]

---

### 10. Appendix

**Useful Commands:**
```bash
# Monitoring
[List of monitoring commands]

# Troubleshooting
[List of common troubleshooting commands]

# Logs
[How to access logs]
```

**Dashboards:**
- Application Metrics: [URL]
- Infrastructure Metrics: [URL]
- Error Tracking: [URL]
- User Analytics: [URL]

**Runbooks:**
- Service Restart: [Link]
- Database Failover: [Link]
- Cache Clear: [Link]

**Related Documentation:**
- Architecture Diagram: [Link]
- Service Dependencies: [Link]
- Incident Response Plan: [Link]

---

## Plan Customization

Adapt this template based on:

1. **Infrastructure Type:**
   - Kubernetes: Focus on pod health, rolling updates, PDBs
   - VMs: Focus on load balancer draining, health checks
   - Serverless: Focus on function versions, gradual rollout
   - Databases: Focus on migrations, replication lag, backups

2. **Risk Level:**
   - **Low Risk:** Simplify verification, shorter monitoring
   - **Medium Risk:** Standard process as above
   - **High Risk:** Add additional approval gates, extended monitoring, phased rollout

3. **Change Type:**
   - **Code Deployment:** Focus on application metrics
   - **Infrastructure Change:** Focus on resource metrics
   - **Configuration Change:** Focus on behavior validation
   - **Data Migration:** Focus on data integrity

4. **Environment:**
   - **Production:** Full plan as above
   - **Staging:** Simplified approvals, but full technical process
   - **Development:** Minimal approvals, rapid verification

---

## Remember

- ✅ Always have a tested rollback plan
- ✅ Communicate early and often
- ✅ Monitor metrics, not just logs
- ✅ Document everything
- ✅ Learn from each deployment
- ❌ Never deploy on Friday afternoon (unless critical)
- ❌ Never skip verification steps
- ❌ Never assume "it should work"
