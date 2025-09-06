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

// Реация на пуш
pipeline {
    // ВАЖНО: Запускаем ВСЕ стадии внутри одного агента с Docker
    agent {
        docker {
            // Используем образ, в котором есть и Maven, и Docker CLI
            // *** ПРОВЕРЯЕМ ОБРАЗ ***
            image 'maven:3.9-eclipse-temurin-17' // Или ваш кастомный образ
            args '-v /var/run/docker.sock:/var/run/docker.sock -v /usr/bin/docker:/usr/bin/docker' // Пробрасываем сокет и бинарник Docker с хоста
        }
    }

    environment {
    // *** ПРОВЕРЯЕМ ДАННЫЕ ***
        DOCKER_REGISTRY_CREDENTIALS = credentials('docker-hub-credentials')
        IMAGE_TAG = "your-dockerhub-username/our-app:${env.BUILD_ID}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh 'mvn clean test' // Всё выполняется внутри контейнера 'maven:3.9...'
                // Для Go:
                // sh 'go build -o bin/app ./...'
                // sh 'go test ./...'
            }
        }

        stage('Integration Test') {
            steps {
                // Запускаем БД в отдельном контейнере. Обратите внимание: теперь мы используем docker-compose, который тоже доступен внутри нашего агента.
                sh 'docker-compose -f docker-compose.test.yml up -d'
                // Ждём, пока PostgreSQL будет готова принимать подключения
                script {
                    waitForPostgres() // Вынесли логику ожидания в отдельный метод
                }
                // Запускаем интеграционные тесты (Maven-профиль)
                sh 'mvn verify -P integration-test'
            }
            post {
                always {
                    // Гарантированно останавливаем контейнеры с БД, даже если тесты упали
                    sh 'docker-compose -f docker-compose.test.yml down'
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    // Теперь мы просто используем команду docker build, которая работает благодаря проброшенному сокету
                    docker.build("${IMAGE_TAG}")
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', env.DOCKER_REGISTRY_CREDENTIALS) {
                        docker.image("${IMAGE_TAG}").push()
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                // Деплой. Здесь мы используем ssh, который тоже должен быть установлен в агенте.
                // Для простоты можно использовать образ с ssh (например, alpine/ubuntu + ssh client).
                // Или вынести деплой на отдельный шаг, который запускается на другом агенте.
                sh """
                    ssh -o StrictHostKeyChecking=no user@test-server.com '
                        docker pull ${IMAGE_TAG} && \
                        docker stop my-app || true && \
                        docker rm my-app || true && \
                        docker run -d --name my-app -p 80:8080 ${IMAGE_TAG}
                    '
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Ура! Мы в production! ( almost :) )'
        }
        failure {
            echo 'Вкусняшки помогут всё починить!'
        }
    }
}

// Вспомогательная функция для ожидания готовности PostgreSQL
def waitForPostgres() {
    def timeout = 60 // seconds
    def counter = 0
    while (counter < timeout) {
        try {
            // Пытаемся выполнить команду внутри контейнера с Postgres
            sh 'docker exec ${JOB_BASE_NAME}-postgres-test-1 pg_isready -U user'
            echo "PostgreSQL is ready!"
            return
        } catch (Exception e) {
            sleep(1)
            counter++
            echo "Waiting for PostgreSQL to be ready... (${counter}/${timeout})"
        }
    }
    error("PostgreSQL failed to start within ${timeout} seconds.")
}
