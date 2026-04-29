pipeline {
    agent any

    environment {
        HALO_NAMESPACE = 'blog-system'
        HALO_LABEL = 'app=halo'
        HALO_WORK_DIR = '/root/.halo2'
        HALO_ASSETS_DIR = '/root/.halo2/assets'
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

echo "Patching theme template: \${TARGET_FILE}"
cp \${TARGET_FILE} \${TARGET_FILE}.jenkins-backup

python3 - \${TARGET_FILE} \${STYLE_FILE} <<'PY'
import re
import sys
from pathlib import Path

target = Path(sys.argv[1])
style_file = Path(sys.argv[2])
start = '${STYLE_MARKER_START}'
end = '${STYLE_MARKER_END}'
css = style_file.read_text(encoding='utf-8')
html = target.read_text(encoding='utf-8')
block = f"{start}\n<style>\n{css}\n</style>\n{end}"
html = re.sub(rf"\s*{re.escape(start)}.*?{re.escape(end)}", "", html, flags=re.S)
if re.search(r"</head>", html, flags=re.I):
    html = re.sub(r"</head>", block + "\n</head>", html, count=1, flags=re.I)
else:
    raise SystemExit('Template no longer contains </head>.')
target.write_text(html, encoding='utf-8')
PY

rm -f \${STYLE_FILE}
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
