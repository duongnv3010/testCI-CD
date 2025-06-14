// Jenkinsfile (Declarative Pipeline)

pipeline {
    // Chỉ định agent (máy thực thi). 'any' nghĩa là bất kỳ agent nào có sẵn.
    // Trong thực tế, bạn nên dùng agent có label cụ thể (ví dụ: label 'docker')
    agent {
        kubernetes {
            // Cung cấp toàn bộ định nghĩa Pod ở đây
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: docker
                image: docker:20.10.21-git # Image có sẵn Docker CLI và Git
                command:
                - cat
                args:
                - "-"
                tty: true
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
            '''
            // defaultContainer: 'docker' không còn cần thiết khi chỉ có 1 container chính
        }
    }

    // Các biến môi trường sử dụng trong pipeline
    environment {
        // ID của credentials chứa username/password Docker Hub, đã được cấu hình trong Jenkins
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials' 
        // Tên image trên Docker Hub
        DOCKER_IMAGE_NAME = 'duong3010/myapp'
        // ID của credentials chứa token để checkout repo cấu hình K8s
        GIT_CONFIG_REPO_CREDENTIALS_ID = 'gitcd' 
        // URL của kho chứa cấu hình K8s
        GIT_CONFIG_REPO_URL = 'https://github.com/duongnv3010/CD-VDT.git'
    }

    stages {
        // Giai đoạn 1: Checkout mã nguồn từ GitLab
        stage('1. Checkout Code') {
            steps {
                script {
                    echo "Bắt đầu checkout mã nguồn..."
                    checkout scm
                    echo "Checkout thành công."
                }
            }
        }

        // Giai đoạn 2: Cài đặt các dependencies
        stage('2. Install Dependencies') {
            steps {
                script {
                    echo "Đang cài đặt các thư viện Node.js..."
                    // Dùng image node để chạy npm install, tránh cài node trực tiếp trên agent
                    docker.image('node:18-alpine').inside {
                        sh 'npm install'
                    }
                    echo "Cài đặt thành công."
                }
            }
        }

        // Giai đoạn 3: Chạy Unit Test
        stage('3. Unit Test & Coverage') {
            steps {
                script {
                    echo "Đang chạy Unit Test và tạo báo cáo độ bao phủ..."
                    docker.image('node:18-alpine').inside {
                        sh 'npm test'
                    }
                    echo "Test hoàn tất."
                }
            }
        }
        // Giai đoạn 6: Build và Push Docker Image
        stage('6. Build & Push Docker Image') {
            steps {
                script {
                    // Lấy 8 ký tự đầu của mã git commit để làm tag cho image
                    def gitCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim().substring(0, 8)
                    def dockerImageTag = "${DOCKER_IMAGE_NAME}:${gitCommit}"
                    
                    echo "Đang build Docker image: ${dockerImageTag}"
                    // Build image
                    def customImage = docker.build(dockerImageTag)

                    echo "Đang push image lên Docker Hub..."
                    // Đẩy image lên Docker Hub sử dụng credentials đã định nghĩa
                    docker.withRegistry('https://registry.hub.docker.com', DOCKERHUB_CREDENTIALS_ID) {
                        customImage.push()
                    }
                    echo "Push image thành công."
                }
            }
        }
        
        // Giai đoạn 7: Cập nhật kho chứa cấu hình Kubernetes (GitOps)
        stage('7. Update K8s Manifest Repo') {
            steps {
                script {
                    def gitCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim().substring(0, 8)
                    def dockerImageTag = "${DOCKER_IMAGE_NAME}:${gitCommit}"

                    echo "Bắt đầu cập nhật kho chứa cấu hình K8s..."
                    // Sử dụng credentials để checkout kho chứa manifest
                    withCredentials([usernamePassword(credentialsId: GIT_CONFIG_REPO_CREDENTIALS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        // Clone repo cấu hình vào một thư mục con
                        sh "git clone https://${GIT_USER}:${GIT_PASS}@github.com/duongnv3010/CD-VDT.git k8s-manifest-repo"
                        
                        // Di chuyển vào thư mục repo vừa clone
                        dir('k8s-manifest-repo') {
                            echo "Đang cập nhật image tag trong deployment.yaml thành ${dockerImageTag}"
                            // Dùng sed để tìm và thay thế image tag trong tệp deployment
                            // Đây là bước "ma thuật" của GitOps!
                            sh "sed -i 's|image: .*|image: ${dockerImageTag}|g' deployment.yaml"
                            
                            // Cấu hình git user
                            sh "git config user.email 'nguyenduong20053010@gmail.com'"
                            sh "git config user.name 'duongnv3010'"
                            
                            // Commit và push thay đổi
                            sh "git add deployment.yaml"
                            sh "git commit -m 'ci: Cập nhật image tag lên ${gitCommit}'"
                            sh "git push origin main"
                        }
                    }
                    echo "Cập nhật kho chứa cấu hình K8s thành công."
                }
            }
        }
    }
    
    // Các hành động sẽ thực hiện sau khi pipeline kết thúc (dù thành công hay thất bại)
    post {
        always {
            echo 'Pipeline đã kết thúc.'
            // Dọn dẹp workspace
            cleanWs()
        }
    }
}
