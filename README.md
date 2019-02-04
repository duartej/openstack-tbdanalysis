# Openstack virtual machine setup-up: tbdanalysis
Virtual machine to run EUTelescope analysis code accessing EOS

## Instance creation 
Create the multipart file to be use for the user configuration
```bash
write-mime-multipart -o user_data_context_mlt.txt user_data_ci.txt user_data_bs.txt
```
Create the instance using a CC7 image (so far there is some problem with the instance
creation using a e-group, workaround in the meantime)
```bash
# Create a persistent volume using CC7 image (2018-12-03:65bc0185-7fba-4b08-b256-cfc7576d9dda) 
# From my personal space (not requested volumes in the project)
$ openstack volume create --image 65bc0185-7fba-4b08-b256-cfc7576d9dda \
    --size 80 --description "CC7 analysis software for dockerize test-beam analysis for IT-CMS upgrade" cms-it-tb2 
#  The volume id is 4ecd9e52-1be1-4cc2-9b43-37989de10f5d
$ openstack volume transfer request 
# Create the server  at "CMS IT Ph2 Test Beam Analysis" (project c44e1040-5691-4ea5-881c-eb9a4b4d97e6)
$ openstack --os-project-id c44e1040-5691-4ea5-881c-eb9a4b4d97e6 \ 
    server create --flavor m2.large --key-name lxplus \
    --volume 4ecd9e52-1be1-4cc2-9b43-37989de10f5d \
    --property landb-description="TEST-BEAM EUDET type dockerized analysis server" \
    --property landb-os="LINUX" --property landb-mainuser="duarte" \ 
    --user-data user_data_context_mlt.txt tbdanalysis
# The server ID is  5eedd104-09fb-40d5-8ced-91a0f1873588
# WORKAROUND: set e-group
# Re-set the mainuser property, enter in the instance as root and run the /root/post-install.sh script
$ openstack server set --property landb-mainuser="CMS-IT-TB-SPS" 245f6b7e-b0e6-4871-b151-434c7adc6e29
$ ssh root@tbdanalysis.cern.ch
# cd /root/
# ./post-install.sh
# reboot
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
ssh -oGSSAPIDelegateCredentials=no <your user>@tbdanalysis.cern.ch
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

