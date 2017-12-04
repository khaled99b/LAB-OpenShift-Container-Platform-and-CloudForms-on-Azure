Agenda
======

[CHALLENGE -1: Deploy Openshift on azure](challenge1)

CHALLENGE -2: Create and manage projects 

CHALLENGE -3: Create and manage Applications

CHALLENGE -4: Configuring automated builds

CHALLENGE -5: Continuous deployment

CHALLENGE -6: Mini project: JBOSS EAP application

CHALLENGE -7: Monitoring oepnshift with azure oms

Objectives and pre-requisites
=============================

This document describes the steps necessary to deploy and manage Red Hat
OpenShift container platform, and CloudForms on Azure. The lab is based
on OpenShift version 3.5 and CloudForms version 4.1.

You will need an active Red Hat subscription and a valid Azure account
to perform the lab instructions.

-   Create your [free azure
    account](https://azure.microsoft.com/en-us/free/)
    (https://azure.microsoft.com/en-us/free/), today.

-   Request a free 30 days
    [evaluation](https://access.redhat.com/products/red-hat-openshift-container-platform/evaluation)
    from RedHat
    (<https://access.redhat.com/products/red-hat-openshift-container-platform/evaluation>),
    (https://access.redhat.com/solutions/411973)

-   You need to have an account on [GitHub](http://www.github.com/), If
    you don't have one create a free account (<https://github.com/>).

<aside class="notice">
[NB]: As an alternative to OpenShift enterprise, you can
deploy the community version OpenShift.org. And as an alternative to
CloudForms, you can deploy the community version ManageIQ. The community
edition represents a test bed incubator and upstream to the enterprise
one. All OpenShift container platform features are also available in
OpenShift.org.
</aside>

To perform the lab using OpenShift.org, follow the steps at
[https://github.com/Microsoft/openshift-origin](https://github.com/Microsoft/openshift-origin)
and skip Challenge-1.

<aside class="notice">
[NB]: If you want to use your corporate Azure account, request a resource group with owner access control from your Azure admin.

[NB]: Even though, you can bring your own Red Hat subscription for OpenShift, the deployment template in this lab will provision the master and compute nodes from RHEL images, pay as you go.
You will incur extra charges not covered by the Azure free tier. Amount varies depending on the number and size of the vm nodes.
</aside>

The Lab challenges cover:

-   Introduction to OpenShift3

-   Deployment of OpenShit on Azure

-   Creating and managing OpenShift projects on azure

-   Creating and managing OpenShift applications on Azure

-   Automating builds with Linux Containers on Azure.

-   Mini project: Jboss EAP application

-   Integration of OpenShift with Azure OMS

-   Installation and introduction to CloudForms

Introduction to openshift
=========================

[OpenShift](http://www.openshift.com) containers platform is Red Hat's Platform-as-a-Service (PaaS) on top of Kubernetes. It allows developers to quickly develop, host, and scale applications in a cloud environment.

Microsoft and Red Hat have signed a partnership that includes support to run Red Hat OpenShift on Microsoft Azure and Azure Stack.

OpenShift offers multiple access modes including: developer CLI, admin CLI, web console and IDE plugins. Click2cloud is a plugin that allows Visual studio to deploy code to OpenShift, directly.

![](./MediaFolder/media/image3.jpg)


###CHALLENGE-1: Deploy Openshift on azure
=========================================

OpenShift, leverages multiple Azure services such as VM *extensions*, Azure disks, and *Key vaults...* to provide an Enterprise grade offering for customers who would like to containerize and manage their applications, without investing long time and hard effort configuring and integrating various tools.

OpenShift offers another alternative to multiple CaaS (container as a service) solutions available on Azure, such as *Azure container service, Azure service fabric* and *Pivotal* from *CloudFoundry*...

![](./MediaFolder/media/image4.png)
 
OpenShift container platform is available as an Azure Resource Manager solution at https://github.com/Microsoft/openshift-container-platform.

1.  Login to Azure portal and start a new Bash Cloud shell session.

![](./MediaFolder/media/image5.JPG)

2.  From the open terminal, create a new ssh key pair with the name "osslab_rsa" and save it under .ssh directory

![](./MediaFolder/media/image7.JPG)

```bash   
$ ssh-keygen
``` 

3.  Use the Azure CLI v2 to create a new resource group to host the lab resources

![](./MediaFolder/media/image8.JPG)

```bash
$ az group create -n ossdemo -l 'West Europe'
```

4.  Create a Key Vault and add your *ssh* private key, created in the previous step.

![](./MediaFolder/media/image9.JPG)

```bash
$ az keyvault create -n ossKV -g ossdemo -l 'West Europe' --enabled-for-template-deployment true
```
![](./MediaFolder/media/image10.JPG)

```bash
$ az keyvault secret set --vault-name ossKV -n ossSecret --file ~/.ssh/osslab_rsa
```

The deployment of OpenShift relies on Ansible scripts that should be configured for Azure as the cloud provider. During and post-installation, OpenShift requires access to some Azure resources, like provisioning an Azure managed disk for persistence storage backend.
When you have an application that needs to access or modify resources, you must set up an Azure Active Directory (AD) application and assign the required permissions to it. The service principal object defines the
policy and permissions for an application's use in a specific tenant, providing the basis for a security principal to represent the application at run-time.

5.  Create an Azure Active Directory Service Principal and choose a different password. Copy the resource group scope from step number 3.

```bash
$ az ad sp create-for-rbac -n openshiftcloudprovider --password changeMePassword --role contributor --scopes subscriptions/f3a5dfdb-e863-40d9-b23c-752b886c0260/resourceGroups/ossdemo
```
![](./MediaFolder/media/image11.JPG)

![](./MediaFolder/media/image12.JPG)

6.   Now, go to the Azure portal and assign the required permissions to the service principal "osscloudprovider" on the resource group "ossdemo".

![](./MediaFolder/media/image13.JPG)

![](./MediaFolder/media/image14.JPG)

7.  Note the application id of your service principal.

```bash
$ az ad sp show --id http://openshiftcloudprovider
```

8.  Use the following Azure resource manager solutions
[template](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMicrosoft%2Fopenshift-container-platform%2Fmaster%2Fazuredeploy.json)
to deploy your OpenShift environment:
<https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMicrosoft%2Fopenshift-container-platform%2Fmaster%2Fazuredeploy.json>

-   At this stage we will need Red Hat Network or Red Hat satellite credentials to register RHEL vms with RedHat and get access to the required software channels during the deployment. More specifically,
we will need:

    -   Red Hat Network/Satellite name
    -   The associated password or activation key

-   PoolId provides access to the required software channels for Openshift and Cloudforms. PoolId can be listed by invoking the command 'subscription-manager list -available' from a registered RHEL system. Contact your Red Hat admin or Red Hat support if you are missing information.

-   For high availability consideration, we are deploying 3 vms for each type (master, infra and agent). If you want to deploy the lab with minimal cost, you can reduce the number of vms to one per each. Also, you can scale down the vm families, but preferably stick to vms with SSD disks for faster deployment.

-   The master and infra load balancer DNSs (in red circles) should be globally unique. Choose your own names.

![](./MediaFolder/media/image15.JPG)
![](./MediaFolder/media/image16.JPG)
![](./MediaFolder/media/image17.JPG)
![](./MediaFolder/media/image18.JPG)
![](./MediaFolder/media/image19.JPG)
![](./MediaFolder/media/image15.JPG)


9.  From the Azure portal, go to your resource group "ossdemo", track the progress of the deployment and make sure the deployment finishes, successfully. The process should last around 20 minutes. It is a good time to have a break.

![](./MediaFolder/media/image20.png)

![](./MediaFolder/media/image21.png)

The following diagram explains the physical architecture of the deployed cluster.

The bastion server implements mainly two distinct functions. One is that of a secure way to connect to all the nodes, and second that of the "installer" of the system. The bastion host is the only ingress point for SSH in the cluster from external entities. When connecting to the OpenShift Container Platform infrastructure, the bastion forwards the request to the appropriate server.

The next diagram, explains the role and tasks of the Openshift master/agents and the logical architecture of the solution.

![](./MediaFolder/media/image22.jpg)
 

CHALLENGE -2: Create and manage projects 
=========================================

There are many ways to launch images within an OpenShift project. In this challenge we will focus on the graphical portal.

To create an *application*, you must first create a new *project*, then select an *InstantApp* template. From there, OpenShift begins the build process, and creates a new deployment.

1.  Login to your *github* account, or create one if you didn't.

2.  Browse to *openshift/ruby-ex* repository and fork it into your *github* account

![](./MediaFolder/media/image23.PNG)
 

1.  From your browser, visit the OpenShift web console at *https://FQDN-master-node:8443*. The web site, uses a self-signed certificate, so if prompted, continue and ignore the browser warning.

2.  Log in using your username and password.

![](./MediaFolder/media/image24.JPG)

![](./MediaFolder/media/image25.JPG) 

3.  To create a new project, click New Project.

4.  Type a unique name, display name, and description for the new project.

5.  Click Create. The web console's welcome screen should start loading.

![](./MediaFolder/media/image26.JPG)

![](./MediaFolder/media/image27.JPG)

CHALLENGE -3: Create and manage Applications
============================================

The *Select Image* or *Template* page gives you the option to add an application to your project from a publicly accessible *Git* repository, or from a *template*:

1.  If creating a new project did not automatically redirect you to the *Select Image or Template page*, you might need to click Add to Project.

2.  Click Browse, then select

3.  Click the ruby:latest builder image.

![](./MediaFolder/media/image28.JPG)
 

4.  Type a name for your application, and specify the git repository URL you previously forked:
    <https://github.com/'your_github_username'/ruby-ex.git>.

![](./MediaFolder/media/image29.JPG)
 

5.  Optionally, click Show advanced routing, build, and deployment options. Explore the build configuration and other options and note that this example application automatically creates a route,*webhook* trigger, and builds change triggers. A *webhook* is an HTTP call-back triggered by a specific event.

6.  Click Create. Creating your application might take some time. note the *payload url,* we will use it later to set a *webhook* in your *github* repository.

7.  You can follow along on the Overview page of the web console to see the new resources being created, and watch the progress of the build and deployment. Click on "view log", you will notice that Openshift is pulling the code of the application from Github and building the layers of the container image that will host the application. Once the build is complete, Openshift will start a new pod.

![](./MediaFolder/media/image30.JPG)    

OpenShift leverages the Kubernetes concept of a pod, which is one or more containers deployed together on one host. A pod is the smallest compute unit that can be defined, deployed, and managed.

Pods are the rough equivalent of a machine instance (physical or virtual) to a container. Each pod is allocated its own internal IP address, therefore owning its entire port space. Containers within pods can share their local storage and networking.

While the Ruby *pod* is being created, its status is shown as pending.
The Ruby *pod* then starts up and displays its newly-assigned IP address. When the Ruby *pod* is running, the build is complete.

Pods have a lifecycle; they are defined, then they are assigned to run on a node, then they run until their container(s) exit or they are removed for some other reason. Pods, depending on policy and exit code, may be removed after exiting, or may be retained in order to enable access to the logs of their containers.

![](./MediaFolder/media/image31.JPG)

8. From the overview page, click the web address for the application in the upper right corner. Verify that the web application is up and available.

![](./MediaFolder/media/image32.JPG)

![](./MediaFolder/media/image33.JPG)
    
9.  Return to the *OpenShift* admin console. Browse to the project's overview page, and test scaling out and in your application by increasing or decreasing the number of *pods*, using the up and down arrow signs on the web console.
Scale out the app into 3 pods and watch the progress.

![](./MediaFolder/media/image34.JPG)
    
10.  Browse to Applications -> Pods, and make sure 3 pods serving the same application are now up and running.

![](./MediaFolder/media/image35.JPG)
    

CHALLENGE -4: Configuring automated builds
==========================================

Since we forked the source code of the application from the [OpenShift GitHub repository](https://github.com/openshift/ruby-ex), we can use a *webhook* to automatically trigger a rebuild of the application whenever code changes are pushed to the forked repository.

To set up a *webhook* for your application:

1.  From the Web Console, navigate to the project containing your application.

2.  Click the Browse tab, then click Builds.

3.  Click your build name, then click the Configuration tab.

4.  Click next to GitHub webhook URL to copy your *webhook* payload URL.

![](./MediaFolder/media/image36.JPG)
 
5.  Navigate to your forked repository on GitHub, then click Settings.

6.  Click Webhooks & Services and Click Add webhook.

![](./MediaFolder/media/image37.PNG)

7.  Paste your *webhook* URL into the Payload URL field.

8.  As Content Type choose application/json

9.  Disable SSL verification and click Add webhook to save.

GitHub will now attempt to send a ping payload to your *OpenShift* server to ensure that communication is successful. If you see a green check mark appear next to your *webhook* URL, then it is correctly configured.

Hover your mouse over the check mark to see the status of the last delivery.

![](./MediaFolder/media/image38.JPG)

Next time you push a code change to your forked repository, your application will automatically rebuild.

CHALLENGE -5: Continuous deployment 
====================================

In this section, we demonstrate one of the most powerful features of *OpenShift*. We will see how we can trigger a continuous deployment pipeline, just by committing code change to Github.

Once there is a code change, the Github *webhook* will trigger the build of a new container image that combines a blueprint image from the registry with the updated code and generate a new image. This feature is called *S2I*, or source to image. Once the build finishes, *OpenShift* will automatically deploy the new application based on the new image. This capability enables multiple deployment strategies such
as A/B testing, Rolling upgrades...

![](./MediaFolder/media/image39.png)

1.  Use Azure cloud shell or install *Git* into your local machine

PS: If you don't want to use *git*, you still can perform this challenge by moving to step-5 and editing the file *config.ru*, directly from the web interface of *github* and then go to step-7.
Create a "dev" folder and change into.

```bash
 $ mkdir dev && cd dev
```

2.  Clone the forked repository to your local system

```bash
 $ git clone https://github.com/<YourGithubUsername>/ruby-ex.git
```

3.  Make sure your local *git* repository is referencing to your
    *ruby-ex git*, on *github*:

```bash
 $ cd ruby-ex
```

```bash
 $ git remote -v
```

4.  On your local machine, use your preferred text editor to change the sample application's source for the file *config.ru*

Make a code change that will be visible from within your application.
For example: on line 229, change the title to "Welcome to your Ruby application on OpenShift POWERED BY AZURE!", then save your changes.

5.  Verify the working tree status

```bash
 $ git status
```

6.  Add config.ru content to the index, Commit the change in *git*, and push the change to your fork. You will need to authenticate with your *github* credentials


```bash
 $ git add config.ru
 $ git commit -m "simple message"  
 $ git status
 $ git push
```

7.  If your *webhook* is correctly configured, your application will immediately rebuild itself, based on your changes. Monitor the build from the graphical console. Once the rebuild is successful, view your updated application using the route that was created earlier.
Now going forward, all you need to do is push code updates and OpenShift handles the rest.

![](./MediaFolder/media/image40.JPG)

![](./MediaFolder/media/image41.JPG)

8.  From the web browser refresh the page and note the new change

![](./MediaFolder/media/image42.JPG)


9.  You may find it useful to manually rebuild an image if your *webhook* is not working, or if a build fails and you do not want to change the code before restarting the build. To manually rebuild the image based on your latest committed change to your forked repository:

    a.  Click the Browse tab, then click Builds.

    b.  Find your build, then click Start Build.


CHALLENGE -6: Mini project: JBOSS EAP application
=================================================

In this mini project, we will deploy a three tiers application consisting of Leaflet mapping front end with a JaxRS and MongoDB back end. The application, allows to visualize the locations of major National Parks and Historic Sites (https://github.com/OpenShiftDemos/restify-mongodb-parks).

During this challenge, we will leverage the CLI tool of OpenShift.

1.  Download and install the OpenShift CLI related to your operating system. The easiest way to download the CLI is by accessing the About page on the web console if your cluster administrator has enabled the download links. Another alternative is to ssh into your bastion host that has the CLI tool installed already.

![](./MediaFolder/media/image43.JPG)

If you don't have a valid Red Hat subscription, you could still download the oc tool from: https://github.com/CCI-MOC/moc-public/wiki/Installing-the-oc-CLI-tool-of-OpenShift

2.  Use your openshift url endpoint to login to your environment from the CLI

![](./MediaFolder/media/image44.JPG)    

3.  Create a new project "nationalparks"

![](./MediaFolder/media/image45.JPG)    

4.  From the web console, add a new Java application using the following git lab repository <https://gitlab.com/gshipley/nationalparks.git>

![](./MediaFolder/media/image46.JPG)    

![](./MediaFolder/media/image47.JPG)

5.  List builds operations:

![](./MediaFolder/media/image48.JPG)
    
6.  List existing projects, pods and view logs in real time:

![](./MediaFolder/media/image49.JPG)

![](./MediaFolder/media/image50.JPG)
    
![](./MediaFolder/media/image51.JPG)

Output truncated ....

![](./MediaFolder/media/image52.JPG)
    
7.  Browse the newly created application and verify it is available

![](./MediaFolder/media/image53.JPG)

![](./MediaFolder/media/image54.JPG)
    

8.  We can see the map but not the attraction points.The reason is that we only deployed the front-end application. What we will need now is to add a backend data base. From the web console, add a new persistent mongodb data store. And set the needed environment variables and specification as bellow:
![](./MediaFolder/media/image55.JPG)

![](./MediaFolder/media/image56.JPG)
    
![](./MediaFolder/media/image57.JPG)

9.  The graphical portal should show two applications in our project

![](./MediaFolder/media/image58.JPG)

10. From the left menu in the web console, click on storage and verify that OpenShift created a persistent storage volume using Azure storage.

![](./MediaFolder/media/image59.JPG)
    

11. Change the deployment configuration of the front-end application to include the environment variables required to access the database

![](./MediaFolder/media/image60.JPG)

12. verify the last modification took place by running 

```bash
$ oc get dc nationalparklocator -o json
```

![](./MediaFolder/media/image61.JPG)    

13. Back to the graphical console, note the automatic migration to a new pod based on the new configuration

![](./MediaFolder/media/image62.JPG)
    
14. Navigate to the application end-point and verify that the parks are now showing on the map

![](./MediaFolder/media/image63.JPG)
    
15. Our new application became very popular, and we need to scale out our front end to two pods. Use "oc scale" to do it

![](./MediaFolder/media/image64.JPG)
    
16. Now, let's test the self-healing, capabilities of OpenShift by deleting one of the running pods. Because, the desired state of the replication controller is 2 pods for the application "nationalparklocator", OpenShift will automatically and instantly trigger the deployment of a new pod.
    
![](./MediaFolder/media/image65.JPG)
    

CHALLENGE -7: Monitoring oepnshift with azure oms
=================================================

Azure Operations Management Suite (OMS) provides native support to OpenShift. In this challenge, we will walk through the steps of configuring OpenShift to export monitoring metrics directly to OMS.

1.  From the Azure portal create a new OMS workspace

![](./MediaFolder/media/image66.JPG)
    
2.  Open the OMS portal and note the workspace id and one of the primary keys.

![](./MediaFolder/media/image67.JPG)

![](./MediaFolder/media/image68.JPG)
    
3.  From Cloud Shell ssh into the bastion host, then ssh into one of the master node and create an OpenShift project and user account for OMS.

```bash
 [ossadmin@oss-bastion ~]$ ssh oss-master-0
 [ossadmin@oss-master-0 ~]$ oadm new-project omslogging --node-selector='zone=default'
 [ossadmin@oss-master-0 ~]$ oc project omslogging
 [ossadmin@oss-master-0 ~]$ oc create serviceaccount omsagent
 [ossadmin@oss-master-0 ~]$ oadm policy add-cluster-role-to-user cluster-reader system:serviceaccount:omslogging:omsagent
 [ossadmin@oss-master-0 ~]$ oadm policy add-scc-to-user privileged system:serviceaccount:omslogging:omsagent
```

4.  Use wget to download the ocp-* files from

<https://github.com/Microsoft/OMS-docker/tree/master/OpenShift>

5.  Make the file "secretgen.sh" executable. Run it, and provide our workspace id and key.

```bash
 [ossadmin@oss-master-0 ~]$ chmod +x secretgen.sh
 [ossadmin@oss-master-0 ~]$ ./secretgen.sh
```

6.  Create an OMS daemon set. A DaemonSet ensures that all the openshift cluster nodes run a copy of the oms pod.

```bash
 [ossadmin@oss-master-0 ~]$ oc create -f ocp-secret.yaml
```

7.  Validate that the daemon set is working properly

![](./MediaFolder/media/image69.JPG)

8.  Back to OMS portal, you will find that there are new data sources exporting metrics.

![](./MediaFolder/media/image70.JPG)
    
9.  Create your custom dashboard and start exploring the data exported by OpenShift under different visualization formats

![](./MediaFolder/media/image71.JPG)
    
![](./MediaFolder/media/image72.JPG)    

![](./MediaFolder/media/image73.JPG)

![](./MediaFolder/media/image74.JPG)


CHALLENGE -8: Red Hat Cloud Forms on Azure
==========================================

Red Hat Cloud Forms is an open source cloud management solution providing a unified and consistent set of management capabilities across:

-   Virtualization platforms like Red Hat Virtualization, VMware vCenter, and Microsoft Hyper-V.

-   Private cloud platforms based on OpenStack.

-   Public cloud platforms like Microsoft Azure and Amazon Web Services.

-   Red Hat OpenShift

CloudForms can see and manage both the guest and host systems, allowing management of workloads and infrastructure within the same system.

CloudForms is a first-class citizen on Azure and provides a solid hybrid cloud management solution. During this challenge we will walk through the installation and configuration of CloudFroms on Azure and the integration with the OpenShift environment created in the previous steps. In this section we will leverage another powerful open source SDK to perform the steps: Powershell.

1.  Login to Red Hat network portal and download the latest cloudforms vhd image for Azure. You will need an active Red Hat subscription.(https://www.redhat.com/en/technologies/management/cloudforms)

2.  We will need to round up the size of the vhd image to a multiple number of MB, otherwise the provisioning will fail in the next steps. We can use powershell to perform this operation

```powershell
 > Resize-VHD -Path $LocalImagePath -SizeBytes 32770MB
```

3.  Login to Azure and Configure some variables to use during the deployment. Change $BlobNameSource and $LocalImagePath to reflect your environment.
![](./MediaFolder/media/image75.JPG)
    
4.  Create a new storage account "osscfme" in the "ossdemo" resource group to deploy the vhd image to.

![](./MediaFolder/media/image76.JPG)
    
5.  Upload the vhd image into the newly created storage account. This operation will take 15 minutes. Time for a break!

```powershell
> Add-AzureRmVhd -ResourceGroupName $ResourceGroupName -Destination
https://$StorageAccountName.blob.core.windows.net/$BlobSourceContainer/$BlobNameSource -LocalFilePath $LocalImagePath -NumberOfUploaderThreads 8
```

![](./MediaFolder/media/image77.JPG)

.....

![](./MediaFolder/media/image78.JPG)

6.  Customize the Azure environment and create the CloudForms vm. Use your public key

```powershell
$BlobNameDest = "cfme-azure-5.8.1.5-1.x86_64.vhd"
$BlobDestinationContainer = "vhds"
$VMName = "cfme-5.8"
$DeploySize= "Standard_DS4_V2"
$vmUserName = "ossadmin"
$InterfaceName = "cfmenic"
$VNetName = "openshiftvnet"
$PublicIPName = "cfme-public-ip"
$SSHKey = " ssh-rsa
AAAAB3NzaC1yc2EAAAADAQABAAABAQDQOd6tNwPPYBQ+wveI+dmBmdpmaBB87qOG/dHe/ZEFJIXLDRILytVh2kevgeXn/SzbqL3DJ4qQWVGmsaZwELhlQKdcc/AybwRn9tQ94H2WgPQ79RnRd8BRgdu5sVWTpyGZc5OFdAyFvkIftiasHAg3jb7u6oJ7f3HCH4tax/sbdqhSkTnivH4Uxq0Vx1DQOt1z4WfbCYxmG9cLc2zNzqc6/d7Y/g33iW94FZ5CCPfUoY+HdyOSu5cy/rWtreCskWQZwNR8xkvDOKIlc2bnBTCosN79FMzZYyiOcvMIJbtE9KYH9G49G6p2tXDByhOQVj/4aIgRc2S5SW+l6e7VSn7l
khaled@khbadri"
$StorageAccount = Get-AzureRmStorageAccount -ResourceGroup
$ResourceGroupName -Name $StorageAccountName
$SourceImageUri = "https://$StorageAccountName.blob.core.windows.net/templates/$BlobNameSource"
$Location = $StorageAccount.Location
$OSDiskName = $VMName

# Network

$PIp = New-AzureRmPublicIpAddress -Name $PublicIPName -ResourceGroupName $ResourceGroupName -Location $Location
-AllocationMethod Dynamic -Force
$VNet = Get-AzureRmVirtualNetwork -Name openshiftVnet -ResourceGroupName ossdemo 
$Interface = New-AzureRmNetworkInterface -Name $InterfaceName -ResourceGroupName $ResourceGroupName -Location $Location -SubnetId
$VNet.Subnets[1].Id -PublicIpAddressId $PIp.Id -Force

# Specify the VM Name and Size

$VirtualMachine = New-AzureRmVMConfig -VMName $VMName -VMSize
$DeploySize

# Add User

$cred = Get-Credential -UserName $VMUserName -Message "Setting user
credential - use blank password"

$VirtualMachine = Set-AzureRmVMOperatingSystem -VM $VirtualMachine -Linux -ComputerName $VMName -Credential $cred

# Add NIC

$VirtualMachine = Add-AzureRmVMNetworkInterface -VM $VirtualMachine -Id $Interface.Id

# Add Disk

$OSDiskUri = $StorageAccount.PrimaryEndpoints.Blob.ToString() +
$BlobDestinationContainer + "/" + $BlobNameDest

$VirtualMachine = Set-AzureRmVMOSDisk -VM $VirtualMachine -Name
$OSDiskName -VhdUri $OSDiskUri -CreateOption fromImage -SourceImageUri
$SourceImageUri -Linux

# Set SSH key

Add-AzureRmVMSshPublicKey -VM $VirtualMachine -Path
"/home/$VMUserName/.ssh/authorized_keys" -KeyData $SSHKey

# Create the VM

New-AzureRmVM -ResourceGroupName $ResourceGroupName -Location
$Location -VM $VirtualMachine
```

![](./MediaFolder/media/image79.JPG)
![](./MediaFolder/media/image80.JPG)
![](./MediaFolder/media/image81.JPG)
![](./MediaFolder/media/image82.JPG)
![](./MediaFolder/media/image83.JPG)


7.  Navigate into the "cfme" vm from the Azure Portal, and note the IP address.

![](./MediaFolder/media/image84.JPG)    

8.  From the Cloud Shell, ssh into the virtual appliance using the SSH key.

![](./MediaFolder/media/image85.JPG)
    
9.  Switch to root "sudo -s" and enter "appliance_console" command. The Red Hat CloudForms appliance summary screen displays.

![](./MediaFolder/media/image86.JPG)
    

10.  Press Enter to manually configure settings.

![](./MediaFolder/media/image87.JPG)
    

11.  Press the number for the item you want to change, and press Enter. The options for your selection are displayed. For example configure the time zone (2).

12.  Follow the prompts to make the changes.

13.  Press Enter to accept a setting where applicable and quit (20)

14.  Back to the Azure portal. We need to add a new disk for the CloudForms data base.

![](./MediaFolder/media/image88.JPG)
![](./MediaFolder/media/image89.JPG)
![](./MediaFolder/media/image90.JPG)

15.  Use the command fdisk to verify the newly added disk

![](./MediaFolder/media/image91.JPG)
    
16. Run "appliance_console" again, hit enter and choose (5) to configure the database

17. Choose (1) to create a key

![](./MediaFolder/media/image92.JPG)
    
18. Choose (1) to create internal database for the database location.

![](./MediaFolder/media/image93.JPG)
    
19. Choose (1) for the disk we attached previously

![](./MediaFolder/media/image94.JPG)

20. Select Y to configure the appliance as a database-only appliance and create and confirm a password for the database. As a result, the appliance is configured as a basic PostgreSQL server, without a user interface

![](./MediaFolder/media/image95.JPG)
    
21. Choose (16) to start EVM processes.

22. Once Red Hat CloudForms is installed, you can log in and perform administration tasks.

![](./MediaFolder/media/image96.JPG)

23. Log in to Red Hat CloudForms for the first time after installing:

-   Navigate to the URL for the login screen. (https://xx.xx.xx.xx on the virtual machine instance)

-   Enter the default credentials (Username: admin | Password: smartvm) for the initial login.

-   Click Login.

-   Navigate to the URL for the login screen. (https://xx.xx.xx.xx on the virtual machine instance)

-   Click Update Password beneath the Username and Password text fields.

-   Enter your current Username and Password in the text fields.

-   Input a new password in the New Password field.

-   Repeat your new password in the Verify Password field.

-   Click Login.

24.  Add an Azure Cloud Provider:

-   Navigate to Compute → Clouds → Providers.

-   Click (Configuration), then click (Add a New Cloud Provider).

-   Enter a Name for the provider.

-   From the Type list, select Azure.

-   Select a region from the Region list. One provider will be created for the selected region.

-   Enter Tenant ID.

-   Enter Subscription ID.

-   Enter Zone.

-   In the Credentials section, enter the Client ID and Client Key; click Validate.

-   Click Add.

![](./MediaFolder/media/image97.JPG)

![](./MediaFolder/media/image98.JPG)

25.  Add OpenShift provider

-   Navigate to Compute → Containers → Providers.

-   Click (Configuration), then click (Add Existing Containers Provider).

-   Enter a Name for the provider.

-   From the Type list, select OpenShift Container Platform.

-   Enter the appropriate Zone for the provider. If you do not specify a zone, it is set to default.

-   Under Endpoints in the Default tab, configure the following for the OpenShift provider:

    -   Select a Security Protocol method to specify how to authenticate the provider: Choose SSL without validation

    -   Enter the Hostname or IPv4 of your OpenShift environment and keep default port

    -   Run the following to obtain the token needed to add an OpenShift Container Platform
    
    ![](./MediaFolder/media/image99.JPG)
        
    -   Enter the OpenShift management token in the Token field.

    -   Enter the same token in the Confirm Token field.

    -   Click Validate to confirm that Red Hat CloudForms can connect to the OpenShift Container Platform provider.
    
    ![](./MediaFolder/media/image100.JPG)
        
26.  Explore CloudForms by navigating the left menu to get an idea on the insights and intelligence provided by the solution. CloudForms provides additional modules (compliance, management, reporting...) that won't be covered by the scope of the lab.

![](./MediaFolder/media/image101.JPG)
![](./MediaFolder/media/image98.JPG)
![](./MediaFolder/media/image102.JPG)
![](./MediaFolder/media/image103.JPG)
![](./MediaFolder/media/image104.JPG)
![](./MediaFolder/media/image105.JPG)

End the lab
===========

To end the lab, simply delete the resource group *ossdemo* from the Azure portal or from the Azure CLI. And delete the created *webhook* from your *git* repository.

```bash
 $ az group delete ossdemo
```

References
--------------

### Useful links

<https://access.redhat.com/documentation/en-us/reference_architectures/2017/pdf/deploying_red_hat_openshift_container_platform_3.5_on_microsoft_azure/Reference_Architectures-2017-Deploying_Red_Hat_OpenShift_Container_Platform_3.5_on_Microsoft_Azure-en-US.pdf>

<https://blogs.technet.microsoft.com/msoms/2017/08/04/container-monitoring-solution-in-red-hat-openshift/>

<https://testdrive.azure.com/#/test-drive/redhat.openshift-test-drive>

<https://github.com/Microsoft/openshift-container-platform>

<https://github.com/Microsoft/openshift-origin>

<https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.5/html/installing_red_hat_cloudforms_on_microsoft_azure/>

<http://manageiq.org/>

### Redhat and Microsoft partnership

<http://openness.microsoft.com/2016/04/15/microsoft-red-hat-partnership-accelerating-partner-opportunities/>

<https://www.redhat.com/en/microsoft>
