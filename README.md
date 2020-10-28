# test
source /usr/local/etc/ocp4.config

oc login -u${RHT_OCP4_DEV_USER} -p ${RHT_OCP4_DEV_PASSWORD} ${RHT_OCP4_MASTER_API}

--------------------------------------------------------------------

oc new-project ${RHT_OCP4_DEV_USER}- ${NAME}

----------------------------------------------------------------------
git checkout -b ${new_branch_name}

git push -u origin ${new_branch_name}


---------------------------------------------------------------------
oc new-app  --as-deployment-config https://github.com/RedHatTraining/DO288-apps#master-apps --context-dir apache-httpd  
> for nodejs app use 
	oc new-app --as-deployment-config --name greet --build-env npm_config_registry=http://${RHT_OCP4_NEXUS_SERVER}/repository/nodejs nodejs:12~https://github.com/${RHT_OCP4_GITHUB_USER}/DO288-apps#source-build \
		--context-dir nodejs-helloworld

> oc new-app --as-deployment-config  mysql MYSQL_USER=user MYSQL_PASSWORD=pass  MYSQL_DATABASE=testdb -l db=mysql - env variavel


---------------------------------------------------------------------
oc delete all -l app=${APP_NAME}


---------------------------------------------------------------------
oc start-build ${APP_NAME}


------------------------------------------------------------------
oc get templates -n oepnshift  (can be user with | grep ${name})

-----------------------------------------------------------------------
oc rsh ${POD} bash -c - inicia um bash em um pod

--------------------------------------------------------------------------
oc cp ${arquivo_local} > ${POD}:$/{arquivo_remoto}

-----------------------------------------------------------------------
oc rsh -t ${POD} mysql -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE < /tmp/quote.sql 

------------------------------------------------------------------------
 python -m json.tool nodejs-helloworld/package.json --validate json file
 
 -------------------------------------------------------------------------
 oc start-build --follow bc/${APP_NAME}
 
 -----------------------------------------------------------------------------
 oc create configmap config_map_name  --from-literal key1=value1  --from-literal key2=value2
 
 oc create configmap config_map_name --from-file /home/demo/conf.tx
 
 ----------------------------------------------------------------------------
 oc create secret generic secret_name --from-literal username=user1 --from-literal password=mypa55w0rd
 
 oc create secret generic secret_name --from-file /home/demo/mysecret.tx
 
 ----------------------------------------------------------------------------
 oc edit configmap/myconf
 
 oc get secret/mysecret -o json
 
 oc edit secret/mysecret
 
 --------------------------------------------------------------------------------
 oc set env dc/mydcname --from configmap/myconf
 
 oc set volume dc/mydcname --add  -t configmap -m /path/to/mount/volume --name myvol --configmap-name myconf
   
 -----------------------------------------------------------------------------------
  oc set env dc/mydcname  --from secret/mysecre
  
  oc set volume dc/mydcname --add  -t secret -m /path/to/mount/volume  --name myvol --secret-name mysecre
  
 ----------------------------------------------------------------------------------
   oc rollout latest mydcname -  trigger a new deployment of the application
   
 -----------------------------------------------------------------------------
   
   podman login -u ${RHT_OCP4_QUAY_USER} quay.io 
   
 --------------------------------------------------------------------------
 skopeo copy  oci:/home/student/DO288/labs/external-registry/ubi-sleep docker://quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0
 
 --------------------------------------------------------------------------------
  podman search quay.io/ubi-sleep 
  
 -----------------------------------------------------------------------
   skopeo inspect docker://quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0 
   
 -------------------------------------------------------------------------
  sudo podman run -d --name sleep quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0 
  
  
 ------------------------------------------------------------------------------------
 sudo podman ps
 sudo podman logs sleep 
 sudo podman stop sleep
 sudo podman rm sleep 
 
 
 -------------------------------------------------------------------------------
  oc create secret generic quayio  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json  --type kubernetes.io/dockerconfigjson 
  
 ------------------------------------------------------------------
  oc secrets link default quayio --for pull 
  
 -------------------------------------------------------------------
 skopeo delete docker://quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0

---------------------------------------------------------------------
Verify that there is a route in the openshift-image-registry project
> oc get route -n openshift-image-registry 

---------------------------------------------------------------------
Save the host name of the internal registry's route into a shell variable
>  INTERNAL_REGISTRY=$( oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}' )

--------------------------------------------------------------------
Retrieve your developer user account's OpenShift authentication token
> TOKEN=$(oc whoami -t)

---------------------------------------------------------------------
Copy the OCI image to the classroom cluster's internal registry 
> skopeo copy --dest-creds=${RHT_OCP4_DEV_USER}:${TOKEN} oci:/home/student/DO288/labs/expose-registry/ubi-info docker://${INTERNAL_REGISTRY}/${RHT_OCP4_DEV_USER}-common/ubi-info:1.0 
-----------------------------------------------------------------------
Verify that an image stream was created to manage the new container image
> oc get is
------------------------------------------------------------------------
Create a local container from the image in the OpenShift internal registry

1 Log in to the OpenShift internal registry using the authentication token
> sudo podman login -u ${RHT_OCP4_DEV_USER} -p ${TOKEN} ${INTERNAL_REGISTRY} 

2  Download the ubi-info:1.0 container image into the local container engine
>  sudo podman pull ${INTERNAL_REGISTRY}/${RHT_OCP4_DEV_USER}-common/ubi-info:1.0 

3 Start a new container from the ubi-info:1.0 container image
>  sudo podman run --name info ${INTERNAL_REGISTRY}/${RHT_OCP4_DEV_USER}-common/ubi-info:1.0 

4 Clean up. Delete all resources created 
> oc delete is ubi-info 
> oc delete project ${RHT_OCP4_DEV_USER}-common 
>  sudo podman rm info 
> sudo podman rmi -f ${INTERNAL_REGISTRY}/${RHT_OCP4_DEV_USER}-common/ubi-info:1.0 

---------------------------------------------------------------------------
Verify a image
>  skopeo inspect docker://quay.io/redhattraining/hello-world-nginx 

---------------------------------------------------------------------------
Create the hello-world image stream that points to the redhattraining/ hello-world-nginx container image from Quay.io
> oc import-image hello-world --confirm --from quay.io/redhattraining/hello-world-nginx 
---------------------------------------------------------------------------------------
Grante service account
>  oc policy add-role-to-group -n ${RHT_OCP4_DEV_USER}-common system:image-puller  system:serviceaccounts:${RHT_OCP4_DEV_USER}-expose-image 
---------------------------------------------------------------------------------------
listar bc env
> oc set env bc simple --list 

-----------------------------------------------------------------------------------
setar env 
> oc set env bc simple npm_config_registry=http://${RHT_OCP4_NEXUS_SERVER}/repository/nodejs 

----------------------------------------------------------------------------------
Get the secret for the webhook by running the oc get bc command
> oc get bc simple -o jsonpath="{.spec.triggers[*].generic.secret}{'\n'}" 

----------------------------------------------------------------------------
Start a new build using the webhook URL, and the secret discovered from the output of the previous steps. The error message about 'invalid Content-Type on payload' can be safely ignored.
>  curl -X POST -k  ${RHT_OCP4_MASTER_API}/apis/build.openshift.io/v1/namespaces/${RHT_OCP4_DEV_USER}-build-app/buildconfigs/simple/webhooks/4R8kYYf3014kCSPcECmn/generic 

-----------------------------------------------------------------------------
Explore the S2I scripts packaged in the rhscl/httpd-24-rhel7 builder image
> sudo podman run --name test -it rhscl/httpd-24-rhel7 bash 
-----------------------------------------------------------------------------
Inspect the S2I scripts packaged in the builder image. The S2I scripts are located in the /usr/libexec/s2i folder:
> cat /usr/libexec/s2i/assemble 
> cat /usr/libexec/s2i/run 
>  cat /usr/libexec/s2i/usage 
------------------------------------------------------------------------------
Use the s2i command to create the template files and directories needed for the S2I builder image.
> s2i create s2i-do288-httpd s2i-do288-httpd

-------------------------------------------------------------------------------
Verify that the template files have been created
> tree -a s2i-do288-httpd
-----------------------------------------------------------------------------
Create the Apache HTTP server S2I builder image
> sudo podman build -t s2i-do288-httpd .

------------------------------------------------------------------------------
Build and test an application container image that combines the Apache HTTP server S2I builder image and the application source code.
1 Create a new directory 
  >  mkdir /home/student/s2i-sample-app

2 Build the application container image:
  > s2i build test/test-app/  s2i-do288-httpd s2i-sample-app --as-dockerfile ~/s2i-sample-app/Dockerfile 
  > cd ~/s2i-sample-app
----------------------------------------------------------------------------------
Build a test container image from the generated Dockerfile
> sudo podman build --format docker -t s2i-sample-app . 

---------------------------------------------------------------------------------
Test the application container image locally
> sudo podman run --name test -u 1234  -p 8080:8080 -d s2i-sample-app 

--------------------------------------------------------------------------------
Push the Apache HTTP server S2I builder image to your Quay.io account
>  sudo podman login  -u ${RHT_OCP4_QUAY_USER} quay.io 
--------------------------------------------------------------------------------
Use the skopeo copy command to publish the S2I builder image to your Quay.io account.
>  sudo skopeo copy  containers-storage:localhost/s2i-do288-httpd docker://quay.io/${RHT_OCP4_QUAY_USER}/s2i-do288-httpd 

--------------------------------------------------------------------------------
Create an image stream for the Apache HTTP Server S2I builder image.
 1 Log in to OpenShift and create a new project
   > oc login -u ${RHT_OCP4_DEV_USER} \ > -p ${RHT_OCP4_DEV_PASSWORD} ${RHT_OCP4_MASTER_API}
  
 2 Log in to your personal Quay.io account using Podman, this time without the sudo command, so that you can export the auth.json file to use it in a pull secret in the next step. You need to enter your password again.
   > podman login \ > -u ${RHT_OCP4_QUAY_USER} quay.io
 
 3 Create a secret from the container registry API access token that was stored by Podman
   > oc create secret generic quayio  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json  --type=kubernetes.io/dockerconfigjson 
   
 4 Link the new secret to the builder service account
   > oc secrets link builder quayio
 
 5 Create an image stream by importing the S2I builder image from your Quay.io container registry
   > oc import-image s2i-do288-httpd  --from quay.io/${RHT_OCP4_QUAY_USER}/s2i-do288-httpd --confirm 
   
------------------------------------------------------------------------------------------
Deploy and test the html-helloworld application from the classroom Git repository on an OpenShift cluster. This application consists of a single HTML file that displays a message
 1 Create a new application called hello from sources in Git
   > oc new-app --as-deployment-config  --name hello-s2i  s2i-do288-httpd~https://github.com/${RHT_OCP4_GITHUB_USER}/DO288-apps  --context-dir=html-helloworld 
   
-----------------------------------------------------------------------------------------
Customize the run script for the s2i-do288-go builder image in the application source. Change how the application is started by adding a --lang es argument at startup. This changes the default language used by the application.
 >  mkdir -p ~/DO288-apps/go-hello/.s2i/bin
 
---------------------------------------------------------------------------------------
Templates
Create the template definition file. Start with a basic template definition file and then export the required resources one by one, clean out the runtime information, and then copy the cleaned resource configuration to the template.
1 - create a yaml file 
 > 
2 - Export the resources in the project one by one, starting with the image stream resources.
 > oc get -o yaml --export is > /tmp/is.yaml
3 -  Make a copy of the exported file and clean the runtime attributes from it
 >  cp /tmp/is.yaml /tmp/is-clean.yaml
-----------------------------------------------------------------------------------------
oc new-app --as-deployment-config --name xxxx --file

-----------------------------------------------------------------------------------------
Probes
oc set probe dc/probes --liveness  --get-url=http://:8080/healthz --initial-delay-seconds=2 --timeout-seconds=2

oc set probe dc/probes --readiness  --get-url=http://:8080/ready  --initial-delay-seconds=2 --timeout-seconds=2 

----------------------------------------------------------------------------------------
get strategy
 oc get dc/mysql -o jsonpath='{.spec.strategy.type}{"\n"}
 -----------------------------------------------------------------------------------------
 remove triggers
 oc set triggers dc/mysql --from-config --remove 
 
 ------------------------------------------------------------------------------------------
  Change the default deployment strategy
  oc patch dc/mysql --patch  '{"spec":{"strategy":{"type":"Recreate"}}}'
------------------------------------------------------------------------------------------
Remove the rollingParams attribute from the deployment configuration
oc patch dc/mysql --type=json  -p='[{"op":"remove", "path": "/spec/strategy/rollingParams"}]'

-------------------------------------------------------------------------------------------
oc describe dc/mysql | grep -A 3 'Strategy:' 


---------------------------------------------------------------------------------------------
Force a new deployment to test changes to the strategy and the new post life-cycle hook
oc rollout latest dc/mysql

---------------------------------------------------------------------------------------------
Set the HOOK_RETRIES environment variable in the deployment configuration to a value of 5.
 oc set env dc/mysql HOOK_RETRIES=5 
----------------------------------------------------------------------------------------------
 oc rollout latest dc/mysql 
 
 ---------------------------------------------------------------------------------------------
 log into a remove pod
 oc rsh mysql-3-3p4m1 
 --------------------------------------------------------------------------------------------
 log to mysql
  mysql -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE 

-----------------------------------------------------------------------------------------------
CLI
To start a deployment, use the oc rollout command
> oc rollout latest dc/name
----------------------------------------------------------------------------------------------
To view the history of deployments for a specific deployment configuration, use the oc rollout history command:
> oc rollout history dc/name
----------------------------------------------------------------------------------------------
To access details about a specific deployment, append the --revision parameter to the oc rollout history command:
> oc rollout history dc/name --revision=1
---------------------------------------------------------------------------------------------
To access details about a deployment configuration and its latest revision, use the oc describe dc command:
>  oc rollout cancel dc/name
-----------------------------------------------------------------------------------------------
To retry a deployment that failed, run the oc rollout command with the retry option:
> oc rollout retry dc/name

When you retry a deployment, it restarts the deployment process but does not create a new deployment revision.
-------------------------------------------------------------------------------------------------
To use a previous version of the application you can roll back the deployment with the oc rollback command:
> oc rollback dc/name
-------------------------------------------------------------------------------------------------
 you can revert to a previous known working version of the application using the oc rollback command.
 > 
 
 -----------------------------------------------------------------------------------------------
 To prevent accidentally starting a new deployment process after a rollback is complete, image change triggers are disabled as part of the rollback process
> oc set triggers dc/name --auto

----------------------------------------------------------------------
You can also view logs from older failed deployment processes, provided they have not been pruned or deleted manually
> oc logs --version=1 dc/name
----------------------------------------------------------------------
You can scale the number of pods in a deployment using the oc scale command:
> oc scale dc/name --replicas=3
---------------------------------------------------------------------
Use the oc set triggers command to set a deployment trigger for a deployment configuration. For example, to set the ImageChange trigger, run the following command:
> oc set triggers dc/name --from-image=myproject/origin-ruby-sample:latest -c helloworld

---------------------------------------------------------------------
