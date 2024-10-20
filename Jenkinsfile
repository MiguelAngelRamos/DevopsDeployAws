node {
    def myMavenContainer = docker.image('maven:3.8.4-openjdk-17')
    myMavenContainer.pull()

    stage('Preparación') {
        git branch: 'main', url: 'https://github.com/mickinfo/DevopsTestMerge.git'
    }

    stage('Construcción') {
        myMavenContainer.inside("-v ${env.WORKSPACE}/target:/usr/src/mymaven/target -v ${env.HOME}/.m2:/root/.m2") {
            sh 'mvn clean package'
        }
    }

    stage('Archivar Artefactos') {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }

    stage('Verificar conexión SSH') {
        withCredentials([sshUserPrivateKey(credentialsId: 'EC2_SSH_CREDENTIAL', keyFileVariable: 'identity')]) {
            def sshTestCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${env.EC2_USER}@${env.EC2_IP} 'echo Conexión exitosa'"
            echo "Probando conexión SSH a ${env.EC2_IP}..."
            sh sshTestCommand
        }
    }

    stage('Desplegar en EC2') {
        def ec2IP = env.EC2_IP
        def ec2User = env.EC2_USER
        def jarName = env.JAR_NAME

        withCredentials([sshUserPrivateKey(credentialsId: 'EC2_SSH_CREDENTIAL', keyFileVariable: 'identity')]) {
            echo "Transfiriendo JAR a la instancia EC2..."
            sh "scp -o StrictHostKeyChecking=no -i $identity target/${jarName} ${ec2User}@${ec2IP}:/home/${ec2User}/"

            echo "Verificando la presencia del archivo JAR en la instancia EC2..."
            sh "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'ls /home/${ec2User}/${jarName} || echo \"Archivo no encontrado\"'"

            echo "Deteniendo la instancia previa de la aplicación (si existe)..."
            sh(script: """
                ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} \
                'pgrep -f ${jarName} && pkill -f ${jarName} || echo "No corriendo"'
            """, returnStatus: true)

            echo "Iniciando la aplicación en segundo plano..."
            sh """
                ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} \
                'nohup java -jar /home/${ec2User}/${jarName} > /home/${ec2User}/app.log 2>&1 &'
            """

            echo "Esperando que la aplicación inicie..."
            def maxRetries = 10
            def interval = 2 // segundos entre verificaciones
            def retries = 0
            def isAppRunning = false

            // Espera activa para verificar si la aplicación está corriendo
            while (retries < maxRetries) {
                def result = sh(script: "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'pgrep -f ${jarName}'", returnStatus: true)
                if (result == 0) {
                    isAppRunning = true
                    echo "La aplicación está en ejecución."
                    break
                } else {
                    echo "Esperando que la aplicación inicie... Intento ${retries + 1} de ${maxRetries}."
                    retries++
                    // Pausa corta entre verificaciones
                    try {
                        sh "sleep ${interval}"
                    } catch (Exception e) {
                        echo "Error al esperar: ${e}"
                    }
                }
            }

            if (!isAppRunning) {
                error("La aplicación no se pudo iniciar después de ${maxRetries} intentos.")
            }

            echo "Mostrando las últimas líneas del log de la aplicación..."
            sh "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'tail -n 20 /home/${ec2User}/app.log || echo \"No hay logs disponibles\"'"
        }
    }
}
