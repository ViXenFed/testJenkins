pipeline {
    agent any // Запускаем на любом доступном агенте (можно указать docker agent)

    // Окружения и параметры, которые будут использоваться в stages
    environment {
        // Достаём из заранее созданных в Jenkins credentials
        DOCKER_REGISTRY_CREDENTIALS = credentials('docker-hub-credentials')
        // Устанавливаем переменную для версии образа (например, из номера билда)
        IMAGE_TAG = "our-app:${env.BUILD_ID}"
    }

    stages {
        // Стадия 1: Получение кода из репозитория
        stage('Checkout') {
            steps {
                checkout scm // Jenkins просто клонирует наш репозиторий
            }
        }

        // Стадия 2: Сборка проекта (и Java, и Go, если нужно)
        stage('Build & Unit Test') {
            steps {
                script {
                    // Сборка Java-части
                    sh 'mvn clean compile' 
                    // Запуск Unit-тестов (падают тут -> пайплайн останавливается)
                    sh 'mvn test' 

                    // Если есть Go-модули, можно добавить:
                    // sh 'cd /path/to/go/mod && go build -o bin/app'
                    // sh 'cd /path/to/go/mod && go test ./...'
                }
            }
        }

        // Стадия 3: Интеграционное тестирование с БД
        stage('Integration Test') {
            steps {
                // Запускаем БД в Docker (например, PostgreSQL) для тестов
                sh 'docker-compose -f docker-compose.test.yml up -d'
                // Ждём, пока БД станет доступна (важный кастыль для надёжности)
                sh 'while ! pg_isready -h localhost -p 5432; do sleep 1; done'
                // Запускаем интеграционные тесты (Maven-профиль)
                sh 'mvn verify -P integration-test'
                // Останавливаем контейнеры с БД
                sh 'docker-compose -f docker-compose.test.yml down'
            }
        }

        // Стадия 4: Сборка Docker-образа
        stage('Build Docker Image') {
            steps {
                script {
                    // Собираем образ, используя Dockerfile из репозитория
                    docker.build("${IMAGE_TAG}")
                }
            }
        }

        // Стадия 5: Пуш образа в Registry
        stage('Push Docker Image') {
            steps {
                script {
                    // Логинимся в Docker Registry используя credentials из Jenkins
                    docker.withRegistry('', env.DOCKER_REGISTRY_CREDENTIALS) {
                        docker.image("${IMAGE_TAG}").push()
                    }
                }
            }
        }

        // Стадия 6: Деплой (например, на тестовый сервер)
        stage('Deploy to Staging') {
            steps {
                // Здесь можно использовать Ansible, kubectl, ssh...
                // Например, простой вариант для хакатона - обновляем контейнер через ssh
                sh """

// *** ЗДЕСЬ НУЖНЫ ДАННЫЕ ТЕСТОВОГО СЕРВЕРА/ЛОКАЛХОСТА *** 
                    ssh user@our-test-server.com \
                    'docker pull ${IMAGE_TAG} && \
                    docker-compose -f /app/docker-compose.yml up -d'
                """
            }
        }
    }

    // Пост-действия (выполняются даже если пайплайн упал)
    post {
        always {
            // Всегда очищаем рабочее пространство от временных файлов
            cleanWs()
        }
        success {
            // Если всё успешно - шлём уведомление в Slack/Telegram
            echo 'Пайплайн успешно завершён! Молодцы!'
            // slackSend channel: '#hackathon-team', message: "Build ${env.BUILD_ID} успешно задеплоена!"
        }
        failure {
            // Если упало - шлём тревогу
            echo 'Что-то пошло не так. Проверяйте логи!'
            // slackSend channel: '#hackathon-team', color: 'danger', message: "ВНИМАНИЕ! Build ${env.BUILD_ID} упал!"
        }
    }
}