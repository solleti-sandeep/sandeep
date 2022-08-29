def distributionID = ''
def tenantAwsRegion ='us-east-1'
pipeline {
  agent {
    label 'master'
  }
  options {
    disableConcurrentBuilds()
    timeout(time: 1, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '3'))
  }
  parameters{
    choice(name:'AWS_ENV', choices:['docai'],description:'Which AWS environment do you want to deploy?')
    choice(name:'APP_ENV', choices:['neo-docai'],description:'Which AWS bucket do you want to deploy?')
    choice(name:'AWS_REGION', choices:['us-east-1'],description:'To which AWS Region you want to deploy?')
  }
  stages {
    stage('prepare'){
      steps {
        nodejs('NodeJS') {
          sh 'npm install'
        }
      }
    }
    stage('build number'){
      steps {
        updateBuildNumber()
      }
    }
    stage('compile') {
      steps {	  
        nodejs('NodeJS') {
          sh 'node --max_old_space_size=8048 node_modules/@angular/cli/bin/ng build --prod --no-progress'	          
        }	   
      }	   
    }
    
    stage('unit test'){
      when {
        expression { params.AWS_ENV == 'neo-docai' }
      }
      steps {
        nodejs('NodeJS') {
          sh 'npm rebuild node-sass'
          script {
            try{
              sh 'node --max_old_space_size=8048 ./node_modules/@angular/cli/bin/ng test --watch=false --code-coverage=true --progress=false --browsers=HeadlessChrome'
            } catch(e){
              echo 'exception in unit tests'
            }
          }
        }
      }
    }

    stage('GET TENANT INFO FROM PLATFORM'){
      steps{
        script{
            withAWS(region:"${params.AWS_REGION}",credentials:"${params.MASTER_ACCOUNT_NAME}"){
              sh 'chmod 755 cloud/aws/platform-aws-cli.sh'
              sh "./cloud/aws/platform-aws-cli.sh -t ${params.AWS_ENV}"
            }
        }
      }
    }
    stage('Set environment variable'){
      when{
        branch DEPLOY_BRANCH
      }
      steps{
        script{
        withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){
            sh 'chmod 755 cloud/aws/tenant-aws-cli.sh'
            sh './cloud/aws/tenant-aws-cli.sh'

            awsTenantDetails = readJSON file: 'cloud/aws/tenant-aws-data.json'
            echo " DATABASE_URL : ${awsTenantDetails.DATABASE_URL}"
        }
       }
      }
    }

    stage('Run Sql Script'){
      when{
        branch DEPLOY_BRANCH
      }
      steps{
        script{
          withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){
            sh "sed -i  's/%ENVIRONMENT%/${params.AWS_ENV}/g;s/%VERSION_NUMBER%/${VERSION_NO}/g;s/%AWS_REGION%/${params.AWS_REGION}/g;' db/docai-data.sql"
            sh 'psql postgresql://'+"${awsTenantDetails.DATABASE_URL}"+'/'+"${awsTenantDetails.DATABASE_NAME}"+' -f db/docai-data.sql'
          }
        }
      }
    }

    stage('deploy') {
      when{
        branch 'dev'
      }

      steps {
        script {
          try{
            echo 'aws cloudfront list-distributions --query "DistributionList.Items[*].{Domain: join(\', \', Aliases.Items), DistributionID: Id}[?contains(Domain,\''+"${params.APP_ENV}"+'.'+"${params.AWS_ENV}"+'-neo.bcone.com\')] | [0].DistributionID" | tr -d \'"\' | tr -d \'\\n\''  
            withAWS(credentials:"${params.AWS_ENV}"){
              echo 'aws cloudfront list-distributions --query "DistributionList.Items[*].{Domain: join(\', \', Aliases.Items), DistributionID: Id}[?contains(Domain,\''+"${params.APP_ENV}"+'.'+"${params.AWS_ENV}"+'-neo.bcone.com\')] | [0].DistributionID" | tr -d \'"\' | tr -d \'\\n\''
					    sh "aws s3 sync --delete --cache-control max-age=31556926 dist/. s3://${params.AWS_REGION}-neo-${params.AWS_ENV}-${params.APP_ENV}/"
			        distributionID = sh(script: 'aws cloudfront list-distributions --query "DistributionList.Items[*].{Domain: join(\', \', Aliases.Items), DistributionID: Id}[?contains(Domain,\''+"${params.APP_ENV}"+'.'+"${params.AWS_ENV}"+'-neo.bcone.com\')] | [0].DistributionID" | tr -d \'"\' | tr -d \'\\n\'', returnStdout: true)
            }
            echo distributionID
            if(distributionID!=null){
               withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${params.AWS_ENV}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                echo distributionID
                try{
                  step([$class: 'LambdaInvokeBuildStep', lambdaInvokeBuildStepVariables: [awsAccessKeyId: AWS_ACCESS_KEY_ID, awsRegion: tenantAwsRegion, awsSecretKey: AWS_SECRET_ACCESS_KEY, functionName: 'SYS_CLOUDFRONT_CREATE_INVALIDATION', payload: '{"distributionId":"'+distributionID+'"}', synchronous: true]])
                }
                catch(e)
                {
                  echo 'SYS_CLOUDFRONT_CREATE_INVALIDATION does not exist'
                }
              }
            }
          }
          catch(err){

          }
        }
      }
    }
    stage('cleanup'){  
      steps{
               sh "rm -rf node_modules"  
      }
    }
  }
}
def updateBuildNumber(){
  def packageJson = readFile 'package.json'
  packageJson = packageJson.replaceAll('0.0.0','0.1.0+'+"${env.BUILD_NUMBER}")
  writeFile(file:'package.json',text:packageJson)
}
