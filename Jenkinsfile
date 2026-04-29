pipeline {
    agent any

    environment {
        HALO_NAMESPACE = 'blog-system'
        HALO_LABEL = 'app=halo'
        HALO_THEME_DIR = '/root/.halo2/themes/halo-k8s-theme'
        HALO_ASSETS_DIR = '/root/.halo2/themes/halo-k8s-theme/templates/assets'
    }

    stages {
        stage('Find Halo Pod') {
            steps {
                script {
                    env.HALO_POD = sh(
                        script: "kubectl -n ${HALO_NAMESPACE} get pod -l ${HALO_LABEL} -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    if (!env.HALO_POD) {
                        error('Halo pod was not found. Check namespace and labels.')
                    }
                }
            }
        }

        stage('Restore Theme Template') {
            steps {
                sh '''
                set -eu

                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- /bin/sh -c '
                  if [ -f /root/.halo2/themes/halo-k8s-theme/templates/layout.html.jenkins-backup ]; then
                    echo "Restoring layout.html from Jenkins backup."
                    cp /root/.halo2/themes/halo-k8s-theme/templates/layout.html.jenkins-backup /root/.halo2/themes/halo-k8s-theme/templates/layout.html
                  else
                    echo "No Jenkins layout backup found, keeping current layout.html."
                  fi
                '
                '''
            }
        }

        stage('Sync Theme Assets') {
            steps {
                sh '''
                set -eu

                if [ ! -d assets ]; then
                  echo "No assets directory found, skipping asset sync."
                  exit 0
                fi

                echo "Syncing assets to ${HALO_POD}:${HALO_ASSETS_DIR}"
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- mkdir -p ${HALO_ASSETS_DIR}
                tar -cf - -C assets . | kubectl exec -i -n ${HALO_NAMESPACE} ${HALO_POD} -- tar -xf - -C ${HALO_ASSETS_DIR}
                '''
            }
        }

        stage('Show Published Files') {
            steps {
                sh '''
                set -eu
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- ls -l ${HALO_ASSETS_DIR}/css
                '''
            }
        }

        stage('Restart Halo') {
            steps {
                sh '''
                set -eu
                kubectl -n ${HALO_NAMESPACE} rollout restart deployment/halo
                kubectl -n ${HALO_NAMESPACE} rollout status deployment/halo
                '''
            }
        }
    }
}
