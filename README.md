# Openstack virtual machine setup-up: cmsit-tb
Virtual machines to run EUTelescope analysis code accessing Test beam EOS data

##### Table of Contents  
1. [Instances creation](#instances-creation)
2. [Usage instructions](#usage-instructions)
3. [Troubleshooting](#troubleshooting)
4. [Deployment](#deployment)
5. [Maintenance & Development](#maintenance-and-development)



## Instances creation 
Create the multipart file to be used for the user configuration:
```bash
write-mime-multipart -o user_data_context_mlt.txt user_data_ci.txt user_data_bs.txt
```

Template file `cmsit_tb_group_stack.yaml` for orchestration:
```yml
heat_template_version: 2018-03-02
  
description: >
    Create  VM instances servers for CMS Inner Tracker Phase-II
    test beam analysis

resources:
    group:
        type: OS::Heat::ResourceGroup
        properties:
            count: 10
            resource_def:
                type: OS::Nova::Server
                properties:
                    name: "cmsit-tba0%index%"
                    key_name: lxplus
                    image: 65bc0185-7fba-4b08-b256-cfc7576d9dda
                    flavor: m2.large
                    metadata: {"cern-services": "true",
                        "landb-description":"TEST-BEAM EUDET type dockerized analysis server",
                        "landb-os": "LINUX",
                        "landb-mainuser": "CMS-IT-TB-SPS"
                        }
                    user_data_format: RAW
                    user_data:
                        get_file: user_data_context_mlt.txt
```
Create the stack of servers (10) using the template in the file `cmsit_tb_group_stack.yaml`: 
```bash
openstack --os-project-id c44e1040-5691-4ea5-881c-eb9a4b4d97e6 stack create -t cmsit_tb_group_stack.yaml cmsit-tba
```
A stack of servers named **cmsit-tb0\<ID\>.cern.ch** will be created. Note there is no mechanism to balance the access to 
the servers so far. 
(*TODO*: Incorporate the servers into the puppet configuration mechanism and integrate them into an automatic load balancer)

## Usage Instructions
In order to be able to access in the VM machine you have to belong to the `CMS-IT-TB-SPS`
e-group. Using your NICE user:
```bash
ssh -oGSSAPIDelegateCredentials=no <your user>@cmsit-tb0<ID>.cern.ch
```
Being `<ID>` any number between `0` and `9`. 
Note the `-o` option in order not to accept delegated kerberos credentials. This will avoid having 
problems with the AFS filesystem within the docker analysis container.

Those machines are intented to be used in a similar way than the `lxplus` machines, that means you
will have access to your home `AFS` (which is your home here as well), several EOS directories
(including CMS-related and USER) and some `CVMFS` software (desy).

### Services
* `/afs`: You have read and write permissions like in `lxplus`
   * Your home directory inside `cmsit-tb0<ID>` is the same than in `lxplus`
* `/eos`: You have read and write permissions like in `lxplus`
   * Test beam data is in `/eos/cms/store/group/dpg_tracker_upgrade/BeamTestTelescope`
* Software is placed under `/sw/repos` folder (although you don't have write permissions)

#### Analysis container 
Once logged into the `cmsit-tb0<ID>` machine, you can launch the docker container
with the needed code to perform the EUTelescope offline analysis. 
```bash
docker-analysis
```
The above command will create a container and give you a terminal inside it. `Marlin`,
`root` and other offline analysis commands are available in there. Again, `/afs` and 
`/eos` folders  are available, as well. 

The steering files from the EUTelescope processor can be found in the docker-container at:
* `/home/eudaquser/sps-tb-201806-eutel-cfg`, but in read only mode

## Troubleshooting
* You don't have access to AFS/EOS in the `cmsit-tba<ID>` machines:
    * Be sure you have your kerberos credentials: `kinit <niceuser>@CERN.CH`
* You don't have access to EOS in the docker container:
    * Be sure you have created your kerberos credentials, then try to `ls /eos/cms` in the `cmsit-tba<ID>`
    machine. If there is no problems, enter into the container and try again

## Deployment 

Every time any of the main codes and configuration elements are modified,
the stack should be reinitialized. The easy, less error-prone way to do it,
is by destroying and re-creating the stack. 
The main codes are: 
 * [eudaq](https://github.com/duartej/eudaq), with the relevant docker image at
 [dockerhub:eudaq](https://cloud.docker.com/u/duartej/repository/docker/duartej/eudaqv1-ubuntu)
 * [eutelescope](https://github.com/duartej/eutelescope), with the relevant docker image at
 [dockerhub:eutelescope](https://cloud.docker.com/u/duartej/repository/docker/duartej/eudaqv1-ubuntu)
Note that before creating the new stack, the software code changes should be propagated to the
dockerhub images.

And the main configuration elements:
 * [user_data_bs.txt](https://github.com/duartej/openstack-tbdanalysis/blob/master/user_data_bs.txt): 
 user configuration, blablah
 * [user_data_ci.txt](https://github.com/duartej/openstack-tbdanalysis/blob/master/user_data_ci.txt):
 user configuration, blahblah
 * [cmsit_tb_group_stack](https://github.com/duartej/openstack-tbdanalysis/blob/master/cmsit_tb_group_stack.yaml):
 The description file to create the stack. 

The list of commands to destroy and create again the stack are described
below. The `openstack` commands are available at `lxplus-cloud.cern.ch`.

#### Obtain the project id to be used with `--os-project-id` 
```bash
$ openstack project list
```

### Destroy the stack
```bash
# First get the stack id
$ openstack --os-project-id <PROJECTID> stack list
# Then delete them
$ openstack --os-project-id <PROJECTID> stack delete <STACKID>
```
Wait until the vm instances are actually destroyed and all the resources freed.

### Create the stack
Before create the stack, and if needed, create the multipart file 
```bash
# Create the multipart config file
write-mime-multipart -o user_data_context_mlt.txt user_data_ci.txt user_data_bs.txt
# Create the stack
openstack --os-project-id c44e1040-5691-4ea5-881c-eb9a4b4d97e6 stack create -t cmsit_tb_group_stack.yaml cmsit-tba
```

## Maintenance and development
* Repositories updates within the `/sw/repos` file. Use the `analyser` user (no login):
```bash
# Just sudo /bin/bash first
[root] cd /sw/repos/<repo_to_update>
[root] su -s /bin/bash -c 'git pull' analyser
```
* Analysis software: use of software developement versions. 
The `docker-analysis` alias creates a container defined at `/sw/repos/dockerfiles-eutelescope/docker-compose.yml`. The container uses the eudaq and eutelescope softwares from the pre-built image. In order to use explicitely eudaq software from the host. 
  * Use the **EUDAQ software from the host**
  ```bash
  $ cd /sw/repos/dockerfiles-eutelescope
  # To compile EUTElescope using EUDAQ from the host, and let the compiled objects at the host (which could be use afterwards
  $ docker-compose run --rm compile
  # To used the host compiled EUDAQ with the EUTelescope image
  $ docker-compose run --rm devcode
  # To used the host compiled EUDAQ with the EUTelescope image, privileged mode, able to be used with the TLU
  $ docker-compose run --rm devcode
  ```
  * **EUTelescope development**. In order to develop and test the EUTelescope code in the container, several steps are needed to dump the repo to the host and then be able to write code (in the host), compile (in the container) and use it (in the container). 
  ```bash
  # Edit the docker-compose.override.yml file to include the binding of the host directory to the container
  $ sudo /bin/bash
  [root] su -s /bin/bash -c 'vim docker-compose.override.yml' analyser
  ```
  Add the bind volume in the `devcode` section of the yaml file:
  ```yaml
  - type: bind
    source: /sw/repos/Eutelescope
    target: /eudaq/ilcsoft/v01-19-02/Eutelescope/
  ```
  Get an previous eutelescope container or create a new one, without deleting it: 
  ```bash
  # Back as regular user
  [root] exit
  $ docker container ps -a
  # If no container, create one and exit
  $ docker run devcode
  # Entering again as superuser and copy the container eutelescope software into the host
  [root] su -s /bin/bash -c 'docker cp 7c252d34dbf2:/eudaq/ilcsoft/v01-19-02/Eutelescope/ /sw/repos/' analyser
  # Exit superuser and lauch the devcode service:
  $ docker-compose run --rm devcode
  ```
  Remember to edit the source files on the host using the `analyser` user:
  ```bash
  su -s /bin/bash -c 'vim master/processors/src/EUTelAPIXTbTrackTuple.cc' analyser
  ```
  You must use a different container to run (`analysis` modified with the binded mounted src folder) than to compile.

## References
* Puppet-managed VM at CERN: (foreman) https://judy.cern.ch
    * `http://configdocs.web.cern.ch/configdocs`
    * DNS Load balancing (https://aiermis.cern.ch)
* Self-managed Virtual Machines: `https://clouddocs.web.cern.ch/clouddocs`
    * Stacks: Collection of resources which can be created by the Orchestration service (Heat), 
    including Vms, networks, rules, ... 
        * `https://clouddocs.web.cern.ch/clouddocs/tutorial/create_a_stack.html`
        * `https://clouddocs.web.cern.ch/clouddocs/orchestration/advanced_concepts.html`
    * Cluster of docker containers: Swarm 
