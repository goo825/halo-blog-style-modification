pipeline {
    agent any

    environment {
        HALO_NAMESPACE = 'blog-system'
        HALO_LABEL     = 'app=halo'
        // 确保这是 Halo 真正加载资源的目标目录
        HALO_ASSETS_DIR = '/root/.halo2/assets'
    }

    stages {
        stage('Find Halo Pod') {
            steps {
                script {
                    // 获取当前正在运行的 Pod 名字
                    env.HALO_POD = sh(
                        script: "kubectl -n ${HALO_NAMESPACE} get pod -l ${HALO_LABEL} --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    if (!env.HALO_POD) {
                        error('找不到正在运行的 Halo Pod，请检查 Namespace 和 Label。')
                    }
                    echo "目标 Pod: ${env.HALO_POD}"
                }
            }
        }

        stage('Sync Theme Assets') {
            steps {
                sh '''
                set -eu

                # 检查本地 Jenkins 工作区是否有 assets 目录
                if [ ! -d assets ]; then
                  echo "错误：当前仓库根目录未找到 assets 文件夹！"
                  exit 1
                fi

                echo "正在同步资源到 ${HALO_POD}:${HALO_ASSETS_DIR} ..."

                # 1. 在容器内创建目标目录
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- mkdir -p ${HALO_ASSETS_DIR}

                # 2. 清理目标目录旧文件（可选，如果你想完全同步）
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- rm -rf ${HALO_ASSETS_DIR}/*

                # 3. 核心改进：-C assets . 表示进入 assets 目录进行打包，这样解压后不会多出一层 assets 壳子
                tar -cf - -C assets . | kubectl exec -i -n ${HALO_NAMESPACE} ${HALO_POD} -- tar -xf - -C ${HALO_ASSETS_DIR}

                # 4. 权限修复：确保容器内用户有权访问新传的文件（Halo 镜像通常使用 1000 用户）
                kubectl exec -n ${HALO_NAMESPACE} ${HALO_POD} -- chown -R 1000:1000 ${HALO_ASSETS_DIR}

                echo "同步完成！"
                '''
            }
        }

        // 注意：已移除 Restart 阶段。
        // 因为在没有 PVC 持久化的情况下，重启会导致你刚才同步的文件被还原（丢失）。
    }

    post {
        success {
            echo "部署成功！请刷新博客页面（建议 Ctrl + F5 强制刷新）查看效果。"
        }
    }
}
