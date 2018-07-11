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
# Create a persistent volume using CC7 image
$ openstack volume create --image 94f151f8-dee0-48c3-b5bd-32994fa4be66 \
    --size 160 --description "CC7 analysis software for dockerize test-beam analysis for IT-CMS upgrade" cms-it-tb 
#  The volume id is 82d821d5-5302-4a28-bcf3-66f1c46bb49e
$ openstack server create --flavor m2.large --key-name lxplus \
    --volume 82d821d5-5302-4a28-bcf3-66f1c46bb49e \
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

