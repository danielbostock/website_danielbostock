---
title: "Project 1 – Ansible – Stage 2 - Deploy New Network & Network Security"
date: "2019-12-26"
hero: /images/posts/netauto5.svg
description: Automating tag objects and network address objects.
theme: Toha
menu:
  sidebar:
    name: Network Automation
    identifier: project1-ansible-stage2
    weight: 500
---

Wow, what an epic trip this was for me personally putting this playbook together. It actually started off very simple but then I started question myself and say to myself how to I prevent me deploying this kind of change in a real world prod environment and not breaking anything. How can I ensure consistency and great feed back within my playbook that someone else within my team at any given work place be able to understand and do the change and not break everything.

  
Well I applied some of the stuff I have either indirectly read about or first handedly made stupid mistakes from and which led me to wanting to automate network changes.

So I set out with this playbook to build in some checks and balances so to speak. One of the reasons Network automation has been slow to take off is because it can actually be slightly hard to implement due to the scare factor of making one slight mis configuration and how much can be impacted by this. However network automation can, if done correctly build in error checking and verifications of success. What I will go over with my playbook is the effort to do this and to ensure there are no mistakes. This does remove some of the free flowing and speed of changes or the deployment of changes.

However I would like to remind the readers that not always things need to be that fast. It depends on context. The important thing is that a playbook or the mentality of the person writing the playbook has the cognitive reasoning to understand which approach is best. With the playbook I have built it is easy to remove or comment out portions when unnecessary such as completely new deployments or the sacred “greenfield” deployments us network engineers often wish we had.

As with all things that I post, I by no means consider myself an expert and would really appreciate any questions and or advice on where to improve or what could be done better. Even questioning why I have done something is something I find helpful as it can me re-think my work and ensure I am doing it the best way possible, so please feel free to comment or reach out to me personally!

Also I have decided to post the results portion as a video as I believe a video showing how the playbook plays out with the pauses is much easier to see and see how well the playbook flows.

GitHub: [https://github.com/danielbostock/Project1-Ansible-Deploy\_NewNetwork-SecPol/blob/master/project1\_stage2\_deploy\_l3interfaces.yml](https://github.com/danielbostock/Project1-Ansible-Deploy_NewNetwork-SecPol/blob/master/project1_stage2_deploy_l3interfaces.yml)

## Playbook

_project1\_stage2\_deploy\_l3interfaces.ym_

\---
```yaml
# Playbook Title: project1\_stage2\_deploy\_l3interfaces.yml
# Version: 1.0
# Playbook created by Daniel Bostock
# Contact me, with ideas to improve or to simply discuss - contact@danielbostock.com
# NO COPYRIGHT
# Feel free to re-use, I obviously take no responsibility for where and how you use it.

- name: Project 1 - Stage 2 - Deploy L3 Intefaces
# Define the target hosts here
  hosts: nvnetlab1\_network\_routers
  gather\_facts: yes
# Define the target interface here
  vars:
    Targ\_L3Interface: GigabitEthernet2
    Targ\_L3Interface\_Description: Downlink to LAB2-CS1

# --- Begin the pre-flight checks - backup, obtain current interface configuration, current running configurations ---

# IF this is a new network device deployment these pre\_tasks checks will often be unecessary and I recommend commenting or removing them.

  pre\_tasks:

# Check if there are no current outstanding and not committed configuration changes

    - pause:
       seconds: 5
       prompt: "Pre-flight checklists for deployment are beginning. This first step is checking if there are outstanding changes on the target device."

    - name: Compare the current Startup vs Running configuration to confirm no outstanding changes
      ios\_config:
          diff\_against: intended
          intended\_config: "{{ lookup('file', '~/Google Drive/Ansible/Lab03-Network/pre\_dep\_backups/pre\_dep\_startup/{{ inventory\_hostname }}\_pre\_dep\_startup.cfg') }}"

    - pause:
       prompt: "Press enter to continue if there are no outstanding changes. If there are; proceed with caution and awareness, as this playbook will deploy these changes as well."

# Begin backup, modify backup directory as required

    - name: Backup of the current running config before deploy
      ios\_config:
        backup: yes
        backup\_options:
          dir\_path: ~/Google Drive/Ansible/Lab03-Network/pre\_dep\_backups/pre\_dep\_running/
        host: "{{ inventory\_hostname }}"

# Obtain and Display to the deployer what the current configuration is

    - name: Obtain current running configuration on routers target interface
      cli\_command:
          command: sh run int {{ Targ\_L3Interface }}
      register: current\_target\_int\_conf
    - name: Display current running configuration on routers target interfaces
      debug:
        msg: "{{ current\_target\_int\_conf.stdout }}"

# Obtain and Display to the deployer what they will be deploying

    - name: Display new configuration before deploying
      debug:
        msg: "{{ lookup('template', 'l3\_ipv4int\_template.j2') }}"
    - pause:
        prompt: "Confirm that the target interface and new configuration is correct. Press enter to proceed, or Control+C to cancel."

# --- End of pre-flight checklist ---

  tasks:

# Step 1 - Create and deploy based off Jinja2 template the L3 Interface
    - name: Deploy the new L3 Interfaces to devices
      cli\_config:
        config: "{{ lookup('template', 'l3\_ipv4int\_template.j2') }}"
      when: ansible\_network\_os == 'ios'

# Step 2 - Now interfaces are created, enable and label with description accordingly
    - name: Enable interface and provide Description
      ios\_interfaces:
        config:
          - name: "{{ Targ\_L3Interface }}"
            description: "{{ Targ\_L3Interface\_Description }}"
            enabled: True
      when: ansible\_network\_os == 'ios'

#Give time for the enable of interfaces to take effect
    - pause:
        seconds: 10
        prompt: Waiting for interfaces to come up...

#Obtain the information to display to the deployer with the handlers the configuration deployed
    - name: Obtain updated IP address configuration
      cli\_command:
        command: show ip int br | s {{ Targ\_L3Interface }}
      register: updated\_ip\_int
    - name: Show the updated IP address configuration
      debug:
        msg: "{{ updated\_ip\_int.stdout }}"

    - name: Obtain NEW running configuation on interface
      cli\_command:
        command: show run int {{ Targ\_L3Interface }}
      register: deployed\_targl3int\_runconf
    - name: Display NEW running configuration on interface
      debug:
        msg: "{{ deployed\_targl3int\_runconf.stdout }}"

#Ping Interfaces
    - name: Pinging from the new interfaces to the LAN edge uplink IP
      net\_ping:
          dest: 10.255.255.1
          source: "{{ Targ\_L3Interface }}"
          state: present
      ignore\_errors: yes

    - pause:
        prompt: "Review configuration, if configuration deployed is correct and no issues, press enter to commit to memory or ctrl-c to leave in running config."

#Now deployment is complete the configuration will be written to the memory (startup\_config)
    - name: Save the deployed configuration (running config) to the memory (pre\_dep\_startup config).
      ios\_config:
          save\_when: modified
```

## Jinja2 Template
```jinja2
_L3\_ipv4int\_template.j2_

{% for interface,ip in nvnetlab1\_routers\[inventory\_hostname\].items() %}<br>
interface {{ interface }}<br>
&nbsp; ip address {{ip}} 255.255.255.248<br>
{% endfor %}
```

This Jinja2 templates references the following default all (vars) yml file. As an FYI the all.yml file is a default vars file for all your host inventory items. This is obviously not a requirement for Jinja2 templates, so feel free to adjust yours but read the associated Ansible documentation about referencing vars files within your Jinja template.

I found the following article the best for getting my head around Jinja2 templating (obviously inclusive is the Ansible core documentations).

[https://ansible.github.io/workshops/exercises/ansible\_network/4-jinja/](https://ansible.github.io/workshops/exercises/ansible_network/4-jinja/)


_All.yml_
```yaml
nvnetlab1\_routers:
NV-NET-LAB1-R1:
   GigabitEthernet2: "192.168.2.17"
NV-NET-LAB1-R2:
   GigabitEthernet2: "192.168.2.18"
NV-NET-LAB1-R3:
   GigabitEthernet2: "192.168.2.19"
NV-NET-LAB1-R4:
   GigabitEthernet2: "192.168.2.20"
```

As you can see Jinja2 templating allows for easily moving through interfaces and unique ip addresses for the respective hostnames. This is achieved primarily by using the _magical variable_ “inventory\_hostname”

## Results:

https://www.youtube.com/watch?v=v3Zf\_nbLhYc

### Results Review:

The one thing I struggled with when building in some fact checking and essentially a pre-flight list was the assumption that someone didn’t do something entirely they should have in either the recent tense or long past tense (as in so long ago are no longer even in the business!). 

How many times have we logged onto a router or a switch and found some configuration has been not been written to the startup but is currently in the running? Well I will wager not many, but when did we find out about it? After a firmware update, later that day when someone perks up and says something that has always worked for them isn’t. Lo and behold, there was a network setup one device as a POC that eventually evolved into a prod situation and because it was a POC it was never actually properly deployed. Yeah that situation never happens right… Well IF we deploy network changes as code it should mostly not happen right!?

Also I should say often it's not a problem if we write this code to memory, but it is a problem come auditing when we could just pick up on it now and be ahead of the auditors for a change...

Anyway I can harp on about this thing and we can trade war stories over drinks all night I am sure, but whatever lets move on. What we do know is network devices in prod environments typically do not get restarted unless an urgent firmware update, therefore this problem hardly rears it's ugly head. Also the fact checking to ensure whatever changes were implemented were saved to the start up config within the last few years was actually done probably happens even less. SO I came to the conclusion that it is imperative to check this before beginning a configuration deployment.

The problem with doing such a check is that you need to apply a specific Ansible command variable. I had use the command -D (as you will see in the video) or  -- diff works as well. This portion of the playbook also assumes you keep regular backups of your startup config which can be loaded and referenced for a diff. As a side note one thing that is not clear with the ios\_config module is what config is backed up when you execute the backup module command. The backup taken is actually just the running the configuration. Therefore you can't just create a new backup and load that, unless you do this with an ios\_command action which I felt to be cumbersome to add in. If you are not keeping regular backups you actually have a bigger problem and you should sort that out before doing this change anyway. If for a lab environment, just quickly do a backup and load that here.

Of course we expect to see this very rarely but it is still a good check to do, again if this is new deployment, I strongly recommend removing this action or if you feel more inclined to comment out instead then do that.

As you see here, the configuration pulled up a configuration difference from the startup config and current running. It would appear that I started to make new LAB network but never followed through, wow this seems like a very rare case where we would never see in a production environment, surely everyone would accurately clean up all their previous unfinished configuration? We definitely know it was unfinished because the interface is still configured in shutdown. Therefore we can proceed with the playbook without any concerns if this change is saved or not as it won’t come up but we know to investigate further or remove. This would never happen in a prod environment right?

So we move through the playbook and find no further issues, except when we get to the ping tests of the R3 Gi2 interface. The configuration all shows correct from what I am deploying but it is down down. This is another reason which simply just shows that Network Automation is great. It shows that configuration is like all others and the configuration is not the problem but something else. Sure enough lo and behold the check box for the the interface was not ticked on VMWare VNIC’s in the Cisco CSR1000V VM. Once I ticked this and it saved, the interface was up and all good. Thus the network automation tool through error checking has allowed me to resolve a fault very quickly.

I should say in this situation, "ignore\_errors: yes" options is enabled here because I have a pause section after where I can review that the reason for failure is not a cause to stop deployment. This can be avoided as well with fancier playbook instructions such as “failure\_when” conditions, however this is unnecessary I feel as sometimes a playbook should be paused to confirm nothing is majorly broken especially before writing the configuration. As many will know when working on remote Cisco devices one of the great things is that you can ask someone to just reboot it (or before deploying a risky change - execute "reload in" command... But sushhh Daniel! We would never make a mistake and need that....) and it will return to the last known working on config after the reboot and all your changes are gone!

## Final Thoughts...

So overall things went really well with this playbook and I am really proud of my pre-flight check list and the post verification testing. I plan to keep leveraging these principles in the next stages. It makes for a slightly slower deployment, but helps to ensure that everything is correctly and accurately deployed which for me saves more time.

  
As already stated please let me know your thoughts and what I could do better. I am looking forward to stage 3 where I will deploy the VLANs and VLAN SVI’s to the Nexus 9k Virtual Core Switch.

