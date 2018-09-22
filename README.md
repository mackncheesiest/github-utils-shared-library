# Github Utils Shared Library

A library to be used in conjunction with [Jenkins Pipelines](https://jenkins.io/doc/book/pipeline/shared-libraries/) that adds a few helper functions for use in a Jenkins Pipeline-based CI flow.

Basic usage requires:
- Linking to this repository in your Pipeline/Multibranch Pipeline job configuration
- Including it with something along the lines of `library github-utils-shared-library@master` in your Jenkinsfile
- Using any of the files defined in the `vars` folder directly as functions in your multibranch pipeline

For example, suppose your pipeline job is used to build pull requests. 

(i) If you wanted to post a comment to that pull request whenever the job succeeds or fails, your basic pipeline structure can be along the lines of:
```
library github-utils-shared-library@master

pipeline {
  agent { ... }
  stages {
    ...
  }
  post {
    success {
      postCommentIfPR("Your build succeeded! For more information, see the job results [here](${BUILD_URL}).", "myusername", "myrepository", "${GITHUB_AUTH_TOKEN}")
    }
    failure {
      postCommentIfPR("Your build failed! For more information, see the job results [here](${BUILD_URL}).", "myusername", "myrepository", "${GITHUB_AUTH_TOKEN}")
    }
  }
}
```
(ii) If you want to have your pipeline dynamically set [Github Statuses](https://developer.github.com/v3/repos/statuses/) on a given PR based on the results of intermediate stage executions, you can use something along the lines of:
```
library github-utils-shared-library@master

pipeline {
  agent { ... }
  stages {
    stage('Setup') {
      steps {
        script {
          def jsonBlob = getGithubStatusJsonBlob("pending", "${BUILD_URL}", "Build in progress...", "Jenkins CI/Build")
          postStatusToHash("${jsonBlob}", "myusername", "myrepository", "${COMMIT_HASH}", "${GITHUB_AUTH_TOKEN}")
        }
      }
    }
    stage('Build') {
      steps {
        ...
      }
      post {
        success {
          script {
            def jsonBlob = getGithubStatusJsonBlob("success", "${BUILD_URL}", "Build succeeded!", "Jenkins CI/Build")
            postStatusToHash("${jsonBlob}", "myusername", "myrepository", "${COMMIT_HASH}", "${GITHUB_AUTH_TOKEN}")
          }
        }
        failure {
          script {
            def jsonBlob = getGithubStatusJsonBlob("failure", "${BUILD_URL}", "Build failed!", "Jenkins CI/Build")
            postStatusToHash("${jsonBlob}", "myusername", "myrepository", "${COMMIT_HASH}", "${GITHUB_AUTH_TOKEN}")
          }
        }
      }
    }
    ...
  }
}
```

## Troubleshooting 
Note that posting statuses to a pull request requires knowing the exact hash to send the status to through the Github API. Because Jenkins ends up inadvertently creating a merge commit if the source branch it's building isn't fully rebased with the target branch, some logic can be required to ensure you have the right hash on hand.

One (imperfect) approach is to identify whether or not the HEAD commit is a merge commit, and if so, use the hash at HEAD~1 (under the assumption that this merge commit was created by Jenkins). This can be done with logic along the lines of:
```
script {
  try  {
    sh(script: 'if [ `git cat-file -p HEAD | head -n 3 | grep parent | wc -l` -gt 1 ]; then exit 1; else exit 0; fi')
    //No error was thrown -> we called exit 0 -> HEAD is not a merge commit/doesn't have multiple parents
    env.PR_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
  } catch (err) {
    //An error was thrown -> we called exit 1 -> HEAD is a merge commit/has multiple parents
    env.PR_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD~1').trim()
  }
}

```
where `"${env.PR_COMMIT}"` is then used as the argument to pass into `postStatusToHash`
