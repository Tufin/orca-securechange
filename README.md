
# Orca-SecureChange

[![CircleCI](https://circleci.com/gh/Tufin/orca-securechange.svg?style=svg)](https://circleci.com/gh/Tufin/orca-securechange)

## Overview
A SecureChange add-on to automate Kubernetes egress connectivity on firewalls


## Getting Started

These instructions show the steps required for integrating Orca with Tufin SecureChange in order to get an end-to-end egress security policy for kubernetes cluster.

## prerequisites

The following are required in order to succesfully complete the integration:
1. On your organizational firewalls, allow connection from the local SecureChange to https://securecloud.tufin.io
2. Make sure you have 1)SecureChange security admin user rights for the console 2)root access to the SecureChange server.

### Defining an API token in Orca

* Login to Orca, go to **Settings -> Create Token** and create an API token for the bearer authentication. Select 'agent' for the token scope and give it the label 'secure-change'.

### Installing the integration package on SecureChange server

* Download the installation package from the [release](https://github.com/Tufin/orca-securechange/releases) tab on Github.
* Upload the package setup_orca_ps_scripts-x.y.z.run to the SecureChange machine.
* Start the installation: 
   ```
   sh setup_orca_ps_scripts-x.y.z.run
   ```
   * Enter a valid username and password for both SecureChange and SecureTrack.
   * For Orca authentication enter username="orca" and password="Bearer <Orca_API_Token>".
* Open the file **/usr/local/orca/conf/custom.conf** and define the following parameters:
   * hostname=orca.tufin.io
   * For every instance of /bridge/X/Y/Z - replace X with your ORCA DOMAIN ands Y with ORCA PROJECT.
   * securechange host = <your securechange server address>
   * securetrack host = <your securetrack server address>
* Restart the 'tufin-ps-orca-group-change' service on SecureChange:
  ```
  service tufin-ps-orca-group-change restart 
  ```
* Now SecureChange should initiate TCP/443 connection with Orca. You may check it using:
  ```
  netstat -natp | grep 443
  ``` 
  
#### Updating credentials to SecureChange/SecureTrack in the encrypted credentials store

In order to update credentials run the following:
```
/usr/local/orca/bin/set_secure_store.py -o
```
A wizard will allow you to enter your new credentials.

In order to update the Orca_API_Token run the following:
```
/usr/local/orca/bin/set_secure_store.py -s auth_header_integration
```
You'll be prompted for the new token.

### Configuring SecureChange for the integration

**Adding a new workflow in SecureChange**   

* Login to SecureChange through the web UI
* Create a workflow in SecureChange.
   * Workflow name: Orca Group Change
   * Type: Modify group
    
   * First step name: Submit network object group request
     * The first step of the workflow should include the following fields:
      
      | Field Type         | Field display Name           |
      | ----------         | ----------                   |
      | multi_group_change | Modify network object group  | **Modify group + multiple**
      | text_field         | Orca Task ID                 |
      | text_field         | Group Name                   |
      
      The package comes with default names which can be changed later.      
     * Assignments: choose the allowed users to open requests
   * Second step name: Modify group
     * Step mode: auto + actions: Run designer, Update policy/device
     * Assignments: Mode: Auto-assign, Choose a user
     
**Adding a new script to SecureChange**

   * Go to Settings -> SecureChange API.
   * Click on the Add script button.
   * Give a name to the new script.
   * In the "Full path" field enter the following path: /usr/local/orca/bin/rest_integration.py.
   * Enter the Trigger Group Name.
   * Select the Orca workflow.
   * Select all the triggers in the Triggers section.
   * Save settings.
   
### Preparing the Firewall Policies

* The SecureChange workflow triggered from Orca is based on "Modify Group"
* It will look for certain group names accross the monitored firewalls and update them according to the egress end-points (destination) in Orca.
* You need to create these groups manually on the relevant firewalls using the following naming convention for group names: &lt;domain&gt;.&lt;project&gt;.&lt;namespace&gt; 
* You also need to create rules on the firewalls that use these groups as destinations (the source should be the Kubernetes cluster IPs or subnet).
In case the firewall does not allow a rule withoput any destination, you may add a "fake destination".
* Orca does not yet discover protocols and ports. Due to this, there are two options to define the rule "service" field:
    * Use "Any" service and optionally apply Tufin's Automatic Policy Generator (APG) to replace them by specific service later
    * Use https which is the most common protocol for reaching end-points

## Sending security policy to external firewalls

1. In Orca go to **Policy -> Security Policy**. Here you'll see the rule base that was generated dynamicaly following the traffic discovery process.
2. At the upper right corner click on **Actions -> Apply on Firewalls**.
3. At this point Orca will prepare the "Modify group" request to SecureChange.
4. SecureChange will poll Orca (every 60 seconds by default) for requests , open a ticket and process the request. In Orca **Policy -> Firewall tickets**, you'll see a "Processing" log.
5. Follwing completion of "Modify group" workflow, and ticket closure, you'll see in **Policy -> Firewall tickets** a "Implemented" log.

## Logging
The script log is located in /var/log/ps_orca_logger.log 

## License

This project is licensed under the Apache License Version 2.0 - see the [LICENSE](LICENSE) file for details
