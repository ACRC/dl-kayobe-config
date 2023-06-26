Custom Playbook Notes
=====================

Dell Server Profile
-------------------

We use the iDRAC server profile to apply BIOS settings. It all builds on
this ansible collection:
https://docs.ansible.com/ansible/latest/collections/dellemc/openmanage/index.html

It takes care to ensure that the profile applies in an idempontent way.
Also beware that sometimes an iDRAC reset is required for all the BIOS
changes to be recognised.

Dell Firmware Update Docs
-------------------------

We make use of the Dell EMC Repository Manager (DRM):
https://www.dell.com/support/kbdoc/en-uk/000177083/support-for-dell-emc-repository-manager-drm

There is a version for Linux that we make use of via Docker:
https://dl.dell.com/FOLDER09359484M/1/DRMInstaller_3.4.3.869.bin

To run DRM in a container, first start a container, that has a docker
volume to host all the firmware:

::

   mkdir -p /home/stack/dell_firmware
   docker run --detach -v /home/stack/dell_firmware:/dell_firmware --name dell-drm --network host --restart always ubuntu:jammy sleep infinity

Copy in, and then run the installer:
::
   curl -O https://dl.dell.com/FOLDER09359484M/1/DRMInstaller_3.4.3.869.bin -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/110.0'
   docker cp DRMInstaller_3.4.3.869.bin dell-drm:/root
   sudo docker exec -it dell-drm bash
   cd /root
   chmod +x DRMInstaller_3.4.3.869.bin 
   ./DRMInstaller_3.4.3.869.bin

Now you can run DRM, and download a new repo:

::

   /opt/dell/dellrepositorymanager/DRM_Service.sh &
   drm --create -r=idrac_repo_joint --inputplatformlist=R7525
   drm --deployment-type=share --location=/dell_firmware -r=idrac_repo_joint

Note: sometimes the create call had to be run mutlple times before it
worked, with errors relating to Unknown platform: R6525. Restaring the
service might be required.

Now we have the all the files in the docker volume, we can start apache
to expose the repo:

Using this dockerfile to support TLS
::

    FROM httpd:2.4

    RUN sed -i \
                    -e 's/^#\(Include .*httpd-ssl.conf\)/\1/' \
                    -e 's/^#\(LoadModule .*mod_ssl.so\)/\1/' \
                    -e 's/^#\(LoadModule .*mod_socache_shmcb.so\)/\1/' \
                    -e 's/Listen 80/#Listen 80/' \
                    conf/httpd.conf

Build a docker image
::

   [stack@seednode1 https]$ docker build --network host -t httpd:local .

Generate a self-signed cert
::

   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout apache.key -out apache.crt
 
Run the container
::

   sudo docker run -d --name dell-drm-web --network host -v /home/stack/dell_firmware/:/usr/local/apache2/htdocs/ -v $PWD/apache.crt:/usr/local/apache2/conf/server.crt -v $PWD/apache.key:/usr/local/apache2/conf/server.key docker.io/library/httpd:local


Updating the Repo
-----------------

At a later date we will want to re-baseline to a new version. The repo
can be updated:

::

   docker exec -it dell-drm bash
   drm --update -r=idrac_repo_joint
   # check that it has iterated to a new version
   [root@c78921e27b04 dell_firmware]# drm -li=rep

   Listing Repositories...

   Name                   Latest version   Size      Last modified date
   ----                   --------------   ----      -------------
   idrac_repo_join   1.02             8.64 GB   1/31/22 5:42 P.M

   # share the new version
   drm --deployment-type=share --location=/dell_firmware -r=idrac_repo_joint:1.02

   ls -ltra
   -rw-r--r--   1 root root  12038084 Jan 31 17:56 idrac_repo_joint_new_1.02_Catalog.xml

The update the dell_drm_repo variable in drac-firmware-update.yml


Manually adding and update file
--------------------------------

Clone an update package the windows format (the idrac knows how to process these):
::

   curl 'https://dl.dell.com/FOLDER09614074M/2/Network_Firmware_77R8T_WN64_22.36.10.10.EXE?uid=39eab3c7-5ad6-4bfc-be6e-b9d09374accd&fn=Network_Firmware_77R8T_WN64_22.36.10.10.EXE' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/110.0' -O Network_Firmware_77R8T_WN64_22.36.10.10.EXE

import it into your repo:
::

    drm --import --repository=idrac_repo_joint:1.00 --source=/root --update-package="*.EXE"

Export the repository:
::

   drm --deployment-type=share --location=/dell_firmware -r=idrac_repo_joint:1.02

