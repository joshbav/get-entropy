## OVERVIEW
This was developed as a POC to solve for a lack of entropy for a customer's IoT devices.  
Basically they use inexpestive hardware, the CPU has no random number generator instruction.  
Also, the filesystem is mounted read only, so a previous random seed couldn't be stored on disk.  
Their security team said they must solve the lack of entropy on startup, so crypto functions would be secure.  
I questioned why a lot entropy was needed immediately, but decided to just focus on the solution instead of changing their minds. 
Why wasn't /dev/urandom good enough? I decided not to push the issue.  
  
I made this systemd unit to address this, as proof of concept.  
It could be adapted to the ARM architecture, and systemd isn't a requirement.  
  
## TO SEE THE PROBLEM  
1. Boot up a Linux VM.  
2. cd /tmp  
3. sudo su  
4. git clone https://github.com/joshbav/get-entropy.git  
5. cd get-entropy  
4. (Empty the Kernel's existing entropy pool) cp /dev/random /tmp/deleteme  
5. The previous command will appear to hang, because /dev/random blocks when it runs out of entropy. So break out of it.  
6. (See the low entropy value, anything less than 100 is bad) cat /proc/sys/kernel/random/entropy_avail  
7. Wait 10 seconds.  
8. (See that the entropy is still low) cat /proc/sys/kernel/random/entropy_avail  
  
The use case this was built for is such that we supposedly need the entropy pool filled ASAP at startup.  Since the filesystem is readonly, there's no way to save a random seed to be used for the next boot.  
  
## TO SEE IT SOLVED  
  
I used Ryan Finnie's [rndaddentropy](https://github.com/rfinnie/twuewand/blob/master/rndaddentropy/rndaddentropy.c). It directly adds to the kernel entropy pool, because copying to /dev/random just stirs the pool and does not increase the entropy value.  
  
To see how the rndaddentropy utility fixes this problem:  
1. (Create a 5k psudorandom file) dd if=/dev/urandom of=sample.txt bs=5k count=1
2. Repeat the previous section's steps 6-8, to drain the entropy.  
3. (Fill the entropy pool) cat sample.txt | ./rndaddentropy  
4. (Check the entropy pool) cat /proc/sys/kernel/random/entropy_avail  
  
So the goal is to fill the pool during startup using random data. This systemd unit will fetch entropy over the Internet. It will also use its network mac addresses, boot_id, and NTP time statistics as sources, so if a man-in-the-middle network attack occurs they ought to have a hard time.  
  
## INSTALLATION
Clone the repo and cd to it, as shown in previous section  
(as root)
1. sudo yum install -y chronyd 
2. (use my chrony.conf file which will cause chrony to pause on startup until NTP is in sync) curl -o /etc/chrony.conf -sSLO https://raw.githubusercontent.com/joshbav/lab-aws-template/master/chrony.conf  
3. sudo yum install -y jq  
4. cp rndaddentropy /sbin  
5. chmod 755 /sbin/rndaddentropy  
6. (Rebuild the shell's command hash table) hash -r  
7. cp get-entropy.service /etc/systemd/system  
7. systemctl daemon-reload  
8. systemctl enable get-entropy  
10. reboot  
11. (Verify it ran) journalctl -u get-entropy --no-pager
  
