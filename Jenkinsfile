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

    stage('Desplegar en EC2') {
        def ec2IP = env.EC2_IP
        def ec2User = env.EC2_USER
        def jarName = env.JAR_NAME

        withCredentials([sshUserPrivateKey(credentialsId: 'EC2_SSH_CREDENTIAL', keyFileVariable: 'identity')]) {
            sh """
                scp -i $identity target/${jarName} ${ec2User}@${ec2IP}:/home/${ec2User}/
                ssh -i $identity ${ec2User}@${ec2IP} 'pkill -f ${jarName} || echo "No corriendo"'
                ssh -i $identity ${ec2User}@${ec2IP} 'nohup java -jar /home/${ec2User}/${jarName} > app.log 2>&1 &'
            """
        }
    }
}