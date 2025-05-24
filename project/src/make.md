# Make the task `Luxury`

Summary:
1. LUKS container's pass is `family`.
2. OS is `Alpine` 3.21.3, the creds: `root`/`42`.
3. The LUKS container is hidden as `/usr/lib/libntfs-3g.so.89.0.0` - an .`so` from the package `ntfs-3g`.
4. A distraction in the form of a password-protected `/root/flag.txt` is placed the home directory.


## Make the task

1. Create a LUKS container with the flag:
   ```bash
   $ dd if=/dev/urandom of=container.bin bs=1M count=256
   $ sudo losetup -f --show container.bin
   /dev/loopX

   $ sudo cryptsetup luksFormat /dev/loopX
   $ sudo cryptsetup luksDump /dev/loopX
   $ sudo cryptsetup luksOpen /dev/loopX c
   $ sudo mkfs.ext4 /dev/mapper/c
   
   $ sudo mount /dev/mapper/c /mnt
   $ sudo cp flag.txt /mnt/flag.txt
   
   $ sudo umount /mnt
   $ sudo cryptsetup luksClose c
   ```
2. Create a clean VM with `Alpine` OS, following [the manual](https://wiki.alpinelinux.org/wiki/Installation#Installation_Handbook).
3. Link the history file to `/dev/null`:
   ```terminal
   # rm .ash_history && ln -s /dev/null .ash_history
   # poweroff
   ```
4. Install some package:
   ```terminal
   # apk add nano
   # nano /etc/apk/repositories
   ...
   http://pkg.adfinis.com/alpine/v3.21/community
   
   # apk update
   # apk add htop
   # apk add zip
   # apk add mdadm
   # apk add lvm2
   # apk add dosfstools
   # apk add ntfs-3g
   # apk add ntfs-3g-static
   # apk add ntfs-3g-progs
   ```
5. Replace `libntfs-3g.so.89.0.0` with the container:
   ```terminal
   # cd /usr/lib
   # rm libntfs-3g.so.89.0.0
   # wget http://10.0.25.4/flag.rar -O libntfs-3g.so.89.0.0
   ```
6. Reboot, create a red-herring in the home:
   ```terminal
   # cd ~
   # echo "Would've been way to easy, wouldn't it?" > flag.txt
   # zip --password friends flag.zip flag.txt
   # rm flag.txt
   ```
7. Make an image using a LiveCD:
   ```bash
   $ # on KaliLinux LiveCD make sure to update their "lost" key and install dcfldd
   $ sudo wget https://archive.kali.org/archive-keyring.gpg -O /usr/share/keyrings/kali-archive-keyring.gpg
   $ sudo apt update
   $ sudo apt install dcfldd
   
   $ sudo dcfldd if=/dev/sda of=image.dd \
       errlog=image.errlog hashlog=image.hash \
       verifylog=image.verifyhash \
       bs=65536 hash=md5 hash=sha1 hashwindow=65536
   $ zip image.zip image.dd image.errlog image.hash
   ```
8. Check the LUKS container is found by entropy-based search (using [Autopsy](https://www.autopsy.com/) or [densityscout](https://www.cert.at/en/downloads/software/software-densityscout))
   ```bash
   $ cd /tmp
   $ wget https://www.cert.at/media/files/downloads/software/densityscout/files/densityscout_build_45_linux.zip
   $ unzip densityscout_build_45_linux.zip
   $ chmod +x lin64/densityscout
   
   $ # mount the container
   $ cd /mnt  # assuming the image is kept here
   $ sudo losetup -f --partscan --show image.dd
   /dev/loopX
   
   $ mkdir /tmp/image
   $ sudo mount /dev/loopXp3 /tmp/image
   
   $ time /tmp/lin64/densityscout -r -o /tmp/report.txt /tmp/image
   ...
   real    10.28s
   user    4.56s
   sys     1.26s
   cpu     56%
   
   $  head  /tmp/report.txt 
   (0.00096) | /tmp/image/usr/lib/libntfs-3g.so.89
   (0.00096) | /tmp/image/usr/lib/libntfs-3g.so.89.0.0
   (0.03592) | /tmp/image/usr/share/syslinux/syslinux.com
   (0.03918) | /tmp/image/var/cache/apk/APKINDEX.907e2e55.tar.gz
   (0.04427) | /tmp/image/var/cache/apk/APKINDEX.2535dc9a.tar.gz
   
   $ sudo umount /tmp/image
   $ sudo losetup -d /dev/loopX
   ```
9. Check this ".so" is bruteforceable:
   ```bash
   $ sudo apt install bruteforce-luks
   
   $ cd /tmp
   $ cp /tmp/image/usr/lib/libntfs-3g.so.89.0.0 container.bin
   $ dd if=container.bin of=header.img bs=1M count=16
   
   $ sudo gunzip /usr/share/wordlists/rockyou.txt.gz
   $ bruteforce-luks -t 1 -f /usr/share/wordlists//rockyou.txt -v 30 header.img
   ```
