# Steps for Demoing the Dynatrace plugin with Event Driven Ansible 

## This repo can be used to demo Event Driven Ansible with the Dynatrace plugin. This demo features:
1. Event Driven Ansible Controller
2. Ansible Automation Controller
3. Dynatrace
4. ServiceNow
### Use case:
a. Nginx  web server goes down on server.  

b. Process monitor via Dynatrace OneAgent detects failed Nginx  process creates problem alert in Dynatrace  

c. Failed event is sent to Event Driven Ansible Controller  

d. Event Driven Ansible runs Job template that does the following:  

1. Opens ServiceNow incident ticket
2. Attempts to restart Nginx 
3. Marks Incident ticket in progress
4. Closes ticket only if Nginx  is back up
![Alt text](<Screenshot from 2023-10-05 20-20-29.png>)

---
### Prerequisites:
> 1. Access to Dynatrace
> 2. Create an access token in Dynatrace to be used by the Dynatrace plugin in your rulebook.Note you will need the following scope permissions for the token:
>> #### API V1 Scope:
> - Access problem and event feed, metrics, and topology
> - Read configuration
> - Write configuration
>> #### API V2 Scope:
> - Read Problems
> - Write Problems
> - Read Security Problems
> - Write Security Problems  
See screen shot  
![Alt text](<Screenshot from 2023-10-04 19-30-07.png>)

> We use the token in our rule book. See below example:
![Alt text](<Screenshot from 2023-10-04 20-48-32.png>)

> 3. Access to ServiceNow   
> Create a custom Credential for ServiceNow in AAP Controller:  
> In this demo I use basic auth from AAP Controller to ServiceNow.
> 
>> For Input Configuration

```
fields:
  - id: instance
    type: string
    label: Instance
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
required:
  - instance
  - username
  - password
```
For Injector configuration
```
env:
  SN_HOST: '{{instance}}'
  SN_PASSWORD: '{{password}}'
  SN_USERNAME: '{{username}}'
```
>Now that you have a ServiceNow credential type you're ready to use it.   
> - Create a new credential in controller and chose your newly created ServiceNow credental. 
> - Provide the URL to your instance, the userid and password to authentiate to ServiceNow.  
![Alt text](<Screenshot from 2023-10-04 21-21-37.png>)
> You will need this credential later for the job template you'll use to create, update, and close ServiceNow incident tickets in your demo.
> 4. You would have needed to already integrated EDA Controller with AAP Controller using and access token. If you have not done that and need help check out this instruqt lab: [Link here](https://play.instruqt.com/redhat/invite/g0wofhztypx3?utm_source=instruqt&utm_medium=share_button&utm_campaign=referral&icp_instruqt_share_button_referral=true)
> 5. I do this demo in my lab in AWS.You will need to make sure you have a managed node running Nginx  on RHEL and you can connect to the inventory with become true with your machine credential defined in AAP Controller.  
>> - Make sure the managed node is defined in your inventory in AAP Controller.
>> - In this demo we are only using the default landing page that comes with Nginx  when it's installed. Make sure all firewall rules either on RHEL via firewalld or cloud firewall rules are opened for port 80.
> 6. You will need to create a Decision Environment. See below example build file I used:
```
---
version: 3

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-24/de-minimal-rhel8:latest

dependencies:
  galaxy:
    collections:
      - ansible.eda
      - dynatrace.event_driven_ansible
  system:
    - pkgconf-pkg-config [platform:rpm]
    - systemd-devel [platform:rpm]
    - gcc [platform:rpm]
    - python39-devel [platform:rpm]

options:
  package_manager_path: /usr/bin/microdnf
```
> I used ansible-builder v3 to build my DE. I pushed my DE to my Private Automation Hub. You can use what ever container repository you want as long as you can pull it into your EDA Controller. 
![Alt text](<Screenshot from 2023-10-04 21-42-52.png>)
---
> 7. You will need to create an Execution Environment with the servicenow.itsm collection in it and make sure it's available for use in AAP Controller. 
# Setup Resources in AAP Controller and EDA Controller:

### One more thing before we start
As of this writing (10-4-23), you can not vault secret variables that are used in rule books. See link for using variables in rule books. 
[Link here](https://ansible.readthedocs.io/projects/rulebook/en/stable/variables.html#accessing-variables-in-your-rulebook)

In the EDA Controller you can store the variable needed for the Dynatrace plugin when you create the Rulebook Activation a shown below:
![Alt text](<Screenshot from 2023-10-04 19-34-20.png>)

### Okay, now that's out of the way...Let's start. At this point you should have a RHEL vm up running Nginx  (see prerequisites section)

1. Log into Dynatrace. We need to Download the Dynatrace OneAgent to install on our managed RHEL VM running Nginx .
   
> - under Deploy Dynatrace click start installation
 ![Alt text](<Screenshot from 2023-10-04 22-23-23.png>)

 Select Linux:
 ![Alt text](<Screenshot from 2023-10-04 22-25-39.png>)

 If this is your first time downloading the OneAgent, go ahead and click the create token button and then save it in a secure place. Note if you click Create token again in the future you will be replacing your previous token.
 ![Alt text](<Screenshot from 2023-10-04 22-30-22.png>)

 At this point you're ready to follow the instructions above onto your target node running Nginx .  
 Reboot your target node running Nginx .
 Let's move onto the next step.  

 2. You've rebooted your target node. At this point the Nginx  and OneAgent processes should be running. Great! Already the OneAgent is discovering what all is running on the host. Let's take advantage of that and setup a process monitor for Nginx . 
>>In this demonstration, we are setting up a Process availability monitoring rule. Instructions for how to do that can be  [found here](https://www.dynatrace.com/support/help/platform-modules/infrastructure-monitoring/hosts/monitoring/process-availability) 
however, I will be providing the steps here for how I set it up to monitor Nginx .
>> 1. Log into Dynatrace
>> 2. Go to settings at the bottom left then Process availability.
>> 3. Under Process availability click 'Add monitoring rule'
>> 4. Give the monitor rule name a name (I put Nginx  Host monitor)
>> 5. Click Add detection rule
>> 6. Keep 'Select process property' set to Executable
>> 7. For Condition put $contains(Nginx )
>> 8. Save the the setting in the lower left

![Alt text](<Screenshot from 2023-10-06 10-35-33.png>)


> Oh boy you're cooking now. Let's see if this thing works. Ready to kill something? :smirk:
  
> STOP :stop_sign: before we go.. it's important to pause and appreciate what you've done so far. You've setup a host-level monitor for Nginx  running on your target host for your demo. Why do you care? Well by using a host-level monitor you will have the impactedEntities key in the json payload set to the the hostname that will go to EDA to and dynamically run a Job in Controller only on the troubled node that created the problem alert in Dynatrace. See below a preview of what that payload will look like because you'll see in the playbook and rulebook that is used in this repo.
![Alt text](<Screenshot from 2023-10-06 10-48-14.png>)

3. Hop over to your RHEL managed host that's running Nginx . Let's get that Nginx  process so we know what we're killing! :sunglasses:
```
$ systemctl status Nginx 
● Nginx .service - The Nginx  HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/Nginx .service; enabled; preset: disabled)
     Active: active (running) since Wed 2023-10-04 20:41:33 UTC; 6h ago
    Process: 99269 ExecStartPre=/usr/bin/rm -f /run/Nginx .pid (code=exited, status=0/SUCCESS)
    Process: 99270 ExecStartPre=/usr/sbin/Nginx  -t (code=exited, status=0/SUCCESS)
    Process: 99275 ExecStart=/usr/sbin/Nginx  (code=exited, status=0/SUCCESS)
   Main PID: 99286 (Nginx )
```
Looks like we got the pid in our sites. Let's kill it!:scream:
```
$ sudo kill -9 99286 <- such a brutal death 
```
4. Let's see if our process monitor that we set up in Dynatrace even cares about what's happened. This could take a few minutes.
>> Log into Dynatrace. Go to problems and then filter on open status:
![Alt text](<Screenshot from 2023-10-04 23-23-22.png>)
>> Oh boy. Houston we've got a problem :skull:

5. Well at this point we have:
> - installed the OneAgent
> - Setup a host level process monitor in Dynatrace
> - Killed Nginx  and demonstrated we get a problem alert in Dynatrace. 
> - Now let's pivot to what is in this repo. 
> - Go ahead and restart Nginx  on your target node to let the problem alert clear in Dynatrace. We have some setup to do in AAP!

6. Log into your EDA Controller. 
   > That Decision Environment you created earlier, well that needs to be pulled into EDA now.
   ![Alt text](<Screenshot from 2023-10-04 23-43-11.png>)

    > Next fill in the information you need to pull in your DE (You may need to create a credential to use to authenticate to your container repository to complete the pull.) 
   ![Alt text](<Screenshot from 2023-10-04 23-49-30.png>)

   > You have a DE now. We're getting closer to saving this little feller. :ghost:

7. Just like how AAP Controller has projects to pull in your playbooks, EDA has projects to pull in your rulebooks. So let's do that. 
   > Go to projects and then create project
   ![Alt text](<Screenshot from 2023-10-04 23-56-51.png>)
8. Fill in the information. 
   ![Alt text](<Screenshot from 2023-10-04 23-57-54.png>)
   > Note for credential you'll need to create one first before this step. I created a GitHub personal access token to git my repo where my rulebook sits.
9. Okay you should have a project and a DE at this point.
10. Now create a Rulebook Activation
![Alt text](<Screenshot from 2023-10-05 00-02-59.png>)
11. Fill in the fields (Name, Description, Select your project, then you can select your rulebook)
![Alt text](<Screenshot from 2023-10-05 00-04-12.png>)
![Alt text](<Screenshot from 2023-10-05 00-06-22.png>)
> Select the dynatrace.yml from this repository
![Alt text](<Screenshot from 2023-10-05 00-10-28.png>)
> Note you can put in the following variables for connecting to Dynatrace:
```
dynatrace_host: https://xxxxx.live.dynatrace.com
dynatrace_token: YOURTOKENTHATYOUCREATEDINDYNATRACE
dynatrace_delay: 30
```
> Click Create rulebook activation

> At this point we are finished in EDA Controller.Let's pivot to AAP Controller.
12. Log into your AAP Controller. You will need to have created the following resources:
  > - Machine Credential to access your RHEL host running Nginx 
  > - ServiceNow Credential 
  > - Project that has this repo in it
  > - an Execution environment with the Service Collection in it.
  > - Inventory that can be set to the job template that includes the node that we plan to kill Nginx  on.
13. Create the Job Template See below:
  
  ![Alt text](<Screenshot from 2023-10-06 10-59-04.png>)
  
>> As of this writing, the new [run_workflow_template is not GA in EDA... YET!](https://ansible.readthedocs.io/projects/rulebook/en/latest/actions.html#run-workflow-template)  
>> For now, we are opening, updating, and closing tickets while we fix the problem all in 1 playbook. That will change soon so we can take advantage of workflows!

So I think we have all the pieces together. At this point I would have 4 tabs up in my brower:  
1. Tab for ServiceNow Incident screen
2. Tab for Dynatrace so you can see the problem alert when you kill the Nginx  pid on the host
3. Tab for AAP Controller (in the jobs screen)
4. Tab for EDA Controller (in the Rule Audit screen to see the event fire )  
   
   At this point, you can kill the nginix process again on your managed host!

See below examples:
Tab for ServiceNow:
![Alt text](<Screenshot from 2023-10-06 11-27-59.png>)

Tab for Dynatrace (Be in problem screen)
![Alt text](<Screenshot from 2023-10-06 11-37-45.png>)

Tab for EDA Controller (in the Rule Audit Screen). Shortly after you kill Nginx  you should see a new triggered event.
![Alt text](<Screenshot from 2023-10-06 11-30-45.png>)

Tab for AAP Controller (Jobs screen)
![Alt text](<Screenshot from 2023-10-06 11-38-21.png>)

By now you can go look at the job details and see the extra vars passed in from the Rule book:
![Alt text](<Screenshot from 2023-10-06 11-39-52.png>)

Well you're armed up to do a demo! Enjoy. Feel free to improve or enhance the repo as you see fit!



