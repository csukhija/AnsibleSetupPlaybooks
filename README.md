Ansible Role for Jenkins
========================

Installs and completely configures Jenkins using Ansible. Full Automation Possible 

This role is used when you want all your Jenkins configuration 
in version control so you can deploy Jenkins repeatably 
and reliably and you can treat your Jenkins as a [Cow instead 
of a Pet](https://blog.engineyard.com/2014/pets-vs-cattle).

If you are looking for a role to install Jenkins and you 
are ok treating your Jenkins as a pet then you don't need 
this role, have a look at the 
[geerlingguy/ansible-role-jenkins](https://github.com/geerlingguy/ansible-role-jenkins)
role instead.

Requirements
------------

Requires curl to be installed on the server.

If deploying using Docker then you need Docker 
installed on the server.

(Docker is the only supported way at the moment 
although more ways can easily be added, PRs welcome).

Installation
------------

Install using ansible galaxy:

```
$ ansible-galaxy install emmetog.jenkins
```

Role Variables
--------------

```yml
jenkins_version: "2.32.3" # The exact version of jenkins to deploy
jenkins_url: "http://127.0.0.1" # The url that Jenkins will be accessible on
jenkins_port: "8080" # The port that Jenkins will listen on
jenkins_home: /data/jenkins # The directory on the server where the Jenkins configs will live

# If you need to override any java options then do that here.
jenkins_java_opts: "-Djenkins.install.runSetupWizard=false"

# Configuration files owner and group
jenkins_config_owner: "ubuntu"
jenkins_config_group: "ubuntu"

# The locations of the configuration files for jenkins
jenkins_source_dir_configs: "{{ playbook_dir }}/jenkins-configs"
jenkins_source_dir_jobs: "{{ jenkins_source_dir_configs }}/jobs"

# config.xml template source
jenkins_source_config_xml: "{{ jenkins_source_dir_configs }}/config.xml"

# Include custom files for jenkins installation
jenkins_include_custom_files: false
jenkins_custom_files: {}

# Include secrets directory during installation
jenkins_include_secrets: false
jenkins_source_secrets: "{{ jenkins_source_dir_configs }}/secrets/"

# The names of the jobs (config.xml must exist under jenkins_source_dir_jobs/job_name/)
jenkins_jobs: []

# These plugins will be installed in the jenkins instance
jenkins_plugins:
  - git
  - log-parser
  - copyartifact
  - workflow-aggregator
  - workflow-multibranch
  - docker-workflow
  - template-project
  - ec2

# List of sources of custom jenkins plugins to install
jenkins_custom_plugins: []

# Configs specific to the "docker" method of running jenkins
jenkins_docker_container_name: jenkins
```

Example Playbook
----------------

```yml
- hosts: jenkins

  vars:
    jenkins_version: "1.642.4"
    jenkins_url: http://jenkins.example.com
    jenkins_port: 80
    jenkins_install_via: "docker"
    jenkins_jobs:
        - "my-cool-job"
        - "another-awesome-job"
    jenkins_include_secrets: true
    jenkins_include_custom_files: true
    jenkins_custom_files:
      - src: "jenkins.plugins.openstack.compute.UserDataConfig.xml"
        dest: "jenkins.plugins.openstack.compute.UserDataConfig.xml"
    jenkins_custom_plugins:
        - "openstack-cloud-plugin/openstack-cloud.jpi"
      
  roles:
    - emmetog.jenkins
```

Jenkins Configs
---------------

The example above will look for the job configs in 
`{{ playbook_dir }}/jenkins-configs/jobs/my-cool-job/config.xml` and 
`{{ playbook_dir }}/jenkins-configs/jobs/another-awesome-job/config.xml`. 

***NOTE***: These directories are customizable, see the `jenkins_source_dir_configs` and `jenkins_source_dir_jobs` role variables.

The role will also look for `{{ playbook_dir }}/jenkins-configs/config.xml`
These config.xml will be templated over to the server to be used as the job configuration.
It will upload the whole secrets directory under `{{ playbook_dir }}/jenkins-configs/secrets` and configure custom files provided under `{{ jenkins_custom_files }}` variable. Note that `{{ jenkins_include_secrets }}` and `{{ jenkins_include_custom_files }}` varibales should be set to true for these to work.
Additionaly the role can install custom plugins by providing the .jpi or .hpi files as a list under `{{ jenkins_custom_plugins }}` variable.

config.xml and custom files are templated so you can put variables in them,
for example it would be a good idea to encrypt sensitive variables 
in ansible vault.

Example Job Configs
-------------------

Here's an example of what you could put in `{{ playbook_dir }}/jenkins-configs/jobs/my-cool-job/config.xml`:

```xml
<?xml version='1.0' encoding='UTF-8'?>
<flow-definition plugin="workflow-job@2.2">
  <actions/>
  <description>My Cool Job</description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsScmFlowDefinition" plugin="workflow-cps@2.4">
    <scm class="hudson.plugins.git.GitSCM" plugin="git@2.4.4">
      <configVersion>2</configVersion>
      <userRemoteConfigs>
        <hudson.plugins.git.UserRemoteConfig>
          <url>git@github.com:emmetog/ansible-jenkins.git</url>
          <credentialsId>github-deploy-key-jenkins</credentialsId>
        </hudson.plugins.git.UserRemoteConfig>
      </userRemoteConfigs>
      <branches>
        <hudson.plugins.git.BranchSpec>
          <name>*/master</name>
        </hudson.plugins.git.BranchSpec>
      </branches>
      <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
      <submoduleCfg class="list"/>
      <extensions/>
    </scm>
    <scriptPath>Jenkinsfile</scriptPath>
  </definition>
  <triggers/>
</flow-definition>
```

Example Jenkins Configs
-----------------------

In `{{ jenkins_source_dir_configs }}/config.xml` you put your global 
Jenkins configuration, for example:
```xml
<?xml version='1.0' encoding='UTF-8'?>
<hudson>
    <disabledAdministrativeMonitors/>
    <version></version>
    <numExecutors>2</numExecutors>
    <mode>NORMAL</mode>
    <useSecurity>true</useSecurity>
    <authorizationStrategy class="hudson.security.AuthorizationStrategy$Unsecured"/>
    <securityRealm class="hudson.security.SecurityRealm$None"/>
    <disableRememberMe>false</disableRememberMe>
    <projectNamingStrategy class="jenkins.model.ProjectNamingStrategy$DefaultProjectNamingStrategy"/>
    <workspaceDir>${ITEM_ROOTDIR}/workspace</workspaceDir>
    <buildsDir>${ITEM_ROOTDIR}/builds</buildsDir>
    <jdks/>
    <viewsTabBar class="hudson.views.DefaultViewsTabBar"/>
    <myViewsTabBar class="hudson.views.DefaultMyViewsTabBar"/>
    <clouds>
        <hudson.plugins.ec2.EC2Cloud plugin="ec2@1.33">
            <name>ec2-slave-docker-ec2</name>
            <useInstanceProfileForCredentials>false</useInstanceProfileForCredentials>
            <credentialsId>jenkins-aws-ec2</credentialsId>

            <privateKey class="com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey$DirectEntryPrivateKeySource">
                <privateKey>{{ ssh_jenkins_aws_key }}</privateKey>
            </privateKey>

            <instanceCap>1</instanceCap>
            <templates>
                <hudson.plugins.ec2.SlaveTemplate>
                    <ami>ami-2654d755</ami>
                    <description>Docker builder</description>
                    <zone>eu-west-1c</zone>
                    <securityGroups>ssh-only</securityGroups>
                    <remoteFS></remoteFS>
                    <type>T2Micro</type>
                    <ebsOptimized>false</ebsOptimized>
                    <labels>docker</labels>
                    <mode>NORMAL</mode>
                    <initScript></initScript>
                    <tmpDir></tmpDir>
                    <userData></userData>
                    <numExecutors>1</numExecutors>
                    <remoteAdmin>ubuntu</remoteAdmin>
                    <jvmopts></jvmopts>
                    <subnetId></subnetId>
                    <idleTerminationMinutes>30</idleTerminationMinutes>
                    <iamInstanceProfile></iamInstanceProfile>
                    <useEphemeralDevices>false</useEphemeralDevices>
                    <customDeviceMapping></customDeviceMapping>
                    <instanceCap>2147483647</instanceCap>
                    <stopOnTerminate>true</stopOnTerminate>
                    <usePrivateDnsName>false</usePrivateDnsName>
                    <associatePublicIp>false</associatePublicIp>
                    <useDedicatedTenancy>false</useDedicatedTenancy>
                    <amiType class="hudson.plugins.ec2.UnixData">
                        <rootCommandPrefix></rootCommandPrefix>
                        <sshPort>22</sshPort>
                    </amiType>
                    <launchTimeout>2147483647</launchTimeout>
                    <connectBySSHProcess>false</connectBySSHProcess>
                    <connectUsingPublicIp>false</connectUsingPublicIp>
                </hudson.plugins.ec2.SlaveTemplate>
            </templates>
            <region>eu-west-1</region>
        </hudson.plugins.ec2.EC2Cloud>
    </clouds>
    <quietPeriod>5</quietPeriod>
    <scmCheckoutRetryCount>0</scmCheckoutRetryCount>
    <views>
        <hudson.model.AllView>
            <owner class="hudson" reference="../../.."/>
            <name>All</name>
            <filterExecutors>false</filterExecutors>
            <filterQueue>false</filterQueue>
            <properties class="hudson.model.View$PropertyList"/>
        </hudson.model.AllView>
    </views>
    <primaryView>All</primaryView>
    <slaveAgentPort>50000</slaveAgentPort>
    <label></label>
    <nodeProperties/>
    <globalNodeProperties/>
</hudson>
```

Making Changes
--------------

When you want to make a big change in a configuration file 
or you want to add a new job the normal workflow is to make 
the change in the Jenkins UI 
first, then copy the resulting XML back into your VCS.

License
-------

MIT

Author Information
------------------

Made with love by Emmet O'Grady.
