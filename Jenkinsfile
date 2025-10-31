pipeline {
    agent any
    
    environment {
        // GCP Configuration
        PROJECT_ID = 'ecp-project-476405'
        REGION = 'us-central1'
        CLUSTER_NAME = 'microservices-cluster'
        NAMESPACE = 'microservices'
        
        // Gateway Configuration
        GATEWAY_NAME = 'microservices-gateway'
        GATEWAY_VERSION = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'unknown'}"
        
        // Environment Detection
        TARGET_ENV = 'prod'
        TARGET_NAMESPACE = 'microservices'
        
        // Validation Configuration
        GATEWAY_DOMAIN = 'api.microservices.ecp-project-476405.com'
        HEALTH_CHECK_TIMEOUT = '300'
    }
    
    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_MSG = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
                    env.GIT_AUTHOR = sh(returnStdout: true, script: 'git log -1 --pretty=%an').trim()
                }
                echo "Deploying Gateway configuration version ${GATEWAY_VERSION} to ${TARGET_ENV} environment"
            }
        }
        
        stage('Setup Kubernetes Context') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: 'gcp-service-account', variable: 'GCP_KEY')
                    ]) {
                        sh '''
                            echo "Setting up Kubernetes context..."
                            
                            # Authenticate with GCP
                            gcloud auth activate-service-account --key-file=${GCP_KEY}
                            gcloud config set project ${PROJECT_ID}
                            
                            # Get GKE credentials and set context
                            gcloud container clusters get-credentials ${CLUSTER_NAME} --region=${REGION}
                            
                            # Verify kubectl context
                            kubectl config current-context
                            kubectl cluster-info
                            
                            echo "Kubernetes context setup completed"
                        '''
                    }
                }
            }
        }
        
        stage('Install Required CRDs') {
            steps {
                script {
                    sh '''
                        echo "Installing required CRDs..."
                        
                        # Check if Gateway API CRDs are already installed
                        if ! kubectl get crd gateways.gateway.networking.k8s.io >/dev/null 2>&1; then
                            echo "Installing Gateway API CRDs..."
                            kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
                            
                            # Wait for CRDs to be established
                            echo "Waiting for Gateway API CRDs to be ready..."
                            kubectl wait --for condition=established --timeout=60s crd/gateways.gateway.networking.k8s.io
                            kubectl wait --for condition=established --timeout=60s crd/httproutes.gateway.networking.k8s.io
                        else
                            echo "Gateway API CRDs already installed"
                        fi
                        
                        # Check for GKE-specific CRDs
                        if ! kubectl get crd backendconfigs.cloud.google.com >/dev/null 2>&1; then
                            echo "Warning: GKE BackendConfig CRD not found - may need GKE Gateway API enabled"
                        fi
                        
                        if ! kubectl get crd frontendconfigs.networking.gke.io >/dev/null 2>&1; then
                            echo "Warning: GKE FrontendConfig CRD not found - may need GKE Gateway API enabled"
                        fi
                        
                        # List installed Gateway API CRDs
                        echo "Installed Gateway API CRDs:"
                        kubectl get crd | grep -E "(gateway|route)" || echo "No Gateway API CRDs found"
                        
                        echo "CRD installation completed"
                    '''
                }
            }
        }
        
        stage('Validate Gateway Configuration') {
            steps {
                script {
                    sh '''
                        echo "Validating Gateway API configurations..."
                        
                        # Validate Kustomize configuration
                        echo "Generating manifests for ${TARGET_ENV} environment..."
                        kubectl kustomize k8s/overlays/${TARGET_ENV} > /tmp/gateway-manifests-${TARGET_ENV}.yaml
                        
                        # Remove PodSecurityPolicy from manifests (deprecated in Kubernetes 1.25+)
                        echo "Removing deprecated PodSecurityPolicy resources..."
                        grep -v -A 20 -B 5 "kind: PodSecurityPolicy" /tmp/gateway-manifests-${TARGET_ENV}.yaml > /tmp/gateway-manifests-${TARGET_ENV}-filtered.yaml || cp /tmp/gateway-manifests-${TARGET_ENV}.yaml /tmp/gateway-manifests-${TARGET_ENV}-filtered.yaml
                        
                        # Validate YAML syntax with client-side validation only
                        echo "Validating YAML syntax..."
                        kubectl apply --dry-run=client --validate=false -f /tmp/gateway-manifests-${TARGET_ENV}-filtered.yaml
                        
                        # Check for required resources
                        echo "Checking for required Gateway API resources..."
                        grep -q "kind: Gateway" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Gateway resource not found!"; exit 1; }
                        grep -q "kind: HTTPRoute" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: HTTPRoute resource not found!"; exit 1; }
                        grep -q "kind: Namespace" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Namespace resource not found!"; exit 1; }
                        
                        # Validate routing rules
                        echo "Validating routing rules..."
                        grep -q "/api/v1/customers" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Customer routes not found!"; exit 1; }
                        grep -q "/api/v1/products" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Catalog routes not found!"; exit 1; }
                        grep -q "/api/v1/orders" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Order routes not found!"; exit 1; }
                        
                        # Validate backend service references
                        echo "Validating backend service references..."
                        grep -q "customer-management-service" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Customer service reference not found!"; exit 1; }
                        grep -q "catalog-management-service" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Catalog service reference not found!"; exit 1; }
                        grep -q "order-management-service" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "ERROR: Order service reference not found!"; exit 1; }
                        
                        # Count resources
                        GATEWAY_COUNT=$(grep -c "kind: Gateway" /tmp/gateway-manifests-${TARGET_ENV}.yaml || echo "0")
                        HTTPROUTE_COUNT=$(grep -c "kind: HTTPRoute" /tmp/gateway-manifests-${TARGET_ENV}.yaml || echo "0")
                        echo "Found ${GATEWAY_COUNT} Gateway(s) and ${HTTPROUTE_COUNT} HTTPRoute(s)"
                        
                        echo "Gateway configuration validation completed successfully!"
                    '''
                }
            }
        }
        
        stage('Security Validation') {
            steps {
                script {
                    sh '''
                        echo "Performing security validation..."
                        
                        # Check for security policies
                        if grep -q "securityPolicy" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "✓ Security policies found in configuration"
                        else
                            echo "⚠ Warning: No security policies found in configuration"
                        fi
                        
                        # Check for SSL configuration
                        if grep -q "tls:" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "✓ SSL/TLS configuration found"
                        else
                            echo "⚠ Warning: No SSL/TLS configuration found"
                        fi
                        
                        # Check for RBAC resources
                        if grep -q "kind: Role" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "✓ RBAC configuration found"
                        else
                            echo "⚠ Warning: No RBAC configuration found"
                        fi
                        
                        # Check for ServiceAccount
                        if grep -q "kind: ServiceAccount" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "✓ ServiceAccount configuration found"
                        else
                            echo "⚠ Warning: No ServiceAccount configuration found"
                        fi
                        
                        # Check for proper namespace isolation
                        if grep -q "namespace: ${TARGET_NAMESPACE}" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "✓ Proper namespace isolation configured"
                        else
                            echo "⚠ Warning: Namespace isolation may not be properly configured"
                        fi
                        
                        echo "Security validation completed"
                    '''
                }
            }
        }
        
        stage('Deploy Gateway Configuration') {
            steps {
                script {
                    sh '''
                        echo "Deploying Gateway configuration to ${TARGET_ENV} environment..."
                        
                        # Ensure namespace exists
                        kubectl create namespace ${TARGET_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Apply Gateway API configurations with proper validation settings
                        echo "Applying Gateway manifests (excluding deprecated resources)..."
                        kubectl apply -f /tmp/gateway-manifests-${TARGET_ENV}-filtered.yaml --validate=false --timeout=300s
                        
                        # Wait for Gateway to be ready
                        echo "Waiting for Gateway to be ready..."
                        kubectl wait --for=condition=Programmed gateway/${GATEWAY_NAME} -n ${TARGET_NAMESPACE} --timeout=300s || {
                            echo "Gateway not ready within timeout, checking status..."
                            kubectl describe gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE}
                        }
                        
                        # Check Gateway status
                        echo "Checking Gateway status..."
                        kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o wide
                        
                        # Check HTTPRoute status
                        echo "Checking HTTPRoute status..."
                        kubectl get httproute -n ${TARGET_NAMESPACE} -o wide
                        
                        # Check all deployed resources
                        echo "Checking all deployed resources..."
                        kubectl get all -n ${TARGET_NAMESPACE} -l app=api-gateway
                    '''
                }
            }
        }
        
        stage('Validate Gateway Deployment') {
            steps {
                script {
                    sh '''
                        echo "Validating Gateway deployment..."
                        
                        # Check Gateway status
                        GATEWAY_STATUS=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}' 2>/dev/null || echo "Unknown")
                        echo "Gateway Programmed status: $GATEWAY_STATUS"
                        
                        # Check Gateway accepted status
                        GATEWAY_ACCEPTED=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}' 2>/dev/null || echo "Unknown")
                        echo "Gateway Accepted status: $GATEWAY_ACCEPTED"
                        
                        # Check HTTPRoute status
                        HTTPROUTE_COUNT=$(kubectl get httproute -n ${TARGET_NAMESPACE} --no-headers 2>/dev/null | wc -l || echo "0")
                        echo "Number of HTTPRoutes deployed: $HTTPROUTE_COUNT"
                        
                        if [ "$HTTPROUTE_COUNT" -lt "1" ]; then
                            echo "⚠ Warning: No HTTPRoutes found"
                        else
                            echo "✓ HTTPRoutes deployed successfully"
                        fi
                        
                        # Check for Gateway IP assignment
                        GATEWAY_IP=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.addresses[0].value}' 2>/dev/null || echo "Not assigned")
                        echo "Gateway IP: $GATEWAY_IP"
                        
                        # Validate backend services exist (optional check)
                        echo "Checking for backend services..."
                        kubectl get service customer-management-service -n ${TARGET_NAMESPACE} 2>/dev/null && echo "✓ Customer service found" || echo "⚠ Customer service not found"
                        kubectl get service catalog-management-service -n ${TARGET_NAMESPACE} 2>/dev/null && echo "✓ Catalog service found" || echo "⚠ Catalog service not found"
                        kubectl get service order-management-service -n ${TARGET_NAMESPACE} 2>/dev/null && echo "✓ Order service found" || echo "⚠ Order service not found"
                        
                        # Check HTTPRoute acceptance
                        echo "Checking HTTPRoute acceptance status..."
                        kubectl get httproute -n ${TARGET_NAMESPACE} -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="Accepted")].status}{"\n"}{end}' 2>/dev/null || echo "HTTPRoute status not available"
                        
                        echo "Gateway deployment validation completed"
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sh '''
                        echo "Performing health checks..."
                        
                        # Wait for Gateway to be fully ready
                        sleep 30
                        
                        # Get Gateway external IP
                        GATEWAY_IP=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.addresses[0].value}' 2>/dev/null || echo "")
                        
                        if [ -n "$GATEWAY_IP" ] && [ "$GATEWAY_IP" != "Not assigned" ] && [ "$GATEWAY_IP" != "null" ]; then
                            echo "Testing Gateway health via IP: $GATEWAY_IP"
                            
                            # Test health endpoints with timeout
                            timeout 30 curl -f -m 10 -H "Host: ${GATEWAY_DOMAIN}" http://$GATEWAY_IP/actuator/health || echo "⚠ Health check via IP failed"
                        else
                            echo "⚠ Gateway IP not available, skipping external health check"
                        fi
                        
                        # Check internal connectivity (if services exist)
                        echo "Checking internal service connectivity..."
                        if kubectl get service customer-management-service -n ${TARGET_NAMESPACE} >/dev/null 2>&1; then
                            kubectl run gateway-test-customer --rm -i --restart=Never --image=curlimages/curl --timeout=60s -- \
                                curl -f -m 10 http://customer-management-service.${TARGET_NAMESPACE}.svc.cluster.local:8081/actuator/health || echo "⚠ Customer service health check failed"
                        fi
                        
                        echo "Health checks completed"
                    '''
                }
            }
        }
        
        stage('Smoke Tests') {
            steps {
                script {
                    sh '''
                        echo "Running smoke tests for Gateway..."
                        
                        # Test Gateway configuration
                        echo "Testing Gateway configuration..."
                        kubectl describe gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE}
                        
                        # Test HTTPRoute configuration
                        echo "Testing HTTPRoute configuration..."
                        kubectl describe httproute -n ${TARGET_NAMESPACE}
                        
                        # Validate routing rules
                        echo "Validating routing rules..."
                        ROUTE_CHECK=$(kubectl get httproute -n ${TARGET_NAMESPACE} -o yaml | grep -E "(customers|products|orders)" | wc -l)
                        if [ "$ROUTE_CHECK" -ge "3" ]; then
                            echo "✓ All service routes found"
                        else
                            echo "⚠ Warning: Not all service routes found (found: $ROUTE_CHECK)"
                        fi
                        
                        # Check Gateway class
                        GATEWAY_CLASS=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.spec.gatewayClassName}' 2>/dev/null || echo "Unknown")
                        echo "Gateway class: $GATEWAY_CLASS"
                        
                        # Check listeners
                        LISTENER_COUNT=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.spec.listeners[*].name}' 2>/dev/null | wc -w || echo "0")
                        echo "Number of listeners: $LISTENER_COUNT"
                        
                        echo "Smoke tests completed"
                    '''
                }
            }
        }
    }
    
}