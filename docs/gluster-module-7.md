# **THIS LAB IS A WORK IN PROGRESS**
# Lab Guide <br/> Gluster Test Drive Module 6 <br/> Brick replacement

## LAB AGENDA

Welcome to the Gluster Test Drive Module 6 - Brick replacement

- Simulate a brick failure
- Replace the brick 

## GETTING STARTED

If you have not already done so, click the <img src="http://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl02-working-ebs/media/image005.png"> button in the navigation bar above to launch your lab. If you are prompted for a token, use the one distributed to you (or credits you've purchased).

> **NOTE** It may take **up to 10 minutes** for your lab systems to start up before you can access them.

## LAB SETUP

### CONNECT TO THE LAB

connect to the **rhgs1** server instance, using its public IP address from the **Addl. Info** tab to the right. 
```bash
ssh student@<rhgs1PublicIP>
```

## BRICK REPLACEMENT PREREQUISITES

In this module we will make use of **rhgs1** and **rhgs2** and **client1** only. 
Between the two servers a replicated volume will be set up and the brick on
**rhgs2** will then be put into a faulty state. 

This module uses the replicated volume, so if you have not yet deployed it,
please do so from **rhgs1**
```bash
gdeploy -c ~/materials/gdeploy/repvol.conf
```


On **rhgs1** is an ansible script to take care of the required prerequisites
```bash
ansible-playbook -i ~/materials/ansible/inventory
~/materials/ansible/module6.yaml
```
## SIMULATE BRICK FAILURE

On **rhgs2** we will now disable the brick:

```bash
sudo dmsetup load rhgs_vg2-rhgs_lv2 /student/materials/dmsetup-error-target
sudo dmsetup resume rhgs_vg2-rhgs_lv2
```

This will remove the inital blockdevice to which the map points and replace it
with the error target. This will return an I/O error on every read- or write
operation to it. 

Check the status of the volume on **rhgs1**
```bash
sudo gluster volume status
```
``
Status of volume: repvol                                                                                                                                                     
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick rhgs1:/rhgs/brick_xvdc/repvol         49152     0          Y       14066
Brick rhgs2:/rhgs/brick_xvdc/repvol         49152     0          Y       14831
``

Once the health checker has run (by default it runs every 30 seconds) the output
will indicate that the brick on **rhgs2** is no longer working:

``
Status of volume: repvol                                                                                                                                                     
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick rhgs1:/rhgs/brick_xvdc/repvol         49152     0          Y       14066
Brick rhgs2:/rhgs/brick_xvdc/repvol         N/A       N/A        N       N/A  
``


