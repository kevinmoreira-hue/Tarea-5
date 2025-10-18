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
            -t /work/jmeter/Tarea_semana3_KevinMoreira.jmx \
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

    stage('Archive Results') {
      steps {
        script {
          echo "=== Archiving All Performance Testing Artifacts ==="

          // Archive JMeter results and reports
          if (fileExists("${OUT_DIR}/results.jtl")) {
            archiveArtifacts artifacts: "${OUT_DIR}/**", fingerprint: true
            echo "✅ JMeter results and HTML reports archived"
            
            // Performance Plugin - Parse JTL files for trending and analysis
            try {
              perfReport(
                sourceDataFiles: "${OUT_DIR}/results.jtl",
                modeOfThreshold: true,
                configType: 'ART',
                modePerformancePerTestCase: true,
                compareBuildPrevious: true,
                modeThroughput: true,
                nthBuildNumber: 0,
                errorFailedThreshold: 5,
                errorUnstableThreshold: 10,
                relativeFailedThresholdPositive: 20,
                relativeFailedThresholdNegative: 0,
                relativeUnstableThresholdPositive: 50,
                relativeUnstableThresholdNegative: 0,
                modeEvaluation: true
              )
              echo "✅ Performance trends and analysis configured"
            } catch (Exception e) {
              echo "⚠️ Performance Plugin not available: ${e.message}"
              echo "📝 Install Performance Plugin in Jenkins: Manage Jenkins → Manage Plugins → Search 'Performance'"
            }
          }

          // Archive generated reports
          if (fileExists("${REPORTS_DIR}")) {
            archiveArtifacts artifacts: "${REPORTS_DIR}/**", fingerprint: true
            echo "✅ Performance analysis reports archived"
          }

          // Publish HTML reports
          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: "${OUT_DIR}/jmeter-report",
            reportFiles: 'index.html',
            reportName: 'JMeter Performance Report',
            reportTitles: 'JMeter HTML Dashboard'
          ])

          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: "${REPORTS_DIR}/generated",
            reportFiles: 'performance_summary.html',
            reportName: 'Performance Summary',
            reportTitles: 'Performance Test Summary'
          ])

          echo "✅ HTML reports published to Jenkins"
        }
      }
    }

    stage('Performance Analysis') {
      steps {
        script {
          if (fileExists("${OUT_DIR}/results.jtl")) {
            echo "📊 Iniciando análisis automatizado de rendimiento con umbrales dinámicos (P95 < 500 ms, tasa de error < 1%)"

            // Leer métricas base
            def results = sh(script: "tail -n +2 ${OUT_DIR}/results.jtl | wc -l", returnStdout: true).trim().toInteger()
            def errors = sh(script: "tail -n +2 ${OUT_DIR}/results.jtl | awk -F',' '\$8==\"false\"' | wc -l", returnStdout: true).trim().toInteger()
            def successRate = ((results - errors) * 100) / results

            // Calcular tiempos de respuesta
            def avgResponse = sh(script: "tail -n +2 ${OUT_DIR}/results.jtl | awk -F',' '{sum+=\$2; count++} END {if(count>0) print int(sum/count); else print 0}'", returnStdout: true).trim().toInteger()
            def maxResponse = sh(script: "tail -n +2 ${OUT_DIR}/results.jtl | awk -F',' '{if(\$2>max) max=\$2} END {print int(max)}'", returnStdout: true).trim().toInteger()

            // 🔹 Calcular percentil 95 (P95)
            def p95Response = sh(script: """
              tail -n +2 ${OUT_DIR}/results.jtl | awk -F',' '{print \$2}' | sort -n | awk '{
                arr[NR]=\$1
              }
              END {
                idx=int(NR*0.95)
                if (idx<1) idx=1
                print arr[idx]
              }'
            """, returnStdout: true).trim().toInteger()

            echo "📊 Performance Test Results:"
            echo "   Total Requests: ${results}"
            echo "   Errors: ${errors}"
            def successRateRounded = ((successRate * 10) as int) / 10.0
            echo "   Success Rate: ${successRateRounded}%"
            echo "   Average Response Time: ${avgResponse} ms"
            echo "   P95 Response Time: ${p95Response} ms"
            echo "   Max Response Time: ${maxResponse} ms"

            // 🔍 Evaluar umbrales automáticos (error < 1%, P95 < 500ms)
            def errorRate = 100 - successRate
            def performanceIssues = []

            if (errorRate > 1) {
              performanceIssues.add("⚠️ Error rate above 1% (${String.format('%.2f', errorRate)}%)")
            }

            if (p95Response > 500) {
              performanceIssues.add("⚠️ P95 response time above 500 ms (${p95Response} ms)")
            }

            if (performanceIssues.size() > 0) {
              echo "🚨 Performance Issues Detected:"
              performanceIssues.each { issue ->
                echo "   ${issue}"
              }
            }

            // 🔧 Determinar resultado del build según umbrales
            if (errorRate <= 1 && p95Response <= 500) {
              currentBuild.result = 'SUCCESS'
              echo "✅ Performance test PASSED - All automated thresholds met"
            } else if (errorRate <= 3 && p95Response <= 800) {
              currentBuild.result = 'UNSTABLE'
              echo "⚠️ Performance test UNSTABLE - Slightly above thresholds"
            } else {
              currentBuild.result = 'FAILURE'
              echo "❌ Performance test FAILED - Thresholds violated"
            }

            // 📘 Descripción del build
            currentBuild.description = "Success: ${successRateRounded}% | Error: ${String.format('%.2f', errorRate)}% | P95: ${p95Response}ms | Avg: ${avgResponse}ms | Max: ${maxResponse}ms | Req: ${results}"

            echo "📈 Umbrales aplicados: P95 < 500ms, Error rate < 1%"
            echo "📊 Revisión disponible en Performance Trend y reportes HTML"
          }
        }
      }
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
