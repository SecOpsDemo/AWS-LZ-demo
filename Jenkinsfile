def SERVICE_GROUP = "demo"
def SERVICE_NAME = "aws-landingzone"
def IMAGE_NAME = "${SERVICE_NAME}"
def REPOSITORY_URL = "https://github.com/SecOpsDemo/AWS-LZ-demo.git"
def REPOSITORY_SECRET = ""
def SLACK_TOKEN_DEV = ""
def SLACK_TOKEN_DQA = ""

@Library("github.com/opsnow-tools/valve-butler")
def butler = new com.opsnow.valve.v7.Butler()
def label = "worker-${UUID.randomUUID().toString()}"

def bucket = 'aws-landing-zone-configuration-043629069679-ap-northeast-2'

properties([
  buildDiscarder(logRotator(daysToKeepStr: "60", numToKeepStr: "30"))
])
podTemplate(label: label, containers: [
  containerTemplate(name: "builder", image: "opsnowtools/valve-builder:v0.2.38", command: "cat", ttyEnabled: true, alwaysPullImage: true),
  containerTemplate(name: "sam", image: "pahud/aws-sam-cli:0.53.0", command: "cat", ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: "/var/run/docker.sock", hostPath: "/var/run/docker.sock"),
  hostPathVolume(mountPath: "/home/jenkins/.draft", hostPath: "/home/jenkins/.draft"),
  hostPathVolume(mountPath: "/home/jenkins/.helm", hostPath: "/home/jenkins/.helm")
]) {
  node(label) {
    stage("Prepare") {
      container("builder") {
        butler.prepare(IMAGE_NAME)
      }
    }
    stage("Checkout") {
      container("builder") {
        try {
          if (REPOSITORY_SECRET) {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME, credentialsId: REPOSITORY_SECRET)
          } else {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME)
          }
        } catch (e) {
          butler.failure(SLACK_TOKEN_DEV, "Checkout")
          throw e
        }
      }
    }
    // stage("Tests") {
    //   container("maven") {
    //     try {
    //       butler.mvn_test()
    //     } catch (e) {
    //       butler.failure(SLACK_TOKEN_DEV, "Tests")
    //       throw e
    //     }
    //   }
    // }

    stage("Build") {
      container("builder") {
        try {
          sh "zip -r aws-landing-zone-configuration.zip ./manifest.yaml ./parameters ./policies ./template_constraints ./templates"
          butler.success(SLACK_TOKEN_DEV, "Build")
        } catch (e) {
          butler.failure(SLACK_TOKEN_DEV, "Build")
          throw e
        }
      }
    }

    stage("Approval") {
      container("builder") {
        butler.proceed(SLACK_TOKEN_DQA, "Push to S3 Bucket", "")
        timeout(time: 60, unit: "MINUTES") {
          input(message: "Push file to S3 bucket")
        }
      }
    }

    if (BRANCH_NAME == "master") {
      stage('Push to S3 Bucket') {
        container("builder") {
          try {
            withCredentials(
                [string(credentialsId: 'aws-lz-demo-secret-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                string(credentialsId: 'aws-lz-demo-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')]
                ) {
                echo "The access_key is ${env.AWS_ACCESS_KEY_ID}"
                sh "aws s3 cp aws-landing-zone-configuration.zip s3://${bucket}/ --sse=aws:kms"
            }

            butler.success(SLACK_TOKEN_DEV, "Push")
          } catch (e) {
            butler.failure(SLACK_TOKEN_DEV, "Push")
            throw e
          }
        }
      }
    }
  }
}