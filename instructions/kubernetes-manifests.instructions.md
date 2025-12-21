---
applyTo: 'k8s/**/*.yaml,k8s/**/*.yml,manifests/**/*.yaml,manifests/**/*.yml,deploy/**/*.yaml,deploy/**/*.yml,charts/**/templates/**/*.yaml,charts/**/templates/**/*.yml'
description: 'Best practices for Kubernetes YAML manifests including labeling conventions, security contexts, pod security, resource management, probes, and validation commands'
---

# Kubernetes Manifests Instructions

## Your Mission

Create production-ready Kubernetes manifests that prioritize security, reliability, and operational excellence with consistent labeling, proper resource management, and comprehensive health checks.

## Labeling and Annotations Conventions

### **Required Labels**

Every Kubernetes resource should include these standard labels:

```yaml
metadata:
  labels:
    # Kubernetes recommended labels
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: helm  # or kubectl, argocd, flux
    
    # Environment and ownership
    environment: production
    team: platform
    cost-center: engineering
```

### **Recommended Annotations**

```yaml
metadata:
  annotations:
    # Documentation
    description: "Main backend API service"
    documentation: "https://wiki.company.com/myapp"
    
    # Contact information
    owner: "platform-team@company.com"
    oncall: "platform-oncall@company.com"
    
    # Monitoring and observability
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
    
    # Change tracking
    deployed-by: "argocd"
    git-commit: "abc123def456"
    deployment-date: "2024-01-15T10:30:00Z"
```

## SecurityContext Defaults

### **Pod-Level Security Context**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  # Pod-level security settings
  securityContext:
    # Run as non-root user
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    
    # Filesystem settings
    fsGroupChangePolicy: "OnRootMismatch"
    
    # Seccomp profile
    seccompProfile:
      type: RuntimeDefault
    
    # SELinux options (if applicable)
    seLinuxOptions:
      level: "s0:c123,c456"
  
  containers:
  - name: app
    image: myapp:1.2.3
    
    # Container-level security overrides
    securityContext:
      # Privilege escalation
      allowPrivilegeEscalation: false
      
      # Capabilities
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # Only if needed for ports < 1024
      
      # Filesystem
      readOnlyRootFilesystem: true
      
      # User override (if different from pod)
      runAsUser: 1000
      runAsGroup: 3000
```

### **Read-Only Root Filesystem**

When using `readOnlyRootFilesystem: true`, mount tmpfs for writable directories:

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.2.3
    
    securityContext:
      readOnlyRootFilesystem: true
    
    volumeMounts:
    # Temporary directories
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
    - name: logs
      mountPath: /app/logs
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: logs
    emptyDir: {}
```

## Pod Security Standards

### **Restricted Profile (Recommended for Production)**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted pod security standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### **Pod Security Policy Migration**

For clusters transitioning from PSP to Pod Security Admission:

```yaml
# Old PSP (deprecated)
# apiVersion: policy/v1beta1
# kind: PodSecurityPolicy

# New: Use Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
```

## Resource Requests and Limits

### **Always Define Resources**

```yaml
containers:
- name: app
  image: myapp:1.2.3
  
  resources:
    # Guaranteed minimum resources
    requests:
      cpu: "100m"      # 0.1 CPU core
      memory: "128Mi"  # 128 MiB
    
    # Maximum allowed resources
    limits:
      cpu: "500m"      # 0.5 CPU core
      memory: "512Mi"  # 512 MiB
```

### **QoS Classes**

```yaml
# QoS: Guaranteed (requests == limits)
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# QoS: Burstable (requests < limits or only requests)
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# QoS: BestEffort (no resources defined) - NOT RECOMMENDED
# resources: {}
```

### **Resource Sizing Guidelines**

```yaml
# Small microservice (API, cache)
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

# Medium application (web server, worker)
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Large application (database, data processing)
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"
```

## Health Probes

### **Liveness Probe**

Determines if container should be restarted:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Health-Check
      value: liveness
  
  initialDelaySeconds: 30  # Wait before first check
  periodSeconds: 10        # Check every 10 seconds
  timeoutSeconds: 5        # Timeout after 5 seconds
  failureThreshold: 3      # Restart after 3 failures
  successThreshold: 1      # Back to healthy after 1 success
```

### **Readiness Probe**

Determines if container can receive traffic:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  
  initialDelaySeconds: 5   # Check sooner than liveness
  periodSeconds: 5         # Check more frequently
  timeoutSeconds: 3
  failureThreshold: 2      # Fail faster than liveness
  successThreshold: 1
```

### **Startup Probe**

For slow-starting applications (protects liveness probe):

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 30  # 5 minutes total (30 * 10 seconds)
  successThreshold: 1
```

### **Probe Types**

```yaml
# HTTP probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    scheme: HTTPS  # Use HTTPS if applicable

# TCP probe
livenessProbe:
  tcpSocket:
    port: 8080

# Command probe
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - pidof myapp

# gRPC probe (Kubernetes 1.24+)
livenessProbe:
  grpc:
    port: 9090
    service: liveness  # Optional
```

## Rollout and Rollback Considerations

### **Deployment Strategy**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max new pods during update (default: 25%)
      maxUnavailable: 0  # Zero downtime (default: 25%)
  
  # Rollback configuration
  revisionHistoryLimit: 10  # Keep last 10 revisions
  
  # Ensures pods terminate gracefully
  minReadySeconds: 10
  
  # Ensures deployment doesn't hang
  progressDeadlineSeconds: 600  # 10 minutes
  
  template:
    spec:
      # Graceful shutdown period
      terminationGracePeriodSeconds: 30
      
      containers:
      - name: app
        image: myapp:1.2.3
        
        # Graceful shutdown handler
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15  # Allow time for load balancer to update
```

### **Pod Disruption Budget**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  # Option 1: Minimum available
  minAvailable: 2
  
  # Option 2: Maximum unavailable
  # maxUnavailable: 1
  
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
```

### **High Availability with Anti-Affinity**

```yaml
spec:
  template:
    spec:
      # Spread pods across nodes
      affinity:
        podAntiAffinity:
          # Hard requirement (must enforce)
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - myapp
            topologyKey: kubernetes.io/hostname
          
          # Soft preference (best effort)
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - myapp
              topologyKey: topology.kubernetes.io/zone
```

### **Rollout Commands**

```bash
# Apply changes
kubectl apply -f deployment.yaml

# Monitor rollout status
kubectl rollout status deployment/myapp -n production --timeout=5m

# Pause rollout
kubectl rollout pause deployment/myapp -n production

# Resume rollout
kubectl rollout resume deployment/myapp -n production

# Rollback to previous version
kubectl rollout undo deployment/myapp -n production

# Rollback to specific revision
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production --to-revision=3

# Restart deployment (rolling restart)
kubectl rollout restart deployment/myapp -n production
```

## Validation Commands

### **Client-Side Validation**

```bash
# Validate syntax without server
kubectl apply --dry-run=client -f manifest.yaml

# Validate all manifests in directory
kubectl apply --dry-run=client -f manifests/
```

### **Server-Side Validation**

```bash
# Validate with admission controllers
kubectl apply --dry-run=server -f manifest.yaml

# Check what will be created/updated
kubectl diff -f manifest.yaml
```

### **Kubeconform Validation**

```bash
# Install kubeconform
# macOS: brew install kubeconform
# Linux: Download from GitHub releases

# Validate single file
kubeconform -strict manifest.yaml

# Validate directory
kubeconform -strict -summary manifests/

# Validate with specific Kubernetes version
kubeconform -kubernetes-version 1.28.0 -strict manifest.yaml

# Output as JSON
kubeconform -output json -summary manifests/ | jq
```

### **Kubectl Dry-Run**

```bash
# Test creation without applying
kubectl create deployment myapp --image=myapp:1.2.3 --dry-run=client -o yaml

# Generate manifest from imperative command
kubectl create service clusterip myapp --tcp=8080:8080 --dry-run=client -o yaml > service.yaml
```

### **Helm Template Validation**

For Helm charts:

```bash
# Render templates
helm template myapp ./chart --values values.yaml

# Render and validate
helm template myapp ./chart --values values.yaml | kubeconform -strict

# Dry-run install
helm install myapp ./chart --values values.yaml --dry-run --debug

# Test chart
helm lint ./chart
helm template myapp ./chart --values values.yaml --validate
```

### **Policy Validation**

```bash
# OPA Conftest
conftest test manifest.yaml -p policy/

# Kyverno CLI
kyverno apply policy.yaml --resource manifest.yaml

# Datree
datree test manifest.yaml
```

## Complete Example

### **Production-Ready Deployment**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: platform
    app.kubernetes.io/managed-by: argocd
    environment: production
    team: platform
  annotations:
    description: "Main backend API service"
    owner: "platform-team@company.com"
spec:
  replicas: 3
  revisionHistoryLimit: 10
  minReadySeconds: 10
  progressDeadlineSeconds: 600
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/instance: myapp-prod
  
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/instance: myapp-prod
        app.kubernetes.io/version: "1.2.3"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    
    spec:
      # Security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      # High availability
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - myapp
              topologyKey: kubernetes.io/hostname
      
      terminationGracePeriodSeconds: 30
      
      containers:
      - name: app
        image: myapp:1.2.3
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        
        env:
        - name: PORT
          value: "8080"
        - name: LOG_LEVEL
          value: "info"
        
        envFrom:
        - configMapRef:
            name: myapp-config
        - secretRef:
            name: myapp-secrets
        
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        
        startupProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/instance: myapp-prod
```

## Manifest Checklist

- [ ] **Labels:** Standard labels applied (`app.kubernetes.io/*`)
- [ ] **Annotations:** Documentation and monitoring annotations
- [ ] **Security:** runAsNonRoot, readOnlyRootFilesystem, dropped capabilities
- [ ] **Resources:** CPU and memory requests/limits defined
- [ ] **Probes:** Liveness, readiness, and startup probes configured
- [ ] **Images:** Specific tags (never `:latest`), preferably digest-pinned
- [ ] **Replicas:** Minimum 2-3 for production
- [ ] **Strategy:** RollingUpdate with appropriate surge/unavailable
- [ ] **PDB:** Pod Disruption Budget defined
- [ ] **Anti-affinity:** Configured for multi-node distribution
- [ ] **Graceful shutdown:** terminationGracePeriodSeconds set
- [ ] **Validation:** Dry-run and kubeconform passed
- [ ] **Secrets:** Sensitive data in Secrets, not ConfigMaps or environment variables
- [ ] **NetworkPolicy:** (If applicable) Least-privilege network access

## Best Practices Summary

1. **Use standard labels and annotations** for consistency
2. **Always run as non-root** with dropped capabilities
3. **Define resource requests and limits** for all containers
4. **Implement all three probe types** for reliability
5. **Pin image tags** to specific versions or digests
6. **Configure anti-affinity** for high availability
7. **Set Pod Disruption Budgets** to prevent availability loss
8. **Use rolling updates** with zero unavailability
9. **Validate manifests** before applying to cluster
10. **Enable read-only root filesystem** when possible

## Additional Resources

- [Kubernetes Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubeconform](https://github.com/yannh/kubeconform)
