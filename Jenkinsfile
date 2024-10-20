node {
    // Usa una imagen Docker que contenga Maven y OpenJDK 17
    def myMavenContainer = docker.image('maven:3.8.4-openjdk-17')
    myMavenContainer.pull()

    stage('Preparación') {
        git branch: 'main', url: 'https://github.com/mickinfo/DevopsTestMerge.git'
    }

    stage('Construcción') {
        // Usar un volumen para que el JAR se genere directamente en el host
        myMavenContainer.inside("-v ${env.WORKSPACE}/target:/usr/src/mymaven/target -v ${env.HOME}/.m2:/root/.m2") {
            sh 'mvn clean package'
        }
    }

    stage('Archivar Artefactos') {
        // Archivar el JAR generado en el host
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }

    stage('Verificar variables de entorno') {
        echo "EC2_IP: ${env.EC2_IP}"
        echo "EC2_USER: ${env.EC2_USER}"
        echo "JAR_NAME: ${env.JAR_NAME}"
    }

    stage('Verificar JAR generado') {
        // Listar el contenido del directorio target para confirmar la presencia del JAR
        sh 'ls -l target/'
    }

    stage('Verificar conexión SSH') {
        withCredentials([sshUserPrivateKey(credentialsId: 'EC2_SSH_CREDENTIAL', keyFileVariable: 'identity')]) {
            // Probar la conexión SSH antes de proceder con el despliegue
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
            def scpCommand = "scp -o StrictHostKeyChecking=no -i $identity target/${jarName} ${ec2User}@${ec2IP}:/home/${ec2User}/"
            sh scpCommand

            echo "Esperando 5 segundos para asegurar la transferencia..."
            sleep(5)

            echo "Verificando la presencia del archivo JAR en la instancia EC2..."
            def sshCheckFileCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'ls /home/${ec2User}/${jarName} || echo \"Archivo no encontrado\"'"
            sh sshCheckFileCommand

            echo "Deteniendo la instancia previa de la aplicación (si existe)..."
            def sshStopCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'pkill -f ${jarName} || echo \"No corriendo\"'"
            sh sshStopCommand

            echo "Iniciando la aplicación en segundo plano..."
            def sshStartCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'nohup java -jar /home/${ec2User}/${jarName} > /home/${ec2User}/app.log 2>&1 &'"
            sh sshStartCommand

            echo "Esperando 5 segundos para que la aplicación arranque..."
            sleep(5)

            echo "Verificando el estado de la aplicación..."
            def sshCheckAppCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'pgrep -f ${jarName} && echo \"Aplicación en ejecución\" || echo \"Aplicación no en ejecución\"'"
            sh sshCheckAppCommand

            echo "Mostrando las últimas líneas del log de la aplicación..."
            def sshShowLogsCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'tail -n 20 /home/${ec2User}/app.log || echo \"No hay logs disponibles\"'"
            sh sshShowLogsCommand
        }
    }
}
