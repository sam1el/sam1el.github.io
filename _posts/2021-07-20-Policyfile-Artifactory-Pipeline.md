---
layout: post
title:  "Example Jenkinsfile for publishing cookbooks to Artifactory"
date:   2021-06-22 09:35:12 -0500
categories: Demo Jenkinsfile
---


# Policyfile cookbook pipeline for Artifactory

***Created by Jeff Brimager and Collin Mcneese.. Mostly Collin***

*This is an example of how to use `jenkinsfile` with any git repo to publish your cookbooks to an enterprise artifactory install.*

*You will need the following for this jenkinsfile to work correctly.*

+ Artifactory plugin installed in Jenkins

+ Either remove or comment the `library` line. It was left in to show usage. This particular librrary will skip the `kitchen` testing when a certain flag is added to the commit message.

+ Worker nodes able to use `Chef-Workstation`

+ Credentials available for connection to vcenter as well as for kitchen to use in the kitchen.yml

***Section 1 of the jenkinsfile***

*In this section of the file we declare we are using a custom library, define the pipeline and, create the first stage of the pipeline job. This is the `checkout` stage.*

The `checkout` stage in this file completes these tasks.

1. Checks out your git repo on the defined worker node.

1. Checks the local cookbook's `metadata.rb` and verifies the version has changed form the previously published version.
   + if the version has not been bumped, it will fail the build.

1. Creates an archive of the repo to deliver to Artifactory that has been configured as a supermarket repo if all tests are passed.

```java
library 'Kitchen'

pipeline {
  agent any
  stages {
    stage('checkout') {
      steps {
        sh 'git log -n1'
        script {
          def jsonData = readJSON file: "${env.WORKSPACE}/pipeline.json"
          def version_draft = sh returnStdout: true, script: 'grep -e \'^version\' metadata.rb | awk -F \"\'\" \'{printf $2}\''
          def version_check = sh returnStdout: true, script: "if curl -f -s ${jsonData.publish_location}${jsonData.repo_name}/${version_draft}/ > /dev/null ; then echo -n 'exists' ; fi"
          echo "version_draft is $version_draft"
          echo "version_check status is $version_check"
          if ( version_check == 'exists' ) {
            throw new Exception("Checkout version ${version_draft} already exists!!  Bump to new version before submitting. ")
          }
          sh "zip -r ${jsonData.repo_name}.zip * -x *.kitchen/*"
        }

      }
```

***Section 2 of the jenkinsfile***

*In this section of the file, we instruct our worker nodes to verify the chef code as one last verification before testing with Test-Kitchen. These steps are run in parallel to save time.*

The `validate` stage currently only has 2 actions.

1. Runs `cookstyle` to make sure all syntax is correct and in line with best practices for writing cookbooks.

1. Runs `chef install` to pull down dependencies and create the `Policyfile.lock`

```java
 }
    stage('validate') {
      parallel {
        stage('Style Check') {
          environment {
            HOME = "${env.WORKSPACE}"
          }
          steps {
            sh 'cookstyle .'
          }
        }
        stage('Policyfile Lock') {
          environment {
            HOME = "${env.WORKSPACE}"
          }
          steps {
            sh 'chef install'
          }
        }
      }
    }
```

***Section 3 of the jenkinsfile***

*In this section of the file, we are running all of our `unit tests` in parallel*

The `Testing` stage completes the following tasks in parallel.

1. Sets up build environment with required local variables.

1. Renames kitchen files to use jenkins specific one for this test environment.   **_Note:_** Will add example at the end of this file.

1. Runs initial converge to create machine and deploy the code.

1. Runs initial verify against the test nodes to make sure they pass all `inspec` test.

1. Converge and Verify are then run again to ensure there has been no drift and that the code is idempotent. This could be done in less steps but, I wanted to provide a thorough example.

1. Destroys all kitchen instances whether they pass or fail for clean up purposes.

1. Runs `chef exec rspec` in can tests were written to verify the cookbook logic.

```java
    stage('Testing') {
      when { expression { Kitchen action: 'isRunKitchenTests' } }
      environment {
        HOME = "${env.HOME}"
        VCENTER_CRED = credentials('YourVcenterCredsinJenkins')
        VCENTER_USER = "$VCENTER_CRED_USR"
        VCENTER_PASSWORD = "$VCENTER_CRED_PSW"
        VCENTER_HOST = 'vcenterhost.com'
        VCENTER_VMFOLDER = 'folder vms are created in'
        VCENTER_NETWORK = 'network_in_vcenter'
        VCENTER_CLUSTER = 'your_vcenter_cluster'
        VCENTER_DC = 'your_vcenter_datacenter'
        KITCHEN_CRED = credentials('YourKitchenCredsinJenkins')
        KITCHENUSER = "$KITCHEN_CRED_USR"
        KITCHENPASS = "$KITCHEN_CRED_PSW"
      }
      parallel {
        stage('Integration Testing') {
          steps {
            sh 'mv kitchen.yml kitchen.notused'
            sh 'mv kitchen.pipeline.yml kitchen.yml'
            sh 'kitchen converge --color'
            sh 'kitchen verify --color'
            sh 'kitchen converge --color'
            sh 'kitchen verify --color'
          }
          post {
            always {
              sh 'kitchen destroy'
            }
          }
        }
        stage('Unit Testing') {
          steps {
            sh 'chef exec rspec'
          }
        }
      }
    }
```

***Section 4 of the jenkinsfile***

*In this section of the file, we will either notify github that the `feature` branch cam be merged or, publish the cookbook to Artifactory once the master branch runs through the tests.*

The `Publish` phase completes the following steps

1. If not the master branch, Jenkins will notify your git provider that the current branch has been built successfully and is ready to merge if all user defined criteria is met in github or gitlab.

1. If building the master branch, the pipeline will...
   + Provide all necessary variables to connect to your artifactory.

   + Create a staging directory.

   + Unzip the previously created zip archive to the staging directory

   + Fetch ssl certs from the Artifactory server using `knife` **_Note:_** *This was an issue we ran in to when trying to publish the cookbooks. It may not be required but, doesn't hurt anything.*

   + Pushes the cookbook to the Artifactory `Supermarket` repo.

1. Notifies another jenkins job that the cookbook has been published and to start it's build using the `build job:` command. **_Note:_** *This downstream job is currently building `habitat` packages using the same Test-Kitchen and VCenter method. I will share an example of this jenkinsfile soon.*

1. The post steps are currently only echoing message to a log. This section could be used to email a team of job completion or, that the code is ready to be merged to master. **_Note:_** *I will provide an example of this below.*

```java
    stage('Publish') {
      when {
        allOf {
          branch 'master'
        }
      }
      environment {
        ARTIF_CREDS = credentials('artifactory_id')
        ARTIF_USER = "$ARTIF_CREDS_USR"
        ARTIF_PASSWORD = "$ARTIF_CREDS_PSW"
        ARTIF_URL = "https://$ARTIF_USER:$ARTIF_PASSWORD@yourartifactoryrepo.com/artifactory/api/chef/chef"
      }
      steps {
        echo 'Publishing to Artifactory Supermarket ...'
        script {
          def jsonData = readJSON file: "${env.WORKSPACE}/pipeline.json"
          sh "mkdir artifactory_stage"
          sh "unzip ${jsonData.repo_name}.zip -d artifactory_stage/${jsonData.repo_name}"
          sh "knife ssl fetch -s https://yourartifactoryrepo.com"
          sh "knife artifactory share ${jsonData.repo_name} -m $ARTIF_URL -o artifactory_stage/ "
        }
        build job: 'downstream_job', propagate: true, wait: true
      }
      }
      }
    post {
    success {
      echo 'The build was successful please merge'
    }
    failure {
      echo 'The build failed'
    }
    always {
      cleanWs()
    }
  }
}
```

***Completed Jenkinsfile***

**This has been collapsed due to size**

<details>
jenkinsfile

```java
library 'Kitchen'

pipeline {
  agent any
  stages {
    stage('checkout') {
      steps {
        sh 'git log -n1'
        script {
          def jsonData = readJSON file: "${env.WORKSPACE}/pipeline.json"
          def version_draft = sh returnStdout: true, script: 'grep -e \'^version\' metadata.rb | awk -F \"\'\" \'{printf $2}\''
          def version_check = sh returnStdout: true, script: "if curl -f -s ${jsonData.publish_location}${jsonData.repo_name}/${version_draft}/ > /dev/null ; then echo -n 'exists' ; fi"
          echo "version_draft is $version_draft"
          echo "version_check status is $version_check"
          if ( version_check == 'exists' ) {
            throw new Exception("Checkout version ${version_draft} already exists!!  Bump to new version before submitting. ")
          }
          sh "zip -r ${jsonData.repo_name}.zip * -x *.kitchen/*"
        }

      }
    }
    stage('validate') {
      parallel {
        stage('Style Check') {
          environment {
            HOME = "${env.WORKSPACE}"
          }
          steps {
            sh 'cookstyle .'
          }
        }
        stage('Policyfile Lock') {
          environment {
            HOME = "${env.WORKSPACE}"
          }
          steps {
            sh 'chef install'
          }
        }
      }
    }
    stage('Testing') {
      when { expression { Kitchen action: 'isRunKitchenTests' } }
      environment {
        HOME = "${env.HOME}"
        VCENTER_CRED = credentials('YourVcenterCredsinJenkins')
        VCENTER_USER = "$VCENTER_CRED_USR"
        VCENTER_PASSWORD = "$VCENTER_CRED_PSW"
        VCENTER_HOST = 'vcenterhost.com'
        VCENTER_VMFOLDER = 'folder vms are created in'
        VCENTER_NETWORK = 'network_in_vcenter'
        VCENTER_CLUSTER = 'your_vcenter_cluster'
        VCENTER_DC = 'your_vcenter_datacenter'
        KITCHEN_CRED = credentials('YourKitchenCredsinJenkins')
        KITCHENUSER = "$KITCHEN_CRED_USR"
        KITCHENPASS = "$KITCHEN_CRED_PSW"
      }
      parallel {
        stage('Integration Testing') {
          steps {
            sh 'mv kitchen.yml kitchen.notused'
            sh 'mv kitchen.pipeline.yml kitchen.yml'
            sh 'kitchen converge --color'
            sh 'kitchen verify --color'
            sh 'kitchen converge --color'
            sh 'kitchen verify --color'
          }
          post {
            always {
              sh 'kitchen destroy'
            }
          }
        }
        stage('Unit Testing') {
          steps {
            sh 'chef exec rspec'
          }
        }
      }
    }
    stage('Finalize') {
      steps {
        sh 'git status'
      }
    }
    stage('Publish') {
      when {
        allOf {
          branch 'master'
        }
      }
      environment {
        ARTIF_CREDS = credentials('artifactory_id')
        ARTIF_USER = "$ARTIF_CREDS_USR"
        ARTIF_PASSWORD = "$ARTIF_CREDS_PSW"
        ARTIF_URL = "https://$ARTIF_USER:$ARTIF_PASSWORD@yourartifactoryrepo.com/artifactory/api/chef/chef"
      }
      steps {
        echo 'Publishing to Artifactory Supermarket ...'
        script {
          def jsonData = readJSON file: "${env.WORKSPACE}/pipeline.json"
          sh "mkdir artifactory_stage"
          sh "unzip ${jsonData.repo_name}.zip -d artifactory_stage/${jsonData.repo_name}"
          sh "knife ssl fetch -s yourartifactoryrepo.com"
          sh "knife artifactory share ${jsonData.repo_name} -m $ARTIF_URL -o artifactory_stage/ "
        }
        build job: 'inc_compliance', propagate: true, wait: true
      }
      }
      }
    post {
    success {
      echo 'The build was successful please merge'
    }
    failure {
      echo 'The build failed'
    }
    always {
      cleanWs()
    }
  }
}
```

</details>

***Kitchen.yml with vcenter Example***
**This has been collapsed due to size**

<details>
<summarize>kitchen-vcenter</summarize>

```yaml
---
driver:
  name: vcenter
  vcenter_username: <%= ENV['VCENTER_USER'] %>
  vcenter_password: <%= ENV['VCENTER_PASSWORD'] %>
  vcenter_host:  <%= ENV['VCENTER_HOST'] %>
  datacenter: <%= ENV['VCENTER_DC'] %>
  cluster: <%= ENV['VCENTER_CLUSTER'] %>
  interface: <%= ENV['VCENTER_NETWORK'] %>
  folder: <%= ENV['VCENTER_VMFOLDER'] %>
  vcenter_disable_ssl_verify: true
  clone_type: linked
  customize:
    annotation: "Kitchen VM generated by CICD pipeline"

provisioner:
  name: chef_zero
  sudo_command: sudo
  retry_on_exit_code:
    - 35 # 35 is the exit code signaling that the node is rebooting
  max_retries: 1
  wait_for_retry: 90
  # always_update_cookbooks: true
  client_rb:
    chef_license: accept

verifier:
  name: inspec

platforms:
  - name: RHEL7
    driver:
      template: 'Templates/LabJnk/RHEL7'
    transport:
      username: <%= ENV['KITCHENUSER'] %>
      password: <%= ENV['KITCHENPASS'] %>
  - name: RHEL6
    driver:
      template: 'Templates/LabJnk/RHEL6'
    transport:
      username: <%= ENV['KITCHENUSER'] %>
      password: <%= ENV['KITCHENPASS'] %>
  - name: WIN2008R2
    driver:
      template: 'Templates/LabJnk/WIN2008R2'
    transport:
      username: <%= ENV['KITCHENUSER'] %>
      password: <%= ENV['KITCHENPASS'] %>
  - name: WIN2012R2
    driver:
      template: 'Templates/LabJnk/WIN2012R2'
    transport:
      username: <%= ENV['KITCHENUSER'] %>
      password: <%= ENV['KITCHENPASS'] %>
  - name: WIN2016
    driver:
      template: 'Templates/LabJnk/WIN2016'
    transport:
      username: <%= ENV['KITCHENUSER'] %>
      password: <%= ENV['KITCHENPASS'] %>

suites:
  - name: set1
    verifier:
      inspec_tests:
        - test/integration/rhel
    includes:
     - RHEL6
    lifecycle:
     post_create:
      remote: 'sudo hostname sets-$((100000 + RANDOM % 999990)) ; sudo yum -y install http://satteliteserver.com/pub/katello-ca-consumer-latest.noarch.rpm ; sudo subscription-manager register --org="ORG" --activationkey="pipeline" --force'
     pre_destroy:
      remote: 'sudo subscription-manager unregister'
  - name: set2
    verifier:
      inspec_tests:
        - test/integration/rhel
    includes:
     - RHEL7
    lifecycle:
     post_create:
      remote: 'sudo hostname sets-$((100000 + RANDOM % 999990)) ; sudo yum -y install http://satellitserver.com/pub/katello-ca-consumer-latest.noarch.rpm ; sudo subscription-manager register --org="ORG" --activationkey="pipeline" --force'
     pre_destroy:
      remote: 'sudo subscription-manager unregister'
  - name: set3
    named_run_list: testmode
    attributes:
      hardening:
        testmode: 1
    provisioner:
      download_url: http://interalrepo.com/chef/chef-infra/15/chef-client-15.4.45-1-x64.msi
    verifier:
      inspec_tests:
        - test/integration/win2008
    includes:
      - WIN2008R2
  - name: set4
    named_run_list: testmode
    attributes:
      hardening:
        testmode: 1
    provisioner:
      download_url: http://interalrepo.com/chef/chef-infra/15/chef-client-15.4.45-1-x64.msi
    verifier:
      inspec_tests:
        - test/integration/win2012
    includes:
      - WIN2012R2
  - name: set5
    named_run_list: testmode
    attributes:
      hardening:
        testmode: 1
    provisioner:
      download_url: http://interalrepo.com/chef/chef-infra/15/chef-client-15.4.45-1-x64.msi
    verifier:
      inspec_tests:
        - test/integration/win2016
    includes:
      - WIN2016
```
</details>

***Sending email as `post` action***

```java
emailext (
    subject: "Job '${env.JOB_NAME} ${env.BUILD_NUMBER}'",
    body: """<p>Check console output at <a href="${env.BUILD_URL}">${env.JOB_NAME}</a></p>""",
    to: "email@theirs.com",
    from: "jenkins@theirs.com"
)
```
