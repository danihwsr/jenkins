pipeline {

  agent any

  environment {
    DEFAULT_TEST_PARAMS = 'teppich fensterfolie bettwaren gardine kissen lattenrost sichtschutz matratze'
    // hier gewünschte Strukturgruppe(n) eintragen
    CUSTOM_TEST_PARAMS = 'NULL'
    // hier gewünschten Testnamen eintragen (geht nur, wenn EINE Strukturgruppe angegeben wird)
    CUSTOM_TEST_CASE = 'NULL'
    QA_RESULT = 'PENDING'
    TEST_RESULT = 'PENDING'
    PROD_RESULT = 'PENDING'
    REDEPLOY = 'false'
    RUN_USER_REGRESSION_TESTS = 'true'
  }

  stages {

    // deploy master on HEIMAT-QA
    stage('deploy-qa') {

      agent {
        label 'master'
      }

      environment {
        DEPLOY_SCRIPT = '/opt/appdir/heimat-jenkins/deploy.sh'
        VAGRANT_BOX = '/opt/appdir/deployment/vagrantbox'
      }

      steps {
        script {

          try {
            echo 'deploying master on qa'

            echo 'deleting xml-reports'
            sh 'rm -f ${WORKSPACE}/*.xml'
            sh 'rm -f /opt/appdir/Application/heimatapi/testapp/test-reports/*.xml'

            if ( REDEPLOY == 'true' ) {
              echo 'get latest version of vagrantbox'
              sh 'cd ${VAGRANT_BOX} && git pull'
              echo 'redeploying HEIMAT...'
              //sh './application.setup.sh -staging=qa'
              sh 'cd ${VAGRANT_BOX} && ./application.setup.sh -staging=qa -show_version_only'
            } else {
              echo 'using existing HEIMAT deployment...'
            }

            QA_RESULT = 'SUCCESS'
          } catch (Exception err) {
            echo err
            QA_RESULT = 'FAILURE'
          }

        }
      }

    }

    // run regression tests on HEIMAT-QA
    stage('test') {

      agent {
        label 'master'
      }

      environment {
        STAGE = 'qaci'
        ENV_CONFIG_DIR = '/opt/appdir/Application/heimat-user/properties'
        PYTHON_PATH = '/opt/appdir/Application/heimatapi'
        REPORT_DIR = '/opt/appdir/Application/heimatapi/testapp/test-reports/*.xml'
      }

      when {
        beforeAgent true
        // if qa deployment was successful, execute tests
        expression {
          return QA_RESULT == 'SUCCESS' && RUN_USER_REGRESSION_TESTS == 'true';
        }
      }

      steps {

        dir ( PYTHON_PATH ) {

          script {
            try {

                if (CUSTOM_TEST_PARAMS != 'NULL'){
                  if (CUSTOM_TEST_CASE != 'NULL'){
                    echo "testing ${CUSTOM_TEST_PARAMS}"
                    sh """export STAGE=${STAGE}
                    export ENVCONFIGDIR=${ENV_CONFIG_DIR}
                    export PYTHONPATH=${PYTHON_PATH}
                    ./run_usertests.sh --strukturgruppe ${CUSTOM_TEST_PARAMS} --test ${CUSTOM_TEST_CASE}"""
                  } else {
                    def strukturgruppen = CUSTOM_TEST_PARAMS.split(' ')
                    echo "${strukturgruppen}"
                    strukturgruppen.each { strukturgruppe ->
                      echo "testing ${strukturgruppe}"
                      sh """export STAGE=${STAGE}
                      export ENVCONFIGDIR=${ENV_CONFIG_DIR}
                      export PYTHONPATH=${PYTHON_PATH}
                      ./run_usertests.sh --strukturgruppe ${strukturgruppe}"""
                    }
                  }
                } else {
                  def strukturgruppen = DEFAULT_TEST_PARAMS.split(' ')
                  strukturgruppen.each { strukturgruppe ->
                    echo "testing ${strukturgruppe}"
                    sh """export STAGE=${STAGE}
                    export ENVCONFIGDIR=${ENV_CONFIG_DIR}
                    export PYTHONPATH=${PYTHON_PATH}
                    ./run_usertests.sh --strukturgruppe ${strukturgruppe}"""
                  }
                }
                TEST_RESULT = 'SUCCESS'
              } catch (Exception err) {
                echo err
                TEST_RESULT = 'FAILURE'
              }
          }

        }

      }

      post {
        always {
          echo 'publishing test reports '
          sh '''cp -v /opt/appdir/Application/heimatapi/testapp/test-reports/* ${WORKSPACE}/
          cp -r /opt/appdir/Application/heimat-user/testdata ${WORKSPACE}/'''
          junit '*.xml'
        }
      }

    }

    // deploy on HEIMAT-PROD
    stage('deploy-prod') {

      agent {
        label 'heimat-prod'
      }

      environment {
        DEPLOYMENT_DIR = '/opt/appdir/Application/deployment/heimat-user/vagrantbox'
      }

      when {
        beforeAgent true
        // if test was successful, deploy master on HEIMAT-PROD
        expression {
          return TEST_RESULT == 'SUCCESS'
        }
      }

      steps {

        script {

          try {
              echo 'deploying heimat-user master on heimat-prod'
              sh '''cd ${DEPLOYMENT_DIR} && ./application.setup.sh -staging=live -show_version_only'''
              sh '''cd ${DEPLOYMENT_DIR} && ./application.setup.sh -staging=live'''
              sh '''cd ${DEPLOYMENT_DIR} && ./application.setup.sh -staging=live -show_version_only'''
              sh '''sudo /usr/sbin/service apache2 restart'''
              PROD_RESULT = 'SUCCESS'
            } catch (Exception err) {
              echo err
              PROD_RESULT = 'FAILURE'
            }

        }

      }

    }

  }

  post {
    always {

      script {
        if ( TEST_RESULT != 'SUCCESS' || QA_RESULT != 'SUCCESS' || PROD_RESULT != 'SUCCESS' ) {
          echo 'Es wurden nicht alle Stages der Pipeline erfolgreich durchgeführt (s. Log)!'
          currentBuild.result = 'FAILURE'
        }
      }

    }
  }

}
