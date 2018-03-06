# **THIS LAB IS A WORK IN PROGRESS**
# Lab Guide <br/> Gluster Test Drive Module 7 <br/> Brick replacement

## LAB AGENDA

Welcome to the Gluster Test Drive Module 7 - Brick replacement

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


On **rhgs1** is an ansible script to take care of the required prerequisites. This script only needs to be run if you have taken one of the other labs yet. 

```bash
ansible-playbook -i ~/materials/ansible/inventory ~/materials/ansible/module7.yaml
```

This module uses the replicated volume, please deploy it from **rhgs1**

```bash
gdeploy -c ~/materials/gdeploy/repvol.conf
```

and check its status

```bash
sudo gluster volume status repvol
```

```
Status of volume: repvol
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick rhgs1:/rhgs/brick_xvdc/repvol         49152     0          Y       20492
Brick rhgs2:/rhgs/brick_xvdc/repvol         49152     0          Y       14846
Self-heal Daemon on localhost               N/A       N/A        Y       20512
Self-heal Daemon on rhgs2                   N/A       N/A        Y       14866

Task Status of Volume repvol
------------------------------------------------------------------------------
There are no active volume tasks
```

## CREATE DATA FILES ON THE VOLUME

On **client1** create the directories to mount the volume if not yet done

```bash
ssh student@client1
```
```bash 
sudo mkdir -p /rhgs/client/native/repvol 
```
```bash
sudo mount -t glusterfs rhgs1:repvol /rhgs/client/native/repvol/ 
```                                 

Examine the new mount.                                                                         
                                                                                               
```bash                                                                                        
df -h /rhgs/client/native/repvol/
```                                                                                            
```
Filesystem      Size  Used Avail Use% Mounted on                                                                                                                             
rhgs1:repvol     10G   34M   10G   1% /rhgs/client/native/repvol
```

                                                                                 
Create and set permissions on a subdirectory to hold your data.                  
                                                                                 
```bash                                                                          
sudo mkdir /rhgs/client/native/repvol/mydir                                        
```
```bash
sudo chmod 777 /rhgs/client/native/repvol/mydir    
```
 
Add 100 files to the directory.                                                                
                                                                                              
                                                                                               
```bash                                                                                        
for i in {001..100}; do echo hello$i > /rhgs/client/native/repvol/mydir/file$i; done 
``` 

List the directory, counting its contents to confirm the 100 files written.                                                                                                                   
```bash 
ls /rhgs/client/native/repvol/mydir/ | wc -l 
``` 
``100``                                                                                                                                                                                       
Log out off **client1**
```bash
exit
```


## SIMULATE BRICK FAILURE

Now that we have a healthy replicated volume, we will simulate a brick failure.

Check the status of the volume on **rhgs1**
```bash
sudo gluster volume status repvol
```
```
Status of volume: repvol
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick rhgs1:/rhgs/brick_xvdc/repvol         49152     0          Y       14066
Brick rhgs2:/rhgs/brick_xvdc/repvol         49152     0          Y       14831
```


On **rhgs2** we will now disable the brick:

```bash
ssh student@rhgs2
```
```bash
sudo dmsetup load rhgs_vg2-rhgs_lv2 /home/student/materials/dmsetup-error-target
```
```bash
sudo dmsetup resume rhgs_vg2-rhgs_lv2
```

This will remove the inital blockdevice to which the map points and replace it with the error target. 
As a consequence, /dev/xvdc  will return an I/O error on every read- or write operation to it. 

If you follow the system on **rhgs2** logs using
```bash
sudo tail -f /var/log/messages
```

you can see the file system being shutdown due to errors shortly after
```
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): metadata I/O error: block 0x9f1e00 ("xlog_iodone") error 5 numblks 256
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): metadata I/O error: block 0x0 ("xfs_buf_iodone_callback_error") error 5 numblks 1
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): xfs_do_force_shutdown(0x2) called from line 1200 of file fs/xfs/xfs_log.c.  Return address = 0xffffffffc02dfec0 
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): Log I/O Error Detected.  Shutting down filesystem
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): Please umount the filesystem and rectify the problem(s)
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): Failing async write on buffer block 0xffffffffffffffff. Retrying async write.   
```

Once the health checker has run (by default it runs every 30 seconds) the output
will indicate that the brick on **rhgs2** is no longer working:

```bash
sudo gluster volume status repvol
```
```
Status of volume: repvol                                                                                                                                                     
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick rhgs1:/rhgs/brick_xvdc/repvol         49152     0          Y       14066
Brick rhgs2:/rhgs/brick_xvdc/repvol         N/A       N/A        N       N/A  
```


## REPLACE THE FAULTY BRICK

We need to set up the new brick in the identical way the original one was set up.
Still on **rhgs2** perform these steps

```bash
sudo pvcreate /dev/xvde
```
``  Physical volume "/dev/xvde" successfully created.``

```bash
sudo vgcreate rhgs_vg4 /dev/xvde
```
``  Volume group "rhgs_vg4" successfully created``

```bash
sudo lvcreate -l 100%FREE -T rhgs_vg4/rhgs_thinpool4
```
``  Using default stripesize 64.00 KiB.                                                                                                           
  Thin pool volume with chunk size 64.00 KiB can address at most 15.81 TiB of
  data.
  Logical volume "rhgs_thinpool4" created.``

```bash
sudo lvchange --zero n rhgs_vg4/rhgs_thinpool4
```
``  Logical volume rhgs_vg4/rhgs_thinpool4 changed.``

```bash
sudo lvcreate -V 10G -T rhgs_vg4/rhgs_thinpool4 -n rhgs_lv4
```
```
  Using default stripesize 64.00 KiB.                                                                                                             
  WARNING: Sum of all thin volume sizes (10.00 GiB) exceeds the size of thin pool rhgs_vg4/rhgs_thinpool4 and the size of whole volume group (<10.00 GiB)!
  For thin pool auto extension activation/thin_pool_autoextend_threshold should be below 100.
  Logical volume "rhgs_lv4" created.
```

```bash
sudo mkfs.xfs -i size=512 -n size=8192 /dev/rhgs_vg4/rhgs_lv4
```

```
meta-data=/dev/rhgs_vg4/rhgs_lv4 isize=512    agcount=16, agsize=163824 blks                                                                      
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2621184, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=8192   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

```bash
sudo mkdir -p /rhgs/brick_xvde
```

```bash
echo "/dev/rhgs_vg4/rhgs_lv4 /rhgs/brick_xvde xfs rw,inode64,noatime,nouuid 1 2" | sudo tee -a /etc/fstab
```
```bash
sudo mount /rhgs/brick_xvde
```
```bash
sudo semanage fcontext -a -t glusterd_brick_t /rhgs/brick_xvde
```
```bash
sudo restorecon -Rv /rhgs/brick_xvde
```

```bash
lsblk
```
```
...
xvde                              202:64   0  10G  0 disk 
├─rhgs_vg4-rhgs_thinpool4_tmeta   253:15   0  12M  0 lvm                                                                                                                     
│ └─rhgs_vg4-rhgs_thinpool4-tpool 253:17   0  10G  0 lvm                                                                                                                     
│   ├─rhgs_vg4-rhgs_thinpool4     253:18   0  10G  0 lvm                                                                                                                     
│   └─rhgs_vg4-rhgs_lv4           253:19   0  10G  0 lvm  /rhgs/brick_xvde                                                                                                   
└─rhgs_vg4-rhgs_thinpool4_tdata   253:16   0  10G  0 lvm                                                                                                                     
  └─rhgs_vg4-rhgs_thinpool4-tpool 253:17   0  10G  0 lvm                                                                                                                     
      ├─rhgs_vg4-rhgs_thinpool4     253:18   0  10G  0 lvm                                                                                                                     
          └─rhgs_vg4-rhgs_lv4           253:19   0  10G  0 lvm  /rhgs/brick_xvde 
...
```

Now that the LVM layout has been set up and a filesystem has been created on the new brick, we can replace the faulty one.
Log out from **rhgs2**
```bash
exit
```

Back on **rhgs1** we need to tell gluster to replace the faulty brick with the newly created one

```bash
sudo gluster volume replace-brick repvol rhgs2:/rhgs/brick_xvdc/repvol rhgs2:/rhgs/brick_xvde/repvol commit force
```

```
volume replace-brick: success: replace-brick commit force operation successful                                                                                               
```

Check the volume status to see if the new brick is in use
```
sudo gluster volume status repvol
```

```
Status of volume: repvol                                                                                                                                                     
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick rhgs1:/rhgs/brick_xvdc/repvol         49152     0          Y       12946
Brick rhgs2:/rhgs/brick_xvde/repvol         49152     0          Y       13210
Self-heal Daemon on localhost               N/A       N/A        Y       13341
Self-heal Daemon on rhgs2                   N/A       N/A        Y       13216
 
 Task Status of Volume repvol
 ------------------------------------------------------------------------------
 There are no active volume tasks
```

To verify that all the data has been replicated as well, we need to check the
contents of /rhgs/brick_xvde/repvol/mydir on **rhgs2**

```bash
ssh student@rhgs2
```
```bash
ls /rhgs/brick_xvde/repvol/mydir/file* | wc -l
```
```
100
```

So the files have been successfully replicated to the new replacement volume and
the cluster is back to normal operation.


