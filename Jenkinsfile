@Library('ciinabox') _
pipeline {
  environment {
    AWS_REGION = 'ap-southeast-2'
    ASSUME_ROLE = 'bearse/feature/iam/ciinabox-v2'
    CFN_ROLE = "arn:aws:iam::${DEV_ACCOUNT_ID}:role/${CFN_ROLE_NAME}"
    PROJECT_NAME = 'cloudfront-demo'
    CF_PREFIX = "cloudformation/${env.PROJECT_NAME}"
    CF_TEMPLATE_URL = "https://${SOURCE_BUCKET}.s3.amazonaws.com/${CF_PREFIX}"
    ORIGIN = 'www.base2services.com'
  }
  agent {
    label 'docker'
  }
  stages {
    stage('CF Compile') {
      agent {
        docker {
            image 'theonestack/cfhighlander'
            reuseNode true
        }
      }
      steps {
        sh 'cfndsl -u'
        sh "cfcompile"
      }
    }
    stage('Validate') {
      agent {
        docker {
            image 'theonestack/cfhighlander'
            reuseNode true
        }
      }
      steps {
        sh "cfcompile --validate"
      }
    }
    stage('Publish') {
      agent {
        docker {
            image 'theonestack/cfhighlander'
            reuseNode true
        }
      }
      steps {
        script {
          println "env:${env.GIT_COMMIT}"
          env['cf_version'] = env.BRANCH_NAME
          env['project_name'] = env.PROJECT_NAME
          env['BRANCH'] = env.BRANCH_NAME
          env['SHORT_COMMIT'] = env.GIT_COMMIT.substring(0,7)
          if(env.BRANCH_NAME == 'master') {
            env['cf_version'] = "master-${env.SHORT_COMMIT}"
          }
        }
        sh "cfpublish ${env.PROJECT_NAME} --version ${env.CF_VERSION} --dstbucket ${env.SOURCE_BUCKET} --dstprefix ${env.CF_PREFIX}"
      }
    }
    stage('Deploy Dev Stack') {
      environment {
        STACK_NAME = "dev-${env.PROJECT_NAME}"
      }
      steps {
        createChangeSet(
          description: env.GIT_COMMIT,
          region: env.AWS_REGION,
          stackName: env.STACK_NAME,
          templateUrl: "${env.CF_TEMPLATE_URL}/${env.cf_version}/${env.PROJECT_NAME}.compiled.yaml",
          parameters: [
            'cloudfrontapiOriginDomainName': env.ORIGIN,
          ],
          capabilities: false,
          failOnEmptyChangeSet: false,
          awsAccountId: env.DEV_ACCOUNT_ID,
          role: env.ASSUME_ROLE,
          roleArn: env.CFN_ROLE
        )
        executeChangeSet(
          region: env.AWS_REGION,
          stackName: env.STACK_NAME,
          awsAccountId: env.DEV_ACCOUNT_ID,
          role: env.ASSUME_ROLE,
          serviceRole: env.CFN_ROLE
        )
      }
    }
  }
}