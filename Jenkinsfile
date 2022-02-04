String tfProject             = "nia"
String tfPrimaryDeploymentEnv     = "kdev"
//String tfSecondaryDeploymentEnv   = "kdev"
String tfComponent           = "pss"

pipeline {
    agent{
        label 'jenkins-workers'
    }
    
        options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: "10")) // keep only last 10 builds
    }
    
    environment {
        EXP_TEST = false
        GIT_BRANCH = "main"
        BUILD_TAG = "4abcdef"
    }

    stages {
        stage('Terraform Information Gathering') {
            steps {
                script {
                    String tfCodeBranch  = "NIAD-2008-PS-DB-Migration-Script-Execution-During-Deployment"
                    String tfCodeRepo    = "https://github.com/nhsconnect/integration-adaptors"
                    String tfRegion      = "${TF_STATE_BUCKET_REGION}"
                    List<String> tfParams = []
                    Map<String,String> tfVariables = ["${tfComponent}_build_id": BUILD_TAG]
                    String terraformBinPath = tfEnv()
                    //String tfOutput=[]
                    dir ("integration-adaptors") {
                        pwd
                        git (branch: tfCodeBranch, url: tfCodeRepo)
                        dir ("terraform/aws") {
                            pwd
                            if (terraformInit(TF_STATE_BUCKET, tfProject, tfPrimaryDeploymentEnv, tfComponent, tfRegion) !=0) { error("Terraform init failed")}
                            if (terraformOutput(TF_STATE_BUCKET, tfProject, tfPrimaryDeploymentEnv, tfComponent, tfRegion) !=0) { error("Terraform output failed")}
                            

                        }
                    }
                }
            }
        }
        
        stage('PSS DB Migration Code') {
            steps {
                script {
                    String pssCodeBranch  = "main"
                    String pssCodeRepo    = "https://github.com/NHSDigital/nia-patient-switching-standard-adaptor"
                    dir ("nia-patient-switching-standard-adaptor") {
                        pwd
                        git (branch: pssCodeBranch, url: pssCodeRepo)                            
                        
                        sh '''
                            
                            sed -i 's/ = /=/' ~/.psdbsecrets.tfvars
                            source ~/.psdbsecrets.tfvars
                            sed -i -e 's/^/export /g' -e 's/ = /=/g' ~/.tfoutput.tfvars
                            source ~/.tfoutput.tfvars
                            set
                            cd db-connector
                            ./gradlew update
                        '''
                    }
                }

             } // steps-PSS DB Migration Code
        } // stage-PSS DB Migration Code
    } //stages
} //pipeline


String tfEnv(String tfEnvRepo="https://github.com/tfutils/tfenv.git", String tfEnvPath="~/.tfenv") {
  sh(label: "Get tfenv" ,  script: "git clone ${tfEnvRepo} ${tfEnvPath}", returnStatus: true)
  sh(label: "Install TF",  script: "${tfEnvPath}/bin/tfenv install"     , returnStatus: true)
  return "${tfEnvPath}/bin/terraform"
}


// ---------------------------------------------- TERRAFORM FUNCTIONS START --------------------------------------------

int terraformInit(String tfStateBucket, String project, String environment, String component, String region) {
  String terraformBinPath = tfEnv()
  println("Terraform Init for Environment: ${environment} Component: ${component} in region: ${region} using bucket: ${tfStateBucket}")
  String command = "${terraformBinPath} init -backend-config='bucket=${tfStateBucket}' -backend-config='region=${region}' -backend-config='key=${project}-${environment}-${component}.tfstate' -input=false -no-color"
  dir("components/${component}") {
    return( sh( label: "Terraform Init", script: command, returnStatus: true))
  } // dir
} // int TerraformInit


int terraform(String action, String tfStateBucket, String project, String environment, String component, String region, Map<String, String> variables=[:], List<String> parameters=[]) {
    println("Running Terraform ${action} in region ${region} with: \n Project: ${project} \n Environment: ${environment} \n Component: ${component}")
    variablesMap = variables
    variablesMap.put('region',region)
    variablesMap.put('project', project)
    variablesMap.put('environment', environment)
    variablesMap.put('tf_state_bucket',tfStateBucket)
    parametersList = parameters
    parametersList.add("-no-color")

    // Get the secret variables for global
    String secretsFile = "etc/secrets.tfvars"
    writeVariablesToFile(secretsFile,getAllSecretsForEnvironment(environment,"nia",region))
    String terraformBinPath = tfEnv()
    List<String> variableFilesList = [
      "-var-file=../../etc/global.tfvars",
      "-var-file=../../etc/${region}_${environment}.tfvars",
      "-var-file=../../${secretsFile}"
    ]
    if (action == "apply"|| action == "destroy") {parametersList.add("-auto-approve")}
    List<String> variablesList=variablesMap.collect { key, value -> "-var ${key}=${value}" }
    String command = "${terraformBinPath} ${action} ${variableFilesList.join(" ")} ${parametersList.join(" ")} ${variablesList.join(" ")} "
    dir("components/${component}") {
      return sh(label:"Terraform: "+action, script: command, returnStatus: true)
    } // dir
} // int Terraform

int terraformOutput(String tfStateBucket, String project, String environment, String component, String region) {
  List<String> psDbSecretslist = getSecretsByPrefix("postgres",region)
  Map<String,Object> psDbSecretsExtracted = [:]
  Map<String,Object> psDbSecrets = [:]
    psDbSecretslist.each {
        String rawSecret = getSecretValue(it,region)
        psDbSecrets.put(it,rawSecret)
    }
    psDbSecretsExtracted.put("export PS_DB_OWNER_NAME",psDbSecrets.get('postgres-master-username'))
    psDbSecretsExtracted.put("export PS_DB_OWNER_PASSWORD",psDbSecrets.get('postgres-master-password'))
    psDbSecretsExtracted.put("export GP2GP_TRANSLATOR_USER_DB_PASSWORD",psDbSecrets.get('postgres_psdb_gp2gp_translator_user_password'))
    psDbSecretsExtracted.put("export GPC_FACADE_USER_DB_PASSWORD",psDbSecrets.get('postgres_psdb_gpc_facade_user_password'))

    writeVariablesToFile("~/.psdbsecrets.tfvars",psDbSecretsExtracted)

  
  String terraformBinPath = tfEnv()
  println("Terraform outputs for Environment: ${environment} Component: ${component} in region: ${region} using bucket: ${tfStateBucket}")
  String command = "${terraformBinPath} output > ~/.tfoutput.tfvars"
  dir("components/${component}") {
    return( sh( label: "Terraform Output", script: command, returnStatus: true))
  } // dir
} // int TerraformOutput

// ---------------------------------------------- TERRAFORM FUNCTIONS END --------------------------------------------


void writeVariablesToFile(String fileName, Map<String,Object> variablesMap) {
  List<String> variablesList=variablesMap.collect { key, value -> "${key} = ${value}" }
  sh (script: "touch ${fileName} && echo '\n' > ${fileName}")
  variablesList.each {
    sh (script: "echo '${it}' >> ${fileName}")
  }
}

Map<String,Object> getAllSecretsForEnvironment(String environment, String secretsPrefix, String region) {
  List<String> globalSecrets = getSecretsByPrefix("${secretsPrefix}-global",region)
  println "global secrets:" + globalSecrets
  List<String> environmentSecrets = getSecretsByPrefix("${secretsPrefix}-${environment}",region)
  println "env secrets:" + environmentSecrets
  Map<String,Object> secretsMerged = [:]
  globalSecrets.each {
    String rawSecret = getSecretValue(it,region)
    if (it.contains("-kvp")) {
      secretsMerged << decodeSecretKeyValue(rawSecret)
    } else {
      secretsMerged.put(it.replace("${secretsPrefix}-global-",""),rawSecret)
    }
  }
  environmentSecrets.each {
    String rawSecret = getSecretValue(it,region)
    if (it.contains("-kvp")) {
      secretsMerged << decodeSecretKeyValue(rawSecret)
    } else {
      secretsMerged.put(it.replace("${secretsPrefix}-${environment}-",""),rawSecret)
    }
  }
  return secretsMerged
}




List<String> getSecretsByPrefix(String prefix, String region) {
  String awsCommand = "aws secretsmanager list-secrets --region ${region} --query SecretList[].Name --output text"
  List<String> awsReturnValue = sh(script: awsCommand, returnStdout: true).split()
  return awsReturnValue.findAll { it.startsWith(prefix) }
}

// Retrieving Secrets from AWS Secrets
String getSecretValue(String secretName, String region) {
  String awsCommand = "aws secretsmanager get-secret-value --region ${region} --secret-id ${secretName} --query SecretString --output text"
  return sh(script: awsCommand, returnStdout: true).trim()
}

Map<String,Object> decodeSecretKeyValue(String rawSecret) {
  List<String> secretsSplit = rawSecret.replace("{","").replace("}","").split(",")
  Map<String,Object> secretsDecoded = [:]
  secretsSplit.each {
    String key = it.split(":")[0].trim().replace("\"","")
    Object value = it.split(":")[1]
    secretsDecoded.put(key,value)
  }
  return secretsDecoded
}






