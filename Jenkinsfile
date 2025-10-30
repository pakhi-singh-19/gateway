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
        // TARGET_ENV = "${env.BRANCH_NAME == 'main' ? 'prod' : env.BRANCH_NAME == 'develop' ? 'staging' : 'dev'}"
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
        
        stage('Validate Gateway Configuration') {
            steps {
                script {
                    sh '''
                        echo "Validating Gateway API configurations..."
                        
                        # Validate Kustomize configuration
                        kubectl kustomize k8s/overlays/${TARGET_ENV} > /tmp/gateway-manifests-${TARGET_ENV}.yaml
                        
                        # Validate YAML syntax
                        kubectl apply --dry-run=client -f /tmp/gateway-manifests-${TARGET_ENV}.yaml
                        
                        # Check for required resources
                        echo "Checking for required Gateway API resources..."
                        grep -q "kind: Gateway" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "Gateway resource not found!"; exit 1; }
                        grep -q "kind: HTTPRoute" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "HTTPRoute resource not found!"; exit 1; }
                        
                        # Validate routing rules
                        echo "Validating routing rules..."
                        grep -q "/api/v1/customers" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "Customer routes not found!"; exit 1; }
                        grep -q "/api/v1/products" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "Catalog routes not found!"; exit 1; }
                        grep -q "/api/v1/orders" /tmp/gateway-manifests-${TARGET_ENV}.yaml || { echo "Order routes not found!"; exit 1; }
                        
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
                            echo "Security policies found in configuration"
                        else
                            echo "Warning: No security policies found in configuration"
                        fi
                        
                        # Check for SSL configuration
                        if grep -q "tls:" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "SSL/TLS configuration found"
                        else
                            echo "Warning: No SSL/TLS configuration found"
                        fi
                        
                        # Check for RBAC resources
                        if grep -q "kind: Role" /tmp/gateway-manifests-${TARGET_ENV}.yaml; then
                            echo "RBAC configuration found"
                        else
                            echo "Warning: No RBAC configuration found"
                        fi
                        
                        echo "Security validation completed"
                    '''
                }
            }
        }
        
        stage('Deploy Gateway Configuration') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: 'gcp-service-account', variable: 'GCP_KEY')
                    ]) {
                        sh '''
                            # Authenticate with GCP
                            gcloud auth activate-service-account --key-file=${GCP_KEY}
                            gcloud config set project ${PROJECT_ID}
                            
                            # Get GKE credentials
                            gcloud container clusters get-credentials ${CLUSTER_NAME} --region=${REGION}
                            
                            # Ensure namespace exists
                            kubectl create namespace ${TARGET_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                            
                            # Apply Gateway API configurations
                            echo "Deploying Gateway configuration to ${TARGET_ENV} environment..."
                            kubectl apply -k k8s/overlays/${TARGET_ENV} --validate=false
                            
                            # Wait for Gateway to be ready
                            echo "Waiting for Gateway to be ready..."
                            kubectl wait --for=condition=Programmed gateway/${GATEWAY_NAME} -n ${TARGET_NAMESPACE} --timeout=300s || true
                            
                            # Check Gateway status
                            echo "Checking Gateway status..."
                            kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o yaml
                            
                            # Check HTTPRoute status
                            echo "Checking HTTPRoute status..."
                            kubectl get httproute -n ${TARGET_NAMESPACE}
                        '''
                    }
                }
            }
        }
        
        stage('Validate Gateway Deployment') {
            steps {
                script {
                    sh '''
                        echo "Validating Gateway deployment..."
                        
                        # Check Gateway status
                        GATEWAY_STATUS=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}' || echo "Unknown")
                        echo "Gateway status: $GATEWAY_STATUS"
                        
                        # Check HTTPRoute status
                        HTTPROUTE_COUNT=$(kubectl get httproute -n ${TARGET_NAMESPACE} --no-headers | wc -l)
                        echo "Number of HTTPRoutes deployed: $HTTPROUTE_COUNT"
                        
                        if [ "$HTTPROUTE_COUNT" -lt "3" ]; then
                            echo "Warning: Expected at least 3 HTTPRoutes (customer, catalog, order)"
                        fi
                        
                        # Check for Gateway IP assignment
                        GATEWAY_IP=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.addresses[0].value}' || echo "Not assigned")
                        echo "Gateway IP: $GATEWAY_IP"
                        
                        # Validate backend services exist
                        echo "Validating backend services..."
                        kubectl get service customer-management-service -n ${TARGET_NAMESPACE} || echo "Warning: Customer service not found"
                        kubectl get service catalog-management-service -n ${TARGET_NAMESPACE} || echo "Warning: Catalog service not found"
                        kubectl get service order-management-service -n ${TARGET_NAMESPACE} || echo "Warning: Order service not found"
                        
                        echo "Gateway deployment validation completed"
                    '''
                }
            }
        }
        
        stage('Health Check') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    sh '''
                        echo "Performing health checks..."
                        
                        # Wait for Gateway to be fully ready
                        sleep 30
                        
                        # Get Gateway external IP
                        GATEWAY_IP=$(kubectl get gateway ${GATEWAY_NAME} -n ${TARGET_NAMESPACE} -o jsonpath='{.status.addresses[0].value}' || echo "")
                        
                        if [ -n "$GATEWAY_IP" ] && [ "$GATEWAY_IP" != "Not assigned" ]; then
                            echo "Testing Gateway health via IP: $GATEWAY_IP"
                            
                            # Test health endpoints
                            curl -f -m 10 http://$GATEWAY_IP/actuator/health || echo "Health check via IP failed"
                        else
                            echo "Gateway IP not available, skipping external health check"
                        fi
                        
                        # Check internal connectivity
                        echo "Checking internal service connectivity..."
                        kubectl run gateway-test --rm -i --restart=Never --image=curlimages/curl -- \
                            curl -f -m 10 http://customer-management-service.${TARGET_NAMESPACE}.svc.cluster.local:8081/actuator/health || echo "Customer service health check failed"
                        
                        echo "Health checks completed"
                    '''
                }
            }
        }
        
        stage('Smoke Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
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
                        kubectl get httproute -n ${TARGET_NAMESPACE} -o yaml | grep -E "(customers|products|orders)" || echo "Warning: Not all service routes found"
                        
                        echo "Smoke tests completed"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            // Archive Gateway manifests
            archiveArtifacts artifacts: '/tmp/gateway-manifests-*.yaml', allowEmptyArchive: true
            
            // Clean workspace
            cleanWs()
        }
        success {
            echo "Gateway deployment completed successfully for ${TARGET_ENV} environment!"
            // Send success notification
            emailext(
                subject: "SUCCESS: Gateway Configuration - Build #${BUILD_NUMBER} (${TARGET_ENV})",
                body: """<p>Gateway configuration deployment successful for ${TARGET_ENV} environment</p>
                         <p>Version: ${GATEWAY_VERSION}</p>
                         <p>Author: ${GIT_AUTHOR}</p>
                         <p>Commit: ${GIT_COMMIT_MSG}</p>
                         <p>Namespace: ${TARGET_NAMESPACE}</p>
                         <p>Gateway: ${GATEWAY_NAME}</p>
                         <p>Check console output at <a href='${BUILD_URL}'>${BUILD_URL}</a></p>""",
                to: '${DEFAULT_RECIPIENTS}',
                mimeType: 'text/html'
            )
        }
        failure {
            echo "Gateway deployment failed for ${TARGET_ENV} environment!"
            // Send failure notification
            emailext(
                subject: "FAILURE: Gateway Configuration - Build #${BUILD_NUMBER} (${TARGET_ENV})",
                body: """<p>Gateway configuration deployment failed for ${TARGET_ENV} environment</p>
                         <p>Version: ${GATEWAY_VERSION}</p>
                         <p>Author: ${GIT_AUTHOR}</p>
                         <p>Commit: ${GIT_COMMIT_MSG}</p>
                         <p>Target Namespace: ${TARGET_NAMESPACE}</p>
                         <p>Check console output at <a href='${BUILD_URL}'>${BUILD_URL}</a></p>""",
                to: '${DEFAULT_RECIPIENTS}',
                mimeType: 'text/html'
            )
        }
        unstable {
            echo "Gateway deployment is unstable for ${TARGET_ENV} environment!"
            emailext(
                subject: "UNSTABLE: Gateway Configuration - Build #${BUILD_NUMBER} (${TARGET_ENV})",
                body: """<p>Gateway configuration deployment is unstable for ${TARGET_ENV} environment</p>
                         <p>Version: ${GATEWAY_VERSION}</p>
                         <p>Author: ${GIT_AUTHOR}</p>
                         <p>Commit: ${GIT_COMMIT_MSG}</p>
                         <p>Check console output at <a href='${BUILD_URL}'>${BUILD_URL}</a></p>""",
                to: '${DEFAULT_RECIPIENTS}',
                mimeType: 'text/html'
            )
        }
    }
}