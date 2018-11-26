---
title: "The Raspberry Pis"
date: 2018-11-10T16:01:46-05:00
weight: 1
toc: "true"
---

For our servers and clients, we used 8 raspberry pi 3B+ running a modified Raspbian image. Each of these can be purchased for under $50. As they run Linux, they are able to run all of the different congestion control algorithms (specially, we are interested in BBR). 

We provide information on the steps we took to recompile the Kernel, as well as performance measurements for the Pis.

<!--more-->

Each raspberry pi is configured to run [raspbian](https://www.raspberrypi.org/downloads/raspbian/), a modified version of debian for the raspberry pi. The kernel image is rebuilt from source to provide the latest support for bbr. The source for this image can be found [here](https://github.com/SaahilClaypool/raspberry-linux). The content below discusses how to modify, add 

# Kernel 

see **blah TODO** for adding a new congestion control protocol

Flashing a new kernel image is relatively simple; the new kernel just needs to be copied into place (/boot) and the config file /boot/config.txt needs to be edited to point to the new image. 

> Note: it may be easier to simply take the sd card out and copy the files into place. But, it should be possible to do this remotely as well. 

## Cross Compiling with Docker

To compile the custom sources, they first must be downloaded from [this github repository](https://github.com/SaahilClaypool/raspberry-linux). After downloading them, we can use the docker image found [here](https://hub.docker.com/r/saahil/rpi_build/) to compile the kernel for ARM. This includes all of the tools necessary to do so. 

The Docker image can be build from [this directory: (https://github.com/SaahilClaypool/rpi/tree/master/Config/Docker) 

Below is the command that can be used to compile the given source repository for the raspberry pi. This place all the compiled artifacts into the `out/build` directory. The file structure will look like 

```
- out
--- build
------ ext4
------ fat32
```

```sh 
docker run --rm    \
    -v /home/{user}/raspberry/source:/linux \
    -v /home/{user}/raspberry/out:/out \
    -e _UID=`id -u` \
    -e _GID=`id -g` \
    -it saahil/rpi_build sh /root/build_cmd.sh
```

## Flashing the pi

Once the new image is compiled to the `out` directory, it needs to be copied over to the raspberry pi's sd card. This process is similar to what is described in the [official raspberry pi documentation](https://www.raspberrypi.org/documentation/linux/kernel/building.md)

>Note: If you are using a *windows* machine, it will not be able to write to the ext4 partition. This can be solved by using a virtualbox machine with access to usb devices, or a native linux computer. 

The steps are roughly as follows (for flashing an SD card plugged into a reader): 

1. Download a [raspbian stretch light image](https://www.raspberrypi.org/downloads/raspbian/)

2. Plug the sd card into computer

3. Use [etcher](https://www.balena.io/etcher/) to flash the sd card

    Note: there are probably ways to do this from the command line, but etcher is simple to use

4. Check the partition labels for the newly flash'd ssd

    These should be /dev/sdf1 and /dev/sdf2 or /dev/sdc1 etc.

5. Mount these partitions and copy the files onto the sd card to update the kernel

    After the initial installation this *should* be possible to do over ssh. 

    It is also necessary to set up SSH for your first login. This can be done by modifying the `/etc/hosts` and adding a blank ssh file to the fat32 directory. The password will be `raspberry1`, which should be changed.
    
    The script to do this can be found below.

    > copy_kernel.sh
    ```sh
    lsusb # check usb sd card names
    sudo mount /dev/sdf1 /mnt/fat32 # change /dev/sdf1 as per the output from lsusb
    sudo mount /dev/sdf2 /mnt/ext4
    sudo cp -rf $OUT/build/ /mnt/
    sudo sh 'echo new_host_name > /mnt/ext4/etc/hostname' # edit the hostname to be something useful. 
    sudo umount /mnt/fat32 # unmount the mounted usbs
    sudo umount /mnt/fat32
    ```

# Configuration

The steps above *should* cover most of the configuration required, along with the steps [here](/config/router/) to configure the router.

Installing software can be done through 'apt'. To compile from source, you need to ensure you cross compile for ARM. 

We configure the sysctl.conf file to ensure that the memory limits for the ethernet interfaces are not limiting the raspberry pi speed. See our documentation [here](/config/router#systl-conf) for more information.

# Technical Specs

## Throughput

We test the 'optimal' throughput for the raspberry pi using the loopback interface. 

## IO Speed