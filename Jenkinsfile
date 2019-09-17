node('maven') {
  stage('Compile') {
    sh 'hostname'
    git 'https://github.com/pdurbin/maven-hello-world.git'
    sh '''
      ls
      cd my-app
      mvn clean compile package
      ls target
    '''
  }
}
