---
title: "Jenkins"
date: 2019-01-29T00:02:30+01:00
tags: ["ci", "jenkins"]
---

## Jenkinsfile

A `Jenkinsfile` is a text file that contains the definition of a Jenkins Pipeline and is checked into source control.

Sample `Jenkinsfile` for Angular application. Taken from [here](https://github.com/petenorth/angular-5-example)

```groovy
pipeline{
  agent { label 'nodejs8' }
  stages{
    stage ('checkout'){
      steps{
        checkout scm
      }
    }
    stage ('install modules'){
      steps{
        sh '''
          npm install --verbose -d
          npm install --save classlist.js
        '''
      }
    }
    stage ('test'){
      steps{
        sh '''
          $(npm bin)/ng test --single-run --browsers Chrome_no_sandbox
        '''
      }
      post {
          always {
            junit "test-results.xml"
          }
      }
    }
    stage ('code quality'){
      steps{
        sh '$(npm bin)/ng lint'
      }
    }
    stage ('build') {
      steps{
        sh '$(npm bin)/ng build --prod --build-optimizer'
      }
    }
    stage ('build image') {
      steps{
        sh '''
          rm -rf node_modules
          oc start-build angular-5-example --from-dir=. --follow
        '''
      }
    }
  }
}
```

### Links

- [Using a Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile/)
- [Declarative Jenkins Docker Pipeline for Angular CLI applications](https://antonyderham.me/post/jenkins-docker-pipelines/)
