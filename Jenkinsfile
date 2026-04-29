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

                kubectl exec -i -n ${HALO_NAMESPACE} ${HALO_POD} -- /bin/sh -s -- \
                  "${HALO_WORK_DIR}" \
                  "${HALO_ASSETS_DIR}/css/custom.css" \
                  "${STYLE_MARKER_START}" \
                  "${STYLE_MARKER_END}" <<'REMOTE_SCRIPT'
set -eu

HALO_WORK_DIR=$1
STYLE_FILE=$2
STYLE_MARKER_START=$3
STYLE_MARKER_END=$4
BLOCK_FILE=/tmp/halo-jenkins-style-block.html

if [ ! -f "$STYLE_FILE" ]; then
  echo "Style file was not found in Halo pod: $STYLE_FILE"
  exit 1
fi

THEMES_DIR="$HALO_WORK_DIR/themes"
if [ ! -d "$THEMES_DIR" ]; then
  echo "Theme directory was not found: $THEMES_DIR"
  exit 1
fi

TARGET_FILE=$(find "$THEMES_DIR" -type f | sort | while IFS= read -r file; do
  case "$file" in
    *.html|*.ftl)
      if grep -qi '</head>' "$file"; then
        printf '%s\n' "$file"
        exit 0
      fi
      ;;
  esac
done | head -1)

if [ -z "$TARGET_FILE" ]; then
  echo "No theme template containing </head> was found."
  echo "Available theme files:"
  find "$THEMES_DIR" -maxdepth 4 -type f | sort | head -80
  exit 1
fi

{
  printf '%s\n' "$STYLE_MARKER_START"
  printf '<style>\n'
  cat "$STYLE_FILE"
  printf '\n</style>\n'
  printf '%s\n' "$STYLE_MARKER_END"
} > "$BLOCK_FILE"

echo "Patching theme template: $TARGET_FILE"
cp "$TARGET_FILE" "$TARGET_FILE.jenkins-backup"

awk -v start="$STYLE_MARKER_START" -v end="$STYLE_MARKER_END" '
  index($0, start) { skipping = 1; next }
  index($0, end) { skipping = 0; next }
  !skipping { print }
' "$TARGET_FILE" > "$TARGET_FILE.clean"

awk -v block_file="$BLOCK_FILE" '
  BEGIN {
    while ((getline line < block_file) > 0) {
      block = block line "\n"
    }
    close(block_file)
  }
  BEGIN { inserted = 0 }
  tolower($0) ~ /<[/]head>/ && inserted == 0 {
    printf "%s", block
    inserted = 1
  }
  { print }
  END {
    if (inserted == 0) {
      exit 1
    }
  }
' "$TARGET_FILE.clean" > "$TARGET_FILE"

rm -f "$BLOCK_FILE" "$TARGET_FILE.clean"
REMOTE_SCRIPT
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
