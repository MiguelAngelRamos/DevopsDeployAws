node {
    def myMavenContainer = docker.image('maven:3.8.4-openjdk-17')
    myMavenContainer.pull()

    stage('Preparación') {
        git branch: 'main', url: 'https://github.com/mickinfo/DevopsTestMerge.git'
    }

    stage('Construcción') {
        myMavenContainer.inside("-v ${env.HOME}/.m2:/root/.m2") {
            sh 'mvn clean package'
        }
    }

    stage('Test') {
        myMavenContainer.inside("-v ${env.HOME}/.m2:/root/.m2") {
            sh 'mvn test'
        }
    }


    stage('Archivar Artefactos') {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }

    stage('Verificar variables de entorno') {
        echo "EC2_IP: ${env.EC2_IP}"
        echo "EC2_USER: ${env.EC2_USER}"
        echo "JAR_NAME: ${env.JAR_NAME}"
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
            def sshStartCommand = "ssh -o StrictHostKeyChecking=no -i $identity ${ec2User}@${ec2IP} 'nohup java -jar /home/${ec2User}/${jarName} > app.log 2>&1 &'"

            // Ejecutar comandos
            sh scpCommand
            sh sshStopCommand
            sh sshStartCommand
        }
    }

}