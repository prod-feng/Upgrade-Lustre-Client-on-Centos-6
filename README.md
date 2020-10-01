# Upgrade-Lustre-Client-on-Centos-6

The following is my personal experience, use at one's own risk.

I have a legacy server which runs Centos 6.4 and Lustre 1.8.9. Recently, to be able to mount our new on production Lustre strorage on it, it is unavoidable that we have to upgrade this server to a much newer Lustre client version. I planned to upgrade the Lustre client kernel module only, first, because it had KMOD installed already, also our legacy apps can still run instead of upgrading the whole OS.

I found the source rpm files from https://downloads.whamcloud.com/public/lustre/. It seems to me that version lustre-2.10.8 is the most recent version which supports Centos 6(comes with pre-compiled binary rpms). I decided to compile the Lustre client from source:


**(1). Download and compile the Lustre source code to local server.**

```[feng@server1 ~]$ wget  --no-check-certificate https://downloads.whamcloud.com/public/lustre/lustre-2.10.8/el6/client/SRPMS/lustre-2.10.8-1.src.rpm```

```[feng@server1 ~]$ mkdir lustre_client```

```[feng@server1 ~]$ cd lustre_client```

```[feng@server1 ~]$ rpm2cpio   ../lustre-2.10.8-1.src.rpm | cpio -idmv```

```[feng@server1 ~]$ ls -l```
>```-rw-rw-r-- 1 feng feng    17867 May 26  2019 lustre.spec```

>```-rw-rw-r-- 1 feng feng 13623600 May 26  2019 lustre-2.10.8.tar.gz```

>```......```

To compile the source code, I needed to install some extra packages, the compiling process propmted those usefull information for referencing.

I first tried to use rpmbuild version 4.8.0-59. It seems any earlier rpmbuild version can not work. I decided to run rpmbuild as a regular user to compile the code, instead of running it as an admin, in case it may cause mess. To do this, I need to set the rpm macro of "~/.rpmmacros", with something like a line of "%_topdir    /local/feng/rpm". There I have enough local disk space, not NFS which I heard may cause issue.

The rpmbuild process did not go well:

```[feng@server1 ~]$ rpmbuild -ta lustre-2.8.0.tar.gz ```

the compilation failed at install stage, which complains a file/folder can not be found. Then, I decided to go straight and compile manually:

```[feng@server1 ~]$ cd /local/feng/rpm/BUILD/lustre-2.10.8/```

```[feng@server1 ~]$ make rpms```

and it worked pretty smooth. The compiled rpm files are there:

>```-rw-rw-r-- 1 feng feng   552228 Sep 29 14:18 lustre-client-2.10.8-1.el6.x86_64.rpm```

>```-rw-rw-r-- 1 feng feng  2203280 Sep 29 14:18 kmod-lustre-client-2.10.8-1.el6.x86_64.rpm```

>```......```

OK, it worked. The direct rpmbuild process I used which failed seems caused by the ```lustre.spec``` file. There seems to be a workaround to modify the ```lustre.spec``` file, like add one line of ```"make rpms``` in it, just before the ```%install``` line, then use ```rpmbuild -tc ``` or ```rpmbuild -bc ``` accordingly. Or there may be some other ways to make it work that I just missed. When tried to modify ```lustre.spec``` file, I just realized there are actualy 2 of it in total: one is with the tarball file; the other one is inside the tarball.

**(2) Stop and remove the old Lustre client**

This stage needs to be extreamly careful if there are mounted Lustre storage(s) on the local server. To work on this stage, I need to use admin account.

First make sure nobody is using the mounted Lustre storage, if so, unmount all the mounted Lustre storage.

```[root@server1 ~]# umount /mnt/myluster```

Once all luster storages on the local server have been unmounted, I can work on the next step.

**(2.A) Unload the old Lustre client module from the kernel:**

```[root@server1 ~]# lustre_rmmod```

Then make sure there is no Lustre cleint module there in the OS kernel.

```[root@server1 ~]# lsmod |grep -i lustre```

**(2.B) Remove the old Lustre client packages.**

Since I used rpm to install the old Lustre client, so I use rpm tool to remove them.

```[root@server1 ~]# rpm -e lustre-client-modules-1.8.9 lustre-client-1.8.9```

At this tep, I got an error message complaining: ```" error: %preun(lustre-client-modules-1.8.9-wc1_*.x86_64) scriptlet failed, exit status 1"```

So I had to force rpm to remove it anyway:

```[root@server1 ~]# rpm --noscripts -e lustre-client-modules-1.8.9```

After that, make sure the old Lustre client files are cleanly removed, expecially the kernel module files, like the ```"lustre.ko"```. They are always in ```/lib/modules/{mykenel}/``` folder, I can use ```find /lib/modules/{mykenel}/ -name lustre.ko```, etc, to verify if they are still there.

If the old Lustre client was installed without rpm tool, I probably would have to manually remove all of these old Lustre files.

Once this is done, I now have a clean OS without Luster client. And it's time to install the new one that I just have compiled.

**(3) Install the new Lustre Client**

This step is straightforward.

```[root@server1 ~]# rpm -ivh lustre-client-2.10.8-1.el6.x86_64.rpm kmod-lustre-client-2.10.8-1.el6.x86_64.rpm```

If it works without any error, then add the new Lustre client module into the OS kernel:

```[root@server1 ~]# modprobe lustre```

Now we are almost done. Make sure the Luster client module is there:


```[root@server1 ~]# lsmod |grep -i lustre```

**(4) Mount the Lustre storage on the local server.**


```[root@server1 ~]# mount /mnt/myluster```

Now I have my new Lustre Client and mounted storage on the local server.


