
@Library("CI_LIB@lib_master") _

def AGENT_LABELS = lib_Main.getAgentLabels(JOB_NAME)
def IMAGE_TAG = lib_Main.getDockerImage(JOB_NAME)
def CI_STATE = new HashMap()

pipeline {

  parameters {
       booleanParam(name: 'RUN_DOWNSTREAM', description: 'if false skip downstream jobs', defaultValue: true)
       booleanParam(name: 'RUN_TESTS', description: 'if false skip testing', defaultValue: true)
       booleanParam(name: 'RUN_BUILD', description: 'if false skip building', defaultValue: true)
       string(name: 'jsonstr_CI_STATE', description: 'Default State if no upstream job',
              defaultValue: '''{"BSD":{"WAITING":"false", "GIT_BRANCH":"master", "GIT_URL":"https://projecttools.nordicsemi.no/bitbucket/scm/iot/bsd_socket_interface.git"},
                              "BLE":{"WAITING":"false", "GIT_BRANCH":"master", "GIT_URL":"https://projecttools.nordicsemi.no/bitbucket/scm/drgn/dragoon.git"}}''')
                           // "CRYPTO":{"WAITING":"false", "BRANCH_NAME":"master","GIT_BRANCH":" ","GIT_URL":"https://projecttools.nordicsemi.no/bitbucket/scm/drgn/dragoon.git"},
                           // "NFC":{"WAITING":"false", "BRANCH_NAME":"master","GIT_BRANCH":" ","GIT_URL":"https://github.com/thst-nordic/nrfxlib.git"},
  }

  agent {
    docker {
      image IMAGE_TAG
      label AGENT_LABELS
      args '-e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/workdir/.local/bin'
    }
  }

  options {
    checkoutToSubdirectory('nrfxlib')
  }

  triggers {
    cron(env.BRANCH_NAME == 'master' ? '0 */4 * * 1-7' : '') // Only master will be build periodically
  }

  environment {
      // ENVs for check-compliance
      GH_TOKEN = credentials('nordicbuilder-compliance-token') // This token is used to by check_compliance to comment on PRs and use checks
      GH_USERNAME = "NordicBuilder"
      COMPLIANCE_ARGS = "-r NordicPlayground/fw-nrfconnect-nrf"

      // Build all custom samples that match the ci_build tag
      SANITYCHECK_OPTIONS = "--board-root $WORKSPACE/nrf/boards --testcase-root $WORKSPACE/nrf/samples --testcase-root $WORKSPACE/nrf/applications --build-only --disable-unrecognized-section-test -t ci_build --inline-logs"
      ARCH = "-a arm"
      LC_ALL = "C.UTF-8"

      // ENVs for building (triggered by sanitycheck)
      ZEPHYR_TOOLCHAIN_VARIANT = 'gnuarmemb'
      GNUARMEMB_TOOLCHAIN_PATH = '/workdir/gcc-arm-none-eabi-7-2018-q2-update'
  }

  stages {

    stage('Load') {
      steps {
        script {
          lib_State.preLoad()
          CI_STATE = lib_State.load(params['jsonstr_CI_STATE'])
          lib_State.store('NRFXLIB', CI_STATE)
          lib_State.getParentJob(CI_STATE)
          lib_State.pushjobStack('NRFXLIB', CI_STATE)
          println "CI_STATE = $CI_STATE"
        }
      }
    }
    stage('Checkout repositories') {
      when { expression { CI_STATE.NRFXLIB.RUN_TESTS } }
      steps {
        script {
          lib_Main.cloneCItools(JOB_NAME)
          CI_STATE.NRFXLIB.REPORT_SHA = lib_Main.checkoutRepo(CI_STATE.NRFXLIB.GIT_URL, "nrfxlib", CI_STATE, false)
          // lib_Main.westInitUpdate('nrf')
        }
      }
    }
    
    // stage('Apply Manifest Updates') {
    //   when { expression { CI_STATE.NRF.RUN_TESTS.toBoolean() } }
    //   steps {
    //     script {
    //       println "If triggered by an upstream Project, use their changes."
    //       lib_Main.updateManifest(CI_STATE)
    //     }
    //   }
    // }
    
    stage('Run compliance check') {
      when { expression { CI_STATE.NRFXLIB.RUN_TESTS } }
      steps {
        dir('nrf') {
          script {
            // If we're a pull request, compare the target branch against the current HEAD (the PR), and also report issues to the PR
            def BUILD_TYPE = lib_Main.getBuildType(CI_STATE.NRFXLIB)
            if (BUILD_TYPE == "PR") {
              COMMIT_RANGE = "$CI_STATE.NRFXLIB.MERGE_BASE..$CI_STATE.NRFXLIB.REPORT_SHA"
              COMPLIANCE_ARGS = "$COMPLIANCE_ARGS -p $CHANGE_ID -S $CI_STATE.NRFXLIB.REPORT_SHA -g"
              println "Building a PR [$CHANGE_ID]: $COMMIT_RANGE"
            }
            else if (BUILD_TYPE == "TAG") {
              COMMIT_RANGE = "tags/${env.BRANCH_NAME}..tags/${env.BRANCH_NAME}"
              println "Building a Tag: " + COMMIT_RANGE
            }
            // If not a PR, it's a non-PR-branch or master build. Compare against the origin.
            else if (BUILD_TYPE == "BRANCH") {
              COMMIT_RANGE = "origin/${env.BRANCH_NAME}..HEAD"
              println "Building a Branch: " + COMMIT_RANGE
            }
            else {
                assert condition : "Build fails because it is not a PR/Tag/Branch"
            }

            // Run the compliance check
            try {
              sh "(source ../zephyr/zephyr-env.sh && ../ci-tools/scripts/check_compliance.py $COMPLIANCE_ARGS --commits $COMMIT_RANGE)"
            }
            finally {
              junit 'compliance.xml'
              archiveArtifacts artifacts: 'compliance.xml'
            }
          }
        }
      }
    }
    stage('Build samples') {
      when { expression { CI_STATE.NRFXLIB.RUN_BUILD } }
      steps {
          echo "No Samples to build yet."
      }
    }
    stage('Trigger testing build') {
      when { expression { CI_STATE.NRFXLIB.RUN_DOWNSTREAM } }
      steps {
        script {
          CI_STATE.NRFXLIB.WAITING = true
          def DOWNSTREAM_JOBS = lib_Main.getDownStreamJobs(CI_STATE, 'NRFXLIB')
          println "DOWNSTREAM_JOBS = " + DOWNSTREAM_JOBS
          def jobs = [:]
          DOWNSTREAM_JOBS.each {
            jobs["${it}"] = {
              build job: "${it}", propagate: true, wait: true, parameters: [
                        string(name: 'jsonstr_CI_STATE', value: lib_Util.HashMap2Str(CI_STATE))]
            }
          }
          parallel jobs
        }
      }
    }
  }
  post {
    // This is the order that the methods are run. {always->success/abort/failure/unstable->cleanup}
    always {
      echo "always"
    }
    success {
      echo "success"
    }
    aborted {
      echo "aborted"
    }
    unstable {
      echo "unstable"
    }
    failure {
      echo "failure"
    }
    cleanup {
        echo "cleanup"
        cleanWs()
    }
  }
}
