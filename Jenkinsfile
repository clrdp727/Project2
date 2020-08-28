#!groovy
import groovy.json.JsonSlurperClassic

    //Method to return the project directory name
    def getFolderName() {
        def array = pwd().split("/")
        def folderNameList = array[array.length - 2].split('\\\\')
        def projectFolderName = folderNameList[folderNameList.size()-1].split('@')[0]
        return projectFolderName;
    }

node {

    def BUILD_NUMBER=env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
    def SFDC_USERNAME

    def projectFolderName1 = getFolderName()
    print "Project Directory Name--"+"${projectFolderName1}"
    
    //Fetching the project specific properties of the environment variables
    def HUB_ORG=env.getProperty("${projectFolderName1}"+'_User_Name') // to get the username
    def SFDC_HOST = env.getProperty("${projectFolderName1}"+'_Login_URL') // to get the salesforce login url
    def JWT_KEY_CRED_ID = env.getProperty("${projectFolderName1}"+'_JWT_KEY_CRED_ID'); // to get the JWT Id
    def CONNECTED_APP_CONSUMER_KEY=env.getProperty("${projectFolderName1}"+'_CONNECTED_APP_CONSUMER_KEY'); // to get the org consumer key 
    def checkonly = env.getProperty("${projectFolderName1}"+'_Checkonly'); // to get checkonly flag value

    println ("checkonly"+checkonly)

    println 'KEY IS' 
    println JWT_KEY_CRED_ID
    println HUB_ORG
    println SFDC_HOST
    println CONNECTED_APP_CONSUMER_KEY
    def toolbelt = tool 'toolbelt'

    stage('checkout source') {
        // when running in multi-branch job, one must issue this command
        checkout scm
    }

    withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {
        stage('Deploye Code') {
            if (isUnix()) {
                rc = sh returnStatus: true, script: "${toolbelt} force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
            }else{
                rc = bat returnStatus: true, script: "\"${toolbelt}\" force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile \"${jwt_key_file}\" --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"		    
                def rc2 = bat (returnStdout: true, script: "git diff --name-only HEAD HEAD~1").trim()
                result = rc2.readLines().drop(1)
                def folderString
                for(int  i=0; i<result.size();i++){
                    folderString=''
                    println("Res"+i+"->"+result[i])
                    splittedParts = result[i].split('/')
                    echo splittedParts[splittedParts.size()-1]
                    for(int j=0; j<splittedParts.size()-1;j++){
                        folderString=folderString+splittedParts[j]+"\\"
                    }
                    echo folderString
                    rc3 = bat returnStatus: true, script: "mkdir C:\\deploy-cmp\\${folderString}"
                    correctstring = result[i].split('/').join('\\');
                    rc4 = bat returnStatus: true, script: "copy ${correctstring} C:\\deploy-cmp\\${folderString}"
                    rc5 = bat returnStatus: true, script: "copy ${correctstring}-meta.xml C:\\deploy-cmp\\${folderString}"			    
                }
            }
            if (rc != 0) { error 'hub org authorization failed' }

			println rc

			// need to pull out assigned username
			if (isUnix()) {
				rmsg = sh returnStdout: true, script: "${toolbelt} force:mdapi:deploy -d manifest/. -u ${HUB_ORG}"
			}else{
			   rmsg = bat returnStdout: true, script: "\"${toolbelt}\" force:source:deploy -u ${HUB_ORG} --sourcepath C:\\deploy-cmp\\force-app\\main\\default\\"
               def commitDetail = bat (returnStdout: true, script: "git log --oneline -n 1").trim().readLines().drop(1)
               println("commitDetail-"+commitDetail+"--------");
               rmsg2 = bat returnStdout: true, script: "\"${toolbelt}\" force:data:record:create -u ${HUB_ORG} -s Deployment_Status__c -v \"Description__c='${commitDetail}'\""
			}
			  
            printf rmsg
            println('Deployment is Finished Successfully three!!')
            println(rmsg)
            rc5 = bat returnStatus: true, script: "cd C:\\deploy-cmp"			    
            rc6 = bat returnStatus: true, script: "cd C:\\deploy-cmp & rmdir /Q /S force-app"			    

            // Destructive Changes
            try {
                installSfPowerkit = bat returnStdout: true, script: "\"${toolbelt}\" plugins:install sfpowerkit"
                createXML = bat returnStdout: true, script: "\"${toolbelt}\" sfpowerkit:project:diff -r HEAD~1 -d  ..\\sfpowerkitDiff -x --loglevel trace"
                copyXML = bat returnStdout: true, script: "copy ..\\sfpowerkitDiff\\destructiveChanges.xml ..\\sfpowerkitDeploy\\"
                mdapiDeploy = bat returnStdout: true, script: "\"${toolbelt}\" force:mdapi:deploy -d ..\\sfpowerkitDeploy\\ -w 30 -u  ${HUB_ORG} --loglevel trace"
                cleanDestructiveXML = bat returnStatus: true, script: "cd ..\\sfpowerkitDeploy & del destructiveChanges.xml"			    
            }catch(ex){

            }
        }
    }
}
