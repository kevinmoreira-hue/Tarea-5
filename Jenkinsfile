/*
 * Performance Testing Pipeline with Jenkins Performance Plugin Integration
 * 
 * Required Jenkins Plugins:
 * - Performance Plugin: For JMeter result analysis and trending
 * - HTML Publisher Plugin: For HTML report publishing
 * 
 * To install Performance Plugin:
 * 1. Go to Jenkins → Manage Jenkins → Manage Plugins
 * 2. Search for "Performance Plugin" in Available tab
 * 3. Install and restart Jenkins
 */

pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    // Keep builds for report history
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {

    DOCKER_NETWORK = 'jenkins_net'
    OUT_DIR = 'out'
    REPORTS_DIR = 'reports'
    JMETER_IMAGE = 'jmeter-prom:latest'
    JMETER_PROM_PORT = '9270'
    JMETER_CONTAINER_NAME = 'jmeter-run'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        echo "Starting Performance Testing Pipeline for ${env.BRANCH_NAME}"
      }
    }

    stage('Build JMeter Image') {
      steps {
        sh """
          docker build -t ${JMETER_IMAGE} ./jmeter
        """
      }
    }

    stage('Run Performance Tests') {
      steps {
        sh """
          echo "=== Starting JMeter Performance Tests ==="
          # Clean and recreate output directory
          rm -rf ${OUT_DIR}
          mkdir -p ${OUT_DIR}

          # Clean previous container if any
          docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true

          # Create a simple JMeter container and start it in background
          # Override entrypoint to use shell and keep container alive
          docker run -d \
            --name ${JMETER_CONTAINER_NAME} \
            --network=${DOCKER_NETWORK} \
            --memory=1g \
            --memory-swap=2g \
            --shm-size=256m \
            --entrypoint="" \
            ${JMETER_IMAGE} sleep 3600
          
          # Create directory structure in the running container
          docker exec ${JMETER_CONTAINER_NAME} mkdir -p /work/jmeter /work/out
          
          # Copy JMeter files into the container (avoids volume mount issues)
          docker cp jmeter/. ${JMETER_CONTAINER_NAME}:/work/jmeter/
          
          # Clean any existing results in the container's output directory
          docker exec ${JMETER_CONTAINER_NAME} rm -f /work/out/results.jtl || true
          docker exec ${JMETER_CONTAINER_NAME} rm -rf /work/out/jmeter-report || true
          
          # Verify files are copied
          echo "=== DEBUG: Container JMeter directory contents ==="
          docker exec ${JMETER_CONTAINER_NAME} ls -la /work/jmeter/ || echo "Could not list files"
          
          # Execute JMeter inside the running container
          set +e  # Don't fail immediately on error
          echo "=== Running JMeter tests with 5-minute timeout ==="
          timeout 300 docker exec ${JMETER_CONTAINER_NAME} jmeter -n \
            -t /work/jmeter/Tarea semana 3 Kevin Moreira.jmx \
            -l /work/out/results.jtl \
            -e -o /work/out/jmeter-report \
            -f \
            -Jjmeter.save.saveservice.output_format=csv \
            -Jjmeter.save.saveservice.response_data=false \
            -Jjmeter.save.saveservice.samplerData=false \
            -Jjmeter.save.saveservice.responseHeaders=false
          JMETER_EXIT_CODE=\$?
          set -e  # Re-enable immediate failure
          
          if [ \$JMETER_EXIT_CODE -eq 124 ]; then
            echo "=== JMeter test timed out after 5 minutes ==="
            docker kill ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true
          fi
          
          echo "=== JMeter container exit code: \$JMETER_EXIT_CODE ==="

          # Check container status
          CONTAINER_STATUS=\$(docker inspect ${JMETER_CONTAINER_NAME} --format='{{.State.Status}}' 2>/dev/null || echo "not-found")
          echo "=== Container status: \$CONTAINER_STATUS ==="
          
          # Try to get logs before container might be removed
          if [ "\$CONTAINER_STATUS" != "not-found" ]; then
            echo "=== JMeter container logs ==="
            docker logs ${JMETER_CONTAINER_NAME} 2>/dev/null || echo "Could not retrieve logs"
            
            # Check what was generated in the container (if still exists)
            echo "=== DEBUG: Container output directory contents ==="
            docker exec ${JMETER_CONTAINER_NAME} ls -la /work/out/ 2>/dev/null || echo "No output directory in container or container not accessible"
            
            # Copy results back from container to Jenkins workspace
            echo "=== Copying results from container to workspace ==="
            docker cp ${JMETER_CONTAINER_NAME}:/work/out/. ${OUT_DIR}/ 2>/dev/null || echo "Could not copy results from container"
            
            # Stop and remove the container
            docker stop ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true
            docker rm ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true
          else
            echo "=== Container not found - may have been auto-removed ==="
          fi
          
          # Verify results were generated
          echo "=== DEBUG: Final workspace output directory contents ==="
          ls -la ${OUT_DIR}/
          
          if [ -f "${OUT_DIR}/results.jtl" ]; then
            echo "=== JMeter Test Results Generated Successfully ==="
            echo "Total lines in results: \$(wc -l < ${OUT_DIR}/results.jtl)"
            echo "Sample results:"
            head -5 ${OUT_DIR}/results.jtl
          else
            echo "ERROR: No results.jtl file generated"
            exit 1
          fi
          
          # Exit with JMeter's exit code
          exit \$JMETER_EXIT_CODE
        """
      }
    }

 

  post {
    always {
      echo "=== Performance Testing Pipeline Complete ==="
      echo "Build: ${env.BUILD_URL}"
      echo "Reports available in Jenkins artifacts and HTML publisher"
    }

    success {
      echo "✅ Performance testing completed successfully"
    }

    unstable {
      echo "⚠️  Performance testing completed with warnings"
    }

    failure {
      echo "❌ Performance testing failed"
      // Optional: Send notifications here
    }

    cleanup {
      // Clean up Docker containers
      sh """
        docker rm -f ${JMETER_CONTAINER_NAME} >/dev/null 2>&1 || true
        echo "Cleanup completed"
      """
    }
  }
}
