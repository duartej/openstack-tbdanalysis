# Openstack virtual machine setup-up: cmsit-tb
Virtual machines to run EUTelescope analysis code accessing Test beam EOS data

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
openstack --os-project-id c44e1040-5691-4ea5-881c-eb9a4b4d97e6 stack create -t cmsit_tb_group_stack.yaml cmsit-tb_stack
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
    * Be sure you have created your kerberos credentials, then try to `ls /eos/cms` in the `cmsit-tba<ID`
    machine. If there is no problems, enter into the container and try again

## Deployment 
Central changes in the [eudaq](https://github.com/duartej/eudaq) or 
[eutelescope](https://github.com/duartej/eutelescope) codes, shall be 
propagated first to the dockerhub images and after that to the analysis
machines. The quick and safest way is by destroying and re-creating the stack, once
the dockerhub contain the newly build images.


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
