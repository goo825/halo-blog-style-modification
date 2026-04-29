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
CLEAN_FILE=/tmp/halo-jenkins-layout-clean.html
NEW_FILE=/tmp/halo-jenkins-layout-new.html

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

skipping=0
: > "$CLEAN_FILE"
while IFS= read -r line || [ -n "$line" ]; do
  case "$line" in
    *"$STYLE_MARKER_START"*)
      skipping=1
      continue
      ;;
    *"$STYLE_MARKER_END"*)
      skipping=0
      continue
      ;;
  esac

  if [ "$skipping" -eq 0 ]; then
    printf '%s\n' "$line" >> "$CLEAN_FILE"
  fi
done < "$TARGET_FILE"

inserted=0
: > "$NEW_FILE"
while IFS= read -r line || [ -n "$line" ]; do
  if [ "$inserted" -eq 0 ] && printf '%s\n' "$line" | grep -qi '</head>'; then
    cat "$BLOCK_FILE" >> "$NEW_FILE"
    inserted=1
  fi
  printf '%s\n' "$line" >> "$NEW_FILE"
done < "$CLEAN_FILE"

if [ "$inserted" -ne 1 ]; then
  echo "Template no longer contains </head>: $TARGET_FILE"
  exit 1
fi

cat "$NEW_FILE" > "$TARGET_FILE"
rm -f "$BLOCK_FILE" "$CLEAN_FILE" "$NEW_FILE"
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
