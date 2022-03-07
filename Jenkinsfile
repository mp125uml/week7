properties([pipelineTriggers([githubPush()])])

pipeline {
     agent any
     stages {
	  stage("Compile") {
               steps {
                    sh "./gradlew compileJava"
               }
          }
          stage("Unit test") {
	       when { anyOf { branch 'master'; branch 'feature' } }
               steps {
                    sh "./gradlew test"
               }
          }
          stage("Code coverage") {
	       when { branch 'master' }
	       steps {
                    sh "./gradlew jacocoTestReport"
                    sh "./gradlew jacocoTestCoverageVerification"
               }
          }
          stage("Static code analysis") {
	      when { anyOf { branch 'master'; branch 'feature' } }
              steps {
                    sh "./gradlew checkstyleMain"
              }
          }
	  stage("Checkstyle added") {
	      when { anyOf { branch 'master'; branch 'feature' } }
	      steps {
		   sh "./gradlew checkstyleTest"
	      }
          }
          stage("Package") {
               steps {
                    sh "./gradlew build"
               }
          }
     }
}

podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: gradle
        image: gradle:6.3-jdk14
        command:
        - sleep
        args:
        - 99d
        volumeMounts:
        - name: shared-storage
          mountPath: /mnt        
      - name: kaniko
        image: gcr.io/kaniko-project/executor:debug
        command:
        - sleep
        args:
        - 9999999
        volumeMounts:
        - name: shared-storage
          mountPath: /mnt
        - name: kaniko-secret
          mountPath: /kaniko/.docker
      restartPolicy: Never
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: jenkins-pv-claim
      - name: kaniko-secret
        secret:
            secretName: dockercred
            items:
            - key: .dockerconfigjson
              path: config.json
''') {
  node(POD_LABEL) {
    stage('Build a gradle project') {
      try {
      git 'https://github.com/mp125uml/week7.git'
      container('gradle') {
        stage('Build a gradle project') {
          sh '''
          cd /home/jenkins/agent/workspace/week7_$BRANCH_NAME/
          sed -i '4 a /** Main app */' /home/jenkins/agent/workspace/week7_$BRANCH_NAME/src/main/java/com/leszko/calculator/Calculator.java
          chmod +x gradlew
          ./gradlew build
          mv ./build/libs/calculator-0.0.1-SNAPSHOT.jar /mnt
          '''
        }
       }
      } catch (exc) {
        currentBuild.result = 'FAILURE'
      }
    }

    stage('Build Java Image') {
      if ( env.BRANCH_NAME == "playground" ) {
	return
      }
   
      container('kaniko') {
        stage('Build a container') {
          sh '''
          if [ ${env.BRANCH_NAME} -eq "feature" ]
          then
	     container_name="calculator-feature"
	     version="0.1"
          fi
          echo "${container_name}:${version}"
          echo 'FROM openjdk:8-jre' > Dockerfile
          echo 'COPY ./calculator-0.0.1-SNAPSHOT.jar app.jar' >> Dockerfile
          echo 'ENTRYPOINT ["java", "-jar", "app.jar"]' >> Dockerfile
          ls /mnt/*jar
          mv /mnt/calculator-0.0.1-SNAPSHOT.jar .
          /kaniko/executor --context `pwd` --destination mattp262/$container_name:$version
          '''
        }
      }
    }

  }
}
