node {
    // Usa una imagen Docker que contenga Maven y OpenJDK 17
    def myMavenContainer = docker.image('maven:3.8.4-openjdk-17')
    myMavenContainer.pull()

    stage('Preparación') {
        // Clonar el repositorio desde GitHub
        git branch: 'main', url: 'https://github.com/mickinfo/DevopsTestMerge.git'
    }

    stage('Construcción') {
        // Usar un volumen para construir el JAR en el host
        myMavenContainer.inside("-v ${env.WORKSPACE}/target:/usr/src/mymaven/target -v ${env.HOME}/.m2:/root/.m2") {
            sh 'mvn clean package'
        }
    }

    stage('Archivar Artefactos') {
        // Archivar el JAR generado en Jenkins
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }

    stage('Verificar conexión con Nexus') {
        withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIAL', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            // Realiza una solicitud CURL a Nexus para verificar la conectividad
            def nexusUrl = "http://172.23.0.3:8081/service/rest/v1/status"
            def curlCommand = "curl -u ${NEXUS_USER}:${NEXUS_PASS} -s -o /dev/null -w '%{http_code}' ${nexusUrl}"
            def responseCode = sh(script: curlCommand, returnStdout: true).trim()

            // Verifica si el código de respuesta es 200
            if (responseCode != '200') {
                error("No se pudo establecer conexión con Nexus en ${nexusUrl}. Código de respuesta: ${responseCode}")
            } else {
                echo "Conexión con Nexus establecida exitosamente."
            }
        }
    }

    stage('Desplegar en Nexus') {
        withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIAL', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            // Determina si es un snapshot o release
            def isSnapshot = "${env.VERSION}"?.contains("SNAPSHOT") ?: "true" // Asume snapshot si no está definida la variable
            def repoUrl = isSnapshot ? "http://172.23.0.3:8081/repository/maven-snapshots/" : "http://172.23.0.3:8081/repository/maven-releases/"

            myMavenContainer.inside("-v ${env.HOME}/.m2:/root/.m2") {
                try {
                    sh """
                        mvn deploy -s /usr/share/maven/ref/settings-docker.xml \
                            -DaltDeploymentRepository=nexus::default::${repoUrl} -X
                    """
                } catch (Exception e) {
                    error("El despliegue falló. Verifica la conectividad y las credenciales: ${e.message}")
                }
            }
        }
    }
}
