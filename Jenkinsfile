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
            // Imprimir el contenido de la variable 'identity' (sin revelar la clave privada)
            echo "Identity file path: ${identity}"

            // Comandos de despliegue
            def scpCommand = "scp -o StrictHostKeyChecking=no -i $identity target/${jarName} ${ec2User}@${ec2IP}:/home/${ec2User}/"
            def sshStopCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'pkill -f ${jarName} || echo \"No corriendo\"'"
            def sshCheckFileCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'ls /home/${ec2User}/${jarName} || echo \"Archivo no encontrado\"'"
            def sshStartCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'nohup java -jar /home/${ec2User}/${jarName} > app.log 2>&1 &'"

            // Ejecutar comandos
            echo "Transfiriendo JAR a la instancia EC2..."
            sh scpCommand

            echo "Verificando la presencia del archivo JAR en la instancia EC2..."
            sh sshCheckFileCommand

            echo "Deteniendo la instancia previa de la aplicación (si existe)..."
            sh sshStopCommand

            echo "Iniciando la aplicación..."
            sh sshStartCommand
        }
    }
}
