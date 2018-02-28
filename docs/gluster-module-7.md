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


On **rhgs1** is an ansible script to take care of the required prerequisites
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
``
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
``


## SIMULATE BRICK FAILURE

Now that we have a healthy replicated volume, we will simulate a brick failure.

On **rhgs2** we will now disable the brick:

```bash
sudo dmsetup load rhgs_vg2-rhgs_lv2 /home/student/materials/dmsetup-error-target
sudo dmsetup resume rhgs_vg2-rhgs_lv2
```

This will remove the inital blockdevice to which the map points and replace it with the error target. As a consequence, /dev/xvdc  will return an I/O error on every read- or write operation to it. 

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

If you follow the system on **rhgs2** logs using
```bash
sudo tail -f /var/log/messages
```

you can see the file system being shutdown due to errors shortly after
``
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): metadata I/O error: block 0x9f1e00 ("xlog_iodone") error 5 numblks 256
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): metadata I/O error: block 0x0 ("xfs_buf_iodone_callback_error") error 5 numblks 1
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): xfs_do_force_shutdown(0x2) called from line 1200 of file fs/xfs/xfs_log.c.  Return address = 0xffffffffc02dfec0 
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): Log I/O Error Detected.  Shutting down filesystem
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): Please umount the filesystem and rectify the problem(s)
Feb 16 06:44:00 ip-172-31-25-204 kernel: XFS (dm-14): Failing async write on buffer block 0xffffffffffffffff. Retrying async write.   
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

## REPLACE THE FAULTY BRICK

We need to set up the new brick in the identical way the original one was set
up.

```bash
sudo pvcreate /dev/xvdd
```
``  Physical volume "/dev/xvdd" successfully created.``

```bash
sudo vgcreate rhgs_vg3 /dev/xvdd
```
``  Volume group "rhgs_vg3" successfully created``

```bash
sudo lvcreate -l 100%FREE -T rhgs_vg3/rhgs_thinpool3
```
``  Using default stripesize 64.00 KiB.                                                                                                           
  Thin pool volume with chunk size 64.00 KiB can address at most 15.81 TiB of
  data.
  Logical volume "rhgs_thinpool3" created.``

```bash
sudo lvchange --zero n rhgs_vg3/rhgs_thinpool3
```
``  Logical volume rhgs_vg3/rhgs_thinpool3 changed.``

```bash
sudo lvcreate -V 10G -T rhgs_vg3/rhgs_thinpool3 -n rhgs_lv3
```
``  WARNING: Sum of all thin volume sizes (10.00 GiB) exceeds the size of thin pool rhgs_vg3/rhgs_thinpool3 and the size of whole volume group (<10.00 GiB)!
  For thin pool auto extension activation/thin_pool_autoextend_threshold should be below 100.
  Logical volume "rhgs_lv3" created.``

```bash
sudo mkfs.xfs -i size=512 -n size=8192 /dev/rhgs_vg3/rhgs_lv3
```
``
meta-data=/dev/rhgs_vg3/rhgs_lv3 isize=512    agcount=16, agsize=163824 blks                                                                      
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2621184, imaxpct=25
         =                       sunit=16     swidth=16 blks
naming   =version 2              bsize=8192   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
``
```bash
sudo mkdir -p /rhgs/brick_xvdd
```
```bash
sudo echo "/dev/rhgs_vg3/rhgs_lv3 /rhgs/brick_xvdd xfs rw,inode64,noatime,nouuid 1 2" >> /etc/fstab
sudo mount /rhgs/brick_xvdd
sudo semanage fcontext -a -t glusterd_brick_t /rhgs/brick_xvdd
sudo restorecon -Rv /rhgs/brick_xvdd
```



The device xvdd on rhgs2 has been prepared identically to the xvdc, so we can
use it as a replacement for the faulty brick.

```bash
lsblk
```
``
...
xvdd                              202:48   0  10G  0 disk 
├─rhgs_vg3-rhgs_thinpool3_tmeta   253:5    0  12M  0 lvm  
│ └─rhgs_vg3-rhgs_thinpool3-tpool 253:7    0  10G  0 lvm  
│   ├─rhgs_vg3-rhgs_thinpool3     253:8    0  10G  0 lvm  
│   └─rhgs_vg3-rhgs_lv3           253:9    0  10G  0 lvm  /rhgs/brick_xvdd
└─rhgs_vg3-rhgs_thinpool3_tdata   253:6    0  10G  0 lvm  
  └─rhgs_vg3-rhgs_thinpool3-tpool 253:7    0  10G  0 lvm  
      ├─rhgs_vg3-rhgs_thinpool3     253:8    0  10G  0 lvm  
          └─rhgs_vg3-rhgs_lv3           253:9    0  10G  0 lvm  /rhgs/brick_xvdd
``

With that structure on the new brick, we can now prepare it for use
```bash




**rhgs1**

```bash
gluster volume replace-brick test-volume server0:/rhgs/brick1
server5:/rhgs/brick1 commit forc





