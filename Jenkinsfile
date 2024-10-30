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

stage('Desplegar en Nexus') {
    withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIAL', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
        def isSnapshot = "${env.VERSION}".contains("SNAPSHOT")
        def repoUrl = isSnapshot ? "http://172.23.0.3:8081/repository/mvn-snapshots/" : "http://172.23.0.3:8081/repository/mvn-releases/"

        myMavenContainer.inside("-v ${env.HOME}/.m2:/root/.m2") {
            sh """
                mvn deploy -DaltDeploymentRepository=nexus::default::${repoUrl}
            """
        }
    }
}


}
