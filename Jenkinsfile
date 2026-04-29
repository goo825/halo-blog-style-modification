pipeline {
    agent any

    environment {
        HALO_NAMESPACE = 'blog-system'
        HALO_LABEL = 'app=halo'
        HALO_WORK_DIR = '/root/.halo2'
        HALO_ASSETS_DIR = '/root/.halo2/themes/halo-k8s-theme/templates/assets'
        STYLE_MARKER_START = '<!-- halo-jenkins-style:start -->'
        STYLE_MARKER_END = '<!-- halo-jenkins-style:end -->'
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

        stage('Sync Assets') {
            steps {
                sh '''
                set -eu

                if [ ! -d assets ]; then
                  echo "No assets directory found, skipping asset sync."
                  exit 0
                fi

                echo "Syncing assets to ${HALO_POD}:${HALO_ASSETS_DIR}"
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- mkdir -p ${HALO_ASSETS_DIR}
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- rm -rf ${HALO_ASSETS_DIR}/*
                tar -cf - -C assets . | kubectl exec -i -n ${HALO_NAMESPACE} ${HALO_POD} -- tar -xf - -C ${HALO_ASSETS_DIR}
                '''
            }
        }

        stage('Patch Active Theme') {
            steps {
                sh '''
                set -eu

                if [ ! -f assets/css/custom.css ]; then
                  echo "assets/css/custom.css was not found, skipping theme patch."
                  exit 0
                fi

                STYLE_B64=$(base64 < assets/css/custom.css | tr -d '\n')

                kubectl exec -i -n ${HALO_NAMESPACE} ${HALO_POD} -- /bin/sh -s <<EOF
set -eu

STYLE_FILE=/tmp/halo-jenkins-custom.css
BLOCK_FILE=/tmp/halo-jenkins-style-block.html
printf '%s' '${STYLE_B64}' | base64 -d > \${STYLE_FILE}

THEMES_DIR='${HALO_WORK_DIR}/themes'
if [ ! -d \${THEMES_DIR} ]; then
  echo "Theme directory was not found: \${THEMES_DIR}"
  exit 1
fi

TARGET_FILE=""
for file in \
  \$(find \${THEMES_DIR} -type f \( -name '*.html' -o -name '*.ftl' \) 2>/dev/null | sort); do
  if grep -qi '</head>' \${file}; then
    TARGET_FILE=\${file}
    break
  fi
done

if [ -z \${TARGET_FILE} ]; then
  echo "No theme template containing </head> was found."
  echo "Available theme files:"
  find \${THEMES_DIR} -maxdepth 4 -type f | sort | head -80
  exit 1
fi

cat > \${BLOCK_FILE} <<'BLOCK_EOF'
${STYLE_MARKER_START}
<style>
BLOCK_EOF
cat \${STYLE_FILE} >> \${BLOCK_FILE}
cat >> \${BLOCK_FILE} <<'BLOCK_EOF'
</style>
${STYLE_MARKER_END}
BLOCK_EOF

echo "Patching theme template: \${TARGET_FILE}"
cp \${TARGET_FILE} \${TARGET_FILE}.jenkins-backup

awk -v start='${STYLE_MARKER_START}' -v end='${STYLE_MARKER_END}' '
  index(\$0, start) { skipping = 1; next }
  index(\$0, end) { skipping = 0; next }
  !skipping { print }
' \${TARGET_FILE} > \${TARGET_FILE}.clean

awk -v block_file=\${BLOCK_FILE} '
  BEGIN {
    while ((getline line < block_file) > 0) {
      block = block line "\n"
    }
    close(block_file)
  }
  BEGIN { inserted = 0 }
  tolower(\$0) ~ /<\/head>/ && inserted == 0 {
    printf "%s", block
    inserted = 1
  }
  { print }
  END {
    if (inserted == 0) {
      exit 1
    }
  }
' \${TARGET_FILE}.clean > \${TARGET_FILE}

rm -f \${STYLE_FILE} \${BLOCK_FILE} \${TARGET_FILE}.clean
EOF
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
