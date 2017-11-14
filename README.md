# Jenkins-in-the-Cloud
Quickly and easily get your Jenkins build agents running in the cloud. Let Azure worry about the infrastructure, enabling your apps and services to scale and grow. Decrease your costs, spend less time managing your Jenkins and reduce your build times.## Time to Complete: 
15 minutes


## Scenario: 
Use Azure Container agent to add on-demand capacity and use Azure Container Instances (ACI) to build the Spring PetClinic Sample Applicaiton. 


If do have not select this link: https://tutorials.visualstudio.com/jenkins-azure/new/sign-in.html


# Setup Jenkins Master


## Create the Jenkins VM from Solution template
Open the marketplace image for Jenkins in your web browser and select GET IT NOW from the left-hand side of the page. Review the pricing details and select Continue, then select Create to configure the Jenkins server in the Azure portal.

<!--- ![azurejenkinscreate](./images/azurejenkinscreate.PNG) --->

In the Configure basic settings tab, fill in the form:
* Use Jenkins for Name.
* Enter a User name. The user name must meet specific requirements.
* Select Password as the Authentication type and enter a password. The password must have an upper case character, a number, and one special character.
* Use myJenkinsResourceGroup for the Resource Group.
* Choose the East US Azure region from the Location drop-down.
* Select OK to proceed to the Configure additional options tab. Enter a unique domain name to identify the Jenkins server and select OK.


Once validation passes, select OK again from the Summary tab. Finally, select Purchase to create the Jenkins VM. When your server is ready, you get a notification in the Azure portal.


Navigate to your virtual machine (for example, http://jenkins2517454.eastus.cloudapp.azure.com/) in your web browser. The Jenkins console is inaccessible through unsecured HTTP so instructions are provided on the page to access the Jenkins console securely from your computer using an SSH tunnel.


Set up the tunnel using the ssh command on the page from the command line, replacing username with the name of the virtual machine admin user chosen earlier when setting up the virtual machine from the solution template.

~~~sh
ssh -L 127.0.0.1:8080:localhost:8080 <username>@<public DNS name of instance you just created>
~~~	

After you have started the tunnel, navigate to http://localhost:8080/ on your local machine.


Get the initial password by running the following command in the command line while connected through SSH to the Jenkins VM.

~~~sh 
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

Unlock the Jenkins dashboard for the first time using this initial password.


Select Install suggested plugins on the next page and then create a Jenkins admin user used to access the Jenkins dashboard.


>The Jenkins server is now ready to build code.


## Create your first job
Select Create new jobs from the Jenkins console, then name it mySampleApp and select Freestyle project, then select OK.


Select the Source Code Management tab, enable Git, and enter the following URL in Repository URL field: https://github.com/spring-guides/gs-spring-boot.git


Select the Build tab, then select Add build step, Invoke Gradle script. Select Use Gradle Wrapper, then enter complete in Wrapper location and build for Tasks.


Select Advanced and then enter complete in the Root Build script field. Select Save.


## Build the code
Select Build Now to compile the code and package the sample app. When your build completes, select the Workspace link for the project.


Navigate to complete/build/libs and ensure the gs-spring-boot-0.1.0.jar is there to verify that your build was successful. Your Jenkins server is now ready to build your own projects in Azure.


## Update Jenkins DNS
Make sure you update the Jenkins DNS name in Managed Jenkins -> Configure System -> Administrative monitors configuration -> Jenkins URL. Otherwise, the agent won’t be able to connect with the master.


## Allow JNLP
Since the slave/agent connects with master via JNLP, make sure JNLP is allowed. In Jenkins, under Configure Global Security -> TCP port for JNLP agents, select Fixed and specify a port, for example, 12345. 


Make sure you add a corresponding inbound security rule for the Jenkins master. In Azure, you add the rule in the Network Security Group for the Jenkins master.


## Install the Azure plugins
Since you deployed Jenkins on Azure using the solution template, the Azure Credential Plugin and Azure Container Agents are already installed.


---


## Create an Azure service principal
You need an Azure service principal to create an Azure Container Instances in Azure. You can think of Azure service principal as a ‘user identity’ (login and password or certificate) with a specific role, and tightly controlled permissions to access your resources in Azure.


>If you don’t already have an Azure service principal, use Cloud Shell, an interactive, browser-accessible shell for managing Azure resources, to create one. Go to Azure Portal and launch Cloud Shell from the top navigation. 

## Set up job


Follow the instructions to select the _environment_ and subscription to use.

In Cloud Shell, run:
~~~bash
    az ad sp create-for-rbac --name "uuuuuuuu" --password "pppppppp"
~~~


Where `uuuuuuuu` is the user name and `pppppppp` is the password for the service principal.


Azure responds with JSON that resembles the following examples:

~~~json
    {
        "appId": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
        "displayName": "uuuuuuuuuu",
        "name": "http://uuuuuuuu",
        "password": "pppppppp",
        "tenant": "tttttttt-tttt-tttt-tttt-tttttttttttt"
    }
~~~

>You will use the values from this JSON response when you add the Azure Service Principal to Jenkins using the Azure Credential plugin. The aaaaaaaa, uuuuuuuu, pppppppp, and tttttttt are placeholder values.


## Add Azure service principal to Jenkins
Next, you need to add the Azure service principal to Jenkins credential store using the Azure credential plugin.


From the Jenkins dashboard, select Credentials, Select System and then Add Credentials. In the Add Credentials dialog, select Microsoft Azure Service -Principal from the Kind drop-down.

If you don’t know your Azure subscription ID, you can query it from the Cloud Shell:

~~~bash
    az account list
~~~
Enter your subscription ID. Use the appId value for Client ID, password for Client Secret, and a URL for OAuth 2.0 Token Endpoint of https://login.windows.net/<tenant_value>.


Provide mySP as the ID for this crediatial and enter Azure service principal for description. 
Set up credential


Verify the service principal authenticates with Azure by clicking Verify Service Principal


---


# Create a resource group for Azure Container Instances
Azure Container Instances (ACI) makes it easy for you to get up and running without having to provision virtual machines or adopt a higher-level service. ACI provides per second billing based on the capacity you need; making it a lucrative option for transient workload like spinning up a container agent to run a build, push the build artifacts to say Azure Storage and then shut the agent down when you no longer need it.


Azure Containter Instances must be placed in an Azure resource group. An Azure resource group is a container that holds related resources for an Azure solution.


In Cloud Shell, use the following command to create a resource group called myJenkinsAgentGroup in eastus:

~~~bash
    az group create --name myJenkinsAgentGroup --location eastus
~~~


# Configure the Azure Container Agents plugin
From the Jenkins dashboard, select Manage Jenkins, then Configure System.
Scroll to the bottom of the page and find the Cloud section with the Add new cloud dropdown and choose Azure Container Instance. 
* Add new cloud
* Select mySP(Azure service principal) from the Azure Service Principal dropdown.
* In the Resource Group Name section, select myJenkinsAgentGroup.
* Under Aci container Template, enter ACI-container for both Name and Labels.
* Enter cloudbees/jnlp-slave-with-java-build-tools for Docker Image. 
* Configure ACI Agent link Advanced to expand advance settings and update Retention Strategy to Container Idle Retention Strategy to keep the agent up for until no new job is executed on the agent and the idle time specified has elapsed.
* Select Save to update the plugin configuration.


---


# Create a job in Jenkins
Create a Freestyle project in Jenkins to build the Spring PetClinic Application.


* Within the Jenkins dashboard, click New Item.
* Enter demoproject1 for the name and select Freestyle project, then select OK.
* In the General tab, choose Restrict where project can be run and type ACI-agent in Label Expression. You see a message confirming that the label is served by the cloud configuration created in the previous step. Set up job
* In the Source Code Management tab, select Git and add the following URL into the Repository URL field: https://github.com/spring-projects/spring-petclinic.git
* In the Build tab, select Add build step, then Invoke top-level Maven targets. Enter package in the Goals field.
* Select Save to save the job definition.


---


#Build the new job on an Azure Container agent
Finally, it’s time to build the project you just created.


Go back to the Jenkins dashboard.
Select the job you created in the previous step, then click Build now. A new build is queued, but does not start until an ACI agent is created in your Azure subscription. A new ACI agent will appear on the Jenkins dashboard within seconds. ACI agent The job will start building in about 4 minutes after the agent is created. If you rerun the build, the job starts building immediately since the agent has already been created and is online.


>Once the build is complete, go to Console output. Click Full log to see that the build was performed remotely on an Azure agent. Console output

