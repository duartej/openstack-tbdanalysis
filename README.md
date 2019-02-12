# Openstack virtual machine setup-up: cmssit-tb
Virtual machines to run EUTelescope analysis code accessing Test beam EOS data

## Instances creation 
Create the multipart file to be use for the user configuration
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
Create the stack of servers (10) using the template in the file `cmsit_tb_group_stack.yaml: 
```bash
openstack --os-project-id c44e1040-5691-4ea5-881c-eb9a4b4d97e6 stack create -t cmsit_tb_group_stack.yaml cmsit-tb_stack
```
A stack of servers named **cmsit-tb0\<ID\>.cern.ch** will be created. Note there is no mechanism to balance the access to 
the servers so far. 
(TODO: Incorporate the servers into the puppet configuration mechnism


```
## Extra configuration
A new service for the dockerfiles-eutelescope is created for the analysis. 
The `docker-compose.yml` file is updated with the following service:
```yml
analysis:
        image: duartej/eutelescope:latest
        depends_on:
            - "eutelescope"
        volumes:
            - /var/run/eosd:/var/run/eosd
            - /tmp/krb5cc_${ID}:/tmp/krb5cc_${ID}
            - /afs:/afs
            - type: bind
              source: /eos
              target: /eos
              read_only: true
            - type: bind
              source: /sw/repos/sps-tb-201806-eutel-cfg
              target: /home/eudaquser/sps-tb-201806-eutel-cfg
              read_only: true
            - ${HOME:?HOME variable unset!!}/tdba_entrypoint:/home/eudaquser/afs_gate
            - /etc/passwd:/etc/passwd
        user: ${ID}:${GROUP}
```
That assures the EUTelescope software and also that the AFS (where $HOME is) and EOS (where raw data is)
accessible to the user inside the container.

## Usage Instructions
In order to be able to access in the VM machine you have to belong to the `CMS-IT-TB-SPS`
e-group. Using your NICE user:
```bash
8 <your user>@tbdanalysis.cern.ch
```
Not accepting delegated credentials will avoid having problems with the AFS
filesystem within the docker analysis container.
### Services
* `/afs` is mounted to store created output from `tbdanalysis`:
   * Your home directory inside `tbdanalysis` is the same than in lxplus 
   (see $HOME var)
   * The first time you login in the `tbdanalysis`, the directory 
   `$HOME/tdba_entrypoint` is created (see `analysis container` below)
* `/eos/cms` and `/eos/user` is mounted in order to access the test beam data
   * The eos absolute path can be found in the environment variables `TB*`.
   Check the output of `export |grep TB`
* A directory is created in `/home/analysis/<your user>`, don't populate it too much
(the VM has a limited space of 160 GB), use instead AFS or/and EOS
* Software is placed under `/sw/repos` folder

#### Analysis container 
Once logged into the `tbdanalysis` machine, you can launch the docker container
with the needed code to perform the EUTelescope offline analysis. 
```bash
docker-analysis
```
The above command will create a container and give you a terminal inside it. `Marlin`,
`root` and other offline analysis commands are available in there. Again, `/afs` and 
`/eos` (`eos` in read-only mode) are available, as well. However a short-cut is created:
* in the docker container `/home/eudaquser/afs_gate` --> `$HOME/tdba_entrypoint` accessible
from tbdanalysis and lxplus. 

The steering files from the EUTelescope processor can be found in the docker-container at:
* `/home/eudaquser/sps-tb-201806-eutel-cfg`, but in read only mode

#### Problems
[TO BE FILLED]

