properties([
  parameters([
    string(name: 'DataCenter', defaultValue: 'DC25DEV'),
    string(name: 'Zone', defaultValue: 'YZ'),
    string(name: 'DataCenterCode', defaultValue: 'DC25DEV'),
    string(name: 'DeploymentNode', defaultValue: 'DEPLOYER'),
    string(name: 'Prov_Pool', defaultValue: 'dc25devlr50'),
    string(name: 'pool_list', defaultValue: 'dc25devlp50,dc25devlp52', description: 'Pool list seperated by commas.'),
    booleanParam(name: 'SKIP_RUN_AUTOMATION_APP_AND_DB_JOBS', defaultValue: false, description: 'Uncheck the option if you want to trigger the RunAutomation jobs for APP and DB, If not only the App and DB verification jobs will be triggered.'),
    booleanParam(name: 'SKIP_PROVISIONING_DEPLOYMENT', defaultValue: false, description: 'Uncheck the option if you want to trigger the RunAutomation jobs for Provisioning APP'),

  ])
])

//Automation
def datacenter=params.DataCenter
//def zoneValue=params.Zone
def datacentercode=params.DataCenterCode
def deploymentnode=params.DeploymentNode
def provpool=params.Prov_Pool

def pool_params = params.pool_list.split(',')
def consolidated_email="";
def qacand_pools=""
def qaautocand_pools=""
def parallel_deployment_job_object = [:]

def appShutdownJobList = [:]
def runAutomationDBList = [:]
def dbVerificationList = [:]
def runAutomationAPPList = [:]
def appVerificationList = [:]
def build_version = "b2211.0.1.202208241029"

def oneTimeConfigJob=null
def provisioningAppJob=null


//parallel stages for all pools (QACAND + QAAUTOCAND)
for (int f = 0; f < pool_params.length; f++) {
  def pool_name=pool_params[f]
  def zone_value="YZ"
  if (pool_name.startsWith("mc")){
    zone_value="QACAND" 
  }
  else if (pool_name.startsWith("ac")){
      zone_value="QAAUTOCAND"
  }
  else if (pool_name.startsWith("pc")){
      zone_value="RZ"
  }
  else if (pool_name.startsWith("dc")){
      zone_value="YZ"
  }
  
  parallel_deployment_job_object["${pool_params[f]}"] = {

    stage("${f}") {

        def db_job_obj
        //Triggering App_Server_Rolling_Restart - Shutdown App
        if (!params.SKIP_RUN_AUTOMATION_APP_AND_DB_JOBS) {

            echo "Triggering App_Server_Rolling_Restart - Shutdown App - Job for ${pool_name}"
            app_stop_job_obj=build job: "App_Server_Rolling_Restart", propagate: false, wait: true, parameters:
          [
            [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
            [$class: 'StringParameterValue', name: 'Zone', value: "${zone_value}"],
            [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacenter}"],
            [$class: 'StringParameterValue', name: 'pool_id', value: "${pool_name}"],
            [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"],
            [$class: 'StringParameterValue', name: 'Action', value: 'shutdown_app']
          ]
          appShutdownJobList["${pool_name}"]=app_stop_job_obj

          if(app_stop_job_obj!=null && app_stop_job_obj.getResult()!=null){
            echo "App_Server_Rolling_Restart - Shutdown App - ${pool_name}: ${app_stop_job_obj.getResult()}"
          }
            //Triggering RunAutomation DB
            echo "Triggering RunAutomation DB for ${pool_name}"
            db_job_obj=build job: "RunAutomation", propagate: false, wait: true, parameters:
            [
              [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'Zone', value: "${zone_value}"],
              [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'pool_id', value: "${pool_name}"],
              [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"],
              [$class: 'StringParameterValue', name: 'Type', value: 'App_Pool_DB_Only']
            ]
            runAutomationDBList["${pool_name}"]=db_job_obj

              if(db_job_obj!=null && db_job_obj.getResult()!=null){
                echo "RunAutomation DB for ${pool_name}: ${db_job_obj.getResult()}"
              }
          }
          //Triggering DB Verification
          echo "Triggering DB Verification for ${pool_name}"
          def db_ver_job_obj=build job: "DB_Verification_Task", propagate: false, wait: true, parameters:
            [
              [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'Zone', value: "${zone_value}"],
              [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"],
              [$class: 'StringParameterValue', name: 'tenant_id', value: ''],
              [$class: 'StringParameterValue', name: 'pool_id', value: "${pool_name}"],
              [$class: 'StringParameterValue', name: 'lms_version', value: '']
            ]

          dbVerificationList["${pool_name}"]=db_ver_job_obj

          if(db_ver_job_obj!=null && db_ver_job_obj.getResult()!=null){
            echo "DB Verification for ${pool_name}: ${db_ver_job_obj.getResult()}"
            if(db_job_obj){
            if(!db_ver_job_obj.getResult().equals("SUCCESS") || !db_job_obj.getResult().equals("SUCCESS")){
              sendFailureEmail("${pool_name}","DB",db_job_obj,db_ver_job_obj)
            }
          }
          }

          def app_job_obj

          if((db_ver_job_obj.getResult().equals("SUCCESS") && params.SKIP_RUN_AUTOMATION_APP_AND_DB_JOBS) || (db_ver_job_obj.getResult().equals("SUCCESS") && db_job_obj.getResult().equals("SUCCESS"))) {
          //Triggering RunAutomation - App_Pool_App_Nodes_Only
          if (!params.SKIP_RUN_AUTOMATION_APP_AND_DB_JOBS) {
              echo "Triggering RunAutomation APP for ${pool_name}"
              app_job_obj=build job: "RunAutomation", propagate: false, wait: true, parameters:
              [
                [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
                [$class: 'StringParameterValue', name: 'Zone', value: "${zone_value}"],
                [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacenter}"],
                [$class: 'StringParameterValue', name: 'pool_id', value: "${pool_name}"],
                [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"],
                [$class: 'StringParameterValue', name: 'Type', value: 'App_Pool_App_Nodes_Only']
              ]
              runAutomationAPPList["${pool_name}"]=app_job_obj

              if(app_job_obj!=null && app_job_obj.getResult()!=null){
                echo "RunAutomation APP for ${pool_name}: ${app_job_obj.getResult()}"
              }
            }
          }
          //Triggering APP Verification task
          echo "Triggering APP Verification for ${pool_name}"
          def app_ver_job_obj=build job: "APP_Verification_Tasks", propagate: false, wait: true, parameters:
            [
              [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'Zone', value: "${zone_value}"],
              [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'pool_id', value: "${pool_name}"],
              [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"]
            ]
            appVerificationList["${pool_name}"]=app_ver_job_obj

            if(app_ver_job_obj!=null && app_ver_job_obj.getResult()!=null){
              echo "APP Verification for ${pool_name}: ${app_ver_job_obj.getResult()}"
              if(app_job_obj){

              if(!app_job_obj.getResult().equals("SUCCESS")){
                sendFailureEmail("${pool_name}","App",app_job_obj,app_ver_job_obj)
              }
              }
              if(app_ver_job_obj.getResult().equals("FAILURE") || (db_ver_job_obj!=null && db_ver_job_obj.getResult().equals("FAILURE"))){
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "exit 1"
                }
              }
            }


    } 
  } 
}


// Pipeline deployment
pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }
    stages {
        //Fetching the build version from Jfrog 
        stage ('Initiating the deployment'){
            steps{
                script{
                    echo "Fetching the build version from Jfrog"
                    //build_version = sh(
                        //script: 'curl -s -H "X-JFrog-Art-Api:AKCp8ihVhjySe4zwmVX7u2Au8KqpVZXRbpP1UWpzaxxucT7xTXi16xtTQeYke6ki3d6MUDcDF" -X GET "https://common.repositories.cloud.sap/artifactory/sf-common/lms/QUARTERLY_BUILDS/DCE/Fileupload.txt" | sed -E 's/.+b([0-9\.]+).zip/b\1/g'| head -1',
                        //returnStdout: true
                    //).trim()
                    echo "Latest build version -${build_version}"
                }
            }
        }
        //Running "copy Jfrog artifacts" job using build version 
        stage ('copy Jfrog artifacts'){
            steps {
                  script{
                echo "Triggering QC-Copy-JFROG-Artifact job "
                //qcCopyJfrogArtifactJob=build job: 'QC-Copy-JFROG-Artifacts', propagate: true, wait: true, parameters:
                [
                    [$class: 'StringParameterValue', name: 'buildFor', value: "QUARTERLY_BUILDS"],
                    [$class: 'StringParameterValue', name: 'versionNumber', value: "${build_version}"],
                    [$class: 'StringParameterValue', name: 'DataCenter', value: "ALL"],
                ]

            }
            }
          
        }
        //Running One time configuration to pull the latest changes from Repo
        stage('ONE_TIME_CONFIG') {
          when {
               expression { !params.SKIP_ONE_TIME_CONFIG }
           }
          steps {
            script {
          echo 'Triggering ONE_TIME_CONFIG Job'
          //oneTimeConfigJob=build job: 'OneTime-Config-Setup', propagate: true, wait: true, parameters:
            [
              [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacenter}"],
              [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"],
              [$class: 'StringParameterValue', name: 'EnvironmentType', value: 'qa']
            ]
          }
          }

        } 
        // Running automation for Provisioning app
         stage('PROVISIONING_APP') {
          when {
               expression { !params.SKIP_PROVISIONING_APP_DEPLOYMENT }
           }
          steps {
          script {
             echo 'Triggering Provisioning App Job'
              //provisioningAppJob=build job: 'RunAutomation', propagate: false, wait: true, parameters:
            [
            [$class: 'StringParameterValue', name: 'DataCenter', value: "${datacenter}"],
            [$class: 'StringParameterValue', name: 'Zone', value: 'YZ'],
            [$class: 'StringParameterValue', name: 'DataCenterCode', value: "${datacentercode}"],
            [$class: 'StringParameterValue', name: 'pool_id', value: "${provpool}"],
            [$class: 'StringParameterValue', name: 'DeploymentNode', value: "${deploymentnode}"],
            [$class: 'StringParameterValue', name: 'Type', value: 'Provisioning_App']
            ]
                    }
          } 
        } 
        // Running parallel deployment for all the pools
        stage('Deployment') {
          steps {
            script {
                parallel parallel_deployment_job_object
            }
          }
        } 
        //
        stage('Tenant validation') {
          when {
               expression { !params.SKIP_TENANT_VALIDATION }
           }
          steps {
            script {
              echo 'Triggering tenant validation '
              //parallel parallel_tenant_validation
            }
          }
        } // Deployment

        stage('Generate_Report') {
          steps {
            script {
              def header_tbl="<h2 style='text-align: left; color:#444481;font-family: Times New Roman, Times, serif;'><u>DC25 QA DEPLOYMENT STATUS</u></h2><p style='font-family: Times New Roman, Times, serif;'>Hi All,</p><p style='font-family: Times New Roman, Times, serif;'>We have completed the deployment status for TMS version:${build_version}</p>"
              def qacand_tbl="<b>QACAND</b><table border=1><tr><th>POOL ID</th><th>TMS VERSION</th><th>APP Shutdown</th><th>DB JOB</th><th>DB VERIFICATION</th><th>APP JOB</th><th>APP VERIFICATION</th></tr>"
              def qaautocand_tbl='<head> <style> #customers { font-family: "Times New Roman", Times, serif; font-size: 15px; border-collapse: collapse; width: 80%; height: 10px;} #customers td, #customers th { border: 1px solid #ddd; padding: 8px;  } #customers tr:nth-child(even){background-color: #f2f2f2;} #customers tr:hover {background-color: #ddd;} #customers th { padding-top: 10px; padding-bottom: 10px; text-align: center; background-color: #444481; color: white; } </style> </head> <body><h4 style="font-family: Times New Roman, Times, serif; color:#444481;"><u>QAAUTOCAND</u></h3> <table id="customers"><tr><th align=center>POOL ID</th><th align=center>TMS VERSION</th><th align=center>APP Shutdown</th><th align=center>DB JOB</th><th align=center>DB VERIFICATION</th><th align=center>APP JOB</th><th align=center>APP VERIFICATION</th></tr></body>'
              // def qaautocand_tbl="<BR><b>QAAUTOCAND</b><table border=1><tr><th>POOL ID</th><th>TMS VERSION</th><th>APP Shutdown</th><th>DB JOB</th><th>DB VERIFICATION</th><th>APP JOB</th><th>APP VERIFICATION</th></tr>"
              def other_job_tbl='<head> <style> #customers { font-family: "Times New Roman", Times, serif; font-size: 15px; border-collapse: collapse; width: 80%; height: 10px; } #customers td, #customers th { border: 1px solid #ddd; padding: 8px; } #customers tr:nth-child(even){background-color: #f2f2f2;} #customers tr:hover {background-color: #ddd;} #customers th { padding-top: 10px; padding-bottom: 10px; text-align: center; background-color: #444481; color: white; } </style> </head> <body><h4 style="font-family: Times New Roman, Times, serif; color:#444481;"><u>PROVISIONING POOL</u></h3><table id="customers"><tr><th align=center>JOB NAME</th><th align=center>POOL ID</th><th align=center>TMS VERSION</th><th align=center>STATUS</th></tr><tr><td align=center>Provisioning Upgrade</td><td align=center>'+provpool+'</td><td align=center>'+build_version+'</td><td align=center>PROVISIONING_APP</td></tr></table><BR><p style="font-family: Times New Roman, Times, serif;">Regards,<BR>Release Engineering</p>'

              other_job_tbl=getTableRowForPool(provisioningAppJob,other_job_tbl,"PROVISIONING_APP")

              for (int f = 0; f < pool_params.length; f++) {
                def pool_list_id=pool_params[f]
                def row_text="<tr><td align=center>POOL_ID</td><td align=center>TMS_VERSION</td><td align=center>APP_SHUTDOWN</td><td align=center>DB_JOB</td><td align=center>DB_VERIFICATION</td><td align=center>APP_JOB</td><td align=center>APP_VERIFICATION</td></tr>"
                row_text=row_text.replace("POOL_ID","${pool_list_id}")
                echo "my first files"
                row_text=row_text.replace("TMS_VERSION","${build_version}")


                row_text=getTableRowForPool(appShutdownJobList["${pool_list_id}"],row_text,"APP_SHUTDOWN")
                row_text=getTableRowForPool(dbVerificationList["${pool_list_id}"],row_text,"DB_VERIFICATION")
                row_text=getTableRowForPool(appVerificationList["${pool_list_id}"],row_text,"APP_VERIFICATION")
                row_text=getTableRowForPool(runAutomationDBList["${pool_list_id}"],row_text,"DB_JOB")
                row_text=getTableRowForPool(runAutomationAPPList["${pool_list_id}"],row_text,"APP_JOB")
                row_text=getTableRowForPool(runAutomationAPPList["${pool_list_id}"],row_text,"APP_JOB")

                if(pool_list_id.startsWith("dc")){
                  qaautocand_tbl+="${row_text}"
                } else {
                  qacand_tbl+="${row_text}"
                }
              } // end of FOR loop for creating seperate list for QACAND and QAAUTOCAND jobs
              qacand_tbl+="</table>"
              qaautocand_tbl+="</table>"

              if(qacand_tbl.length()<200){
                qacand_tbl=""
              }

              if(qaautocand_tbl.length()<200){
                qaautocand_tbl=""
              }


              consolidated_email="${header_tbl}" + "${qacand_tbl}" + "${qaautocand_tbl}" + "${other_job_tbl}"

            }
          }
        } // Deployment

    } // stagess

    post {
      success {
          echo 'This will run only if successful - to send success email'
          mail body: "${consolidated_email}", charset: 'UTF-8', mimeType: 'text/html', subject: "Daily QA Deployment Status - Job:${currentBuild.number}", to: "gandhamanikandan.d@sap.com"
      }
      failure {
        echo 'This will run on failure - to send failure email'
        mail body: "The job with id ${currentBuild.number} has failed, please validate manually", charset: 'UTF-8', mimeType: 'text/html', subject: "Daily QA Deployment Job Failure - Job:${currentBuild.number}", to: "gandhamanikandan.d@sap.com"
      }
      unstable {
          echo 'This will run only if the run was marked as unstable  - to send unstable email'
          mail body: "The job with id ${currentBuild.number} is unstable, please validate manually", charset: 'UTF-8', mimeType: 'text/html', subject: "Daily QA Deployment Job Unstable - Job:${currentBuild.number}", to: "gandhamanikandan.d@sap.com"

      }
    } // post

} // pileline


def sendFailureEmail(pool_id_str,job_type,automation_job_object,verification_job_object){
  if(automation_job_object && verification_job_object){
    def auto_job_link="https://10.47.13.117:8443/job/${automation_job_object.getFullProjectName()}/${automation_job_object.getNumber()}/console"
    def ver_job_link="https://10.47.13.117:8443/job/${verification_job_object.getFullProjectName()}/${verification_job_object.getNumber()}/console"
    def automation_job_link="<a href=\"${auto_job_link}\">Job Link</a>"
    def verification_job_link="<a href=\"${ver_job_link}\">Job Link</a>"
    echo "my second files"
    def version="b2211.0.1.202208241029"
    def email_content="Hi All,<BR><BR>Please be informed that the Automation/Verification job for ${job_type} has failed for the pool: ${pool_id_str}.<BR><BR>Kindly review and fix if any issues with the below jobs and trigger any further jobs as needed for this pool manually.<BR><BR>RunAutomtion ${job_type}: ${automation_job_link}<BR>${job_type} Verification: ${verification_job_link}<BR>TMS Version: ${version}<BR><BR>Thanks,<BR>LMS RE"
    def email_subject="Daily QA deployment - ${job_type} Job Failure Notification - Job:${currentBuild.number}"
    mail body: "${email_content}", charset: 'UTF-8', mimeType: 'text/html', subject: "${email_subject}", to: "gandhamanikandan.d@sap.com"
  }
}


def getTableRowForPool(jobObject,row_text,replaceKeyWord){
  if(jobObject!=null){
    def db_job_dtls=jobObject.getNumber()+" - "+jobObject.getResult()+" - "+jobObject.getDurationString()
    def job_link="https://10.47.13.117:8443/job/${jobObject.getFullProjectName()}/${jobObject.getNumber()}/console"
    def db_job_dtls_with_link="<a href='${job_link}'>${db_job_dtls}</a>"
    //def db_job_dtls_with_link="<a href='${job_link}' style='text-decoration:none'>${db_job_dtls}</a>"
    if(db_job_dtls.contains("SUCCESS")){
        row_text=row_text.replace(">"+replaceKeyWord," style='color: #16A352'"+replaceKeyWord)
    } else if(db_job_dtls.contains("FAILURE")){
        row_text=row_text.replace(">"+replaceKeyWord," style='color: #D84949'"+replaceKeyWord)
    } else if(db_job_dtls.contains("ABORTED")){
        row_text=row_text.replace(">"+replaceKeyWord," style='color: #1C9D9D'"+replaceKeyWord)
    }

    row_text=row_text.replace(replaceKeyWord,"${db_job_dtls_with_link}")
  }else{
    row_text=row_text.replace(replaceKeyWord,"Skipped")
  }
    return row_text;
}




