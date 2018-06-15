---
published: false
---
# Adding Linux to a Windows Domain

I have created the following script to automatically join a Linux (Centos/RHEL 6/7 system to a Windows Domain)


##Notes
Please Update line 41 with the correct OU to place the computer.

1. This script will update the computer
2. This script will install all required software
3. This script will ask you for a hostname. if you dont wish to change the hostname please enter the same thing as previous.



```bash
#!/bin/bash
#Made for Stormont Vail
#Please Update the OU on Line 41
#Create by Casey Parman
#Its probably Broken, so good luck.

#Creating and running the script

#vim domain.sh
# chmod +x domain.sh 
# ./domain.sh 

if [[ $(id -u) -ne 0 ]]
  then echo "Please run as root"
  exit 10
else 
  #Install Required Software
  yum -y install realmd samba samba-common oddjob oddjob-mkhomedir sssd ntpdate ntp
  #Setup NTP
  systemctl enable ntpd.service 
  ntpdate  time.stormontvail.org
  systemctl restart ntpd.service 

  echo -n "Enter Hostname:"
  read HNAME
  echo "Updating Hostname"
  hostname $HNAME
  sed -i -e 's/.*/'$HNAME'/g' /etc/hostname
  hostnamectl set-hostname $HNAME
  hostnamectl status


  #Joining Domain
  echo "Checking Domain"
  grep stormontvail.org /etc/sssd/sssd.conf > /dev/null 2>&1
  if [ $? != 0 ]
  then
    echo "Joining Domain"
    echo -n "Enter Domain Admin Account:"
    read DNAME
    realm join -U $DNAME -v --computer-ou="Update This will the FULL OU Leave QUOTES" stormontvail.org
    if [ $? = 0 ]
    then
      echo "Computer Joined to Domain"
      sed -i -e 's/\/home\/%u@%d"/"\/home\/%u/g' /etc/sssd/sssd.conf
      sed -i -e 's/use_fully_qualified_names = True/use_fully_qualified_names = False/g' /etc/sssd/sssd.conf
      echo "ldap_search_base = OU=IS Dept,OU=_Users,DC=stormontvail,DC=org" >> /etc/sssd/sssd.conf
      echo "ldap_group_search_base = OU=Linux,OU=Administration,OU=Security Groups,DC=stormontvail,DC=org" >> /etc/sssd/sssd.conf
      echo "AllowGroups linuxadmins" >> /etc/ssh/sshd_config
      systemctl restart sshd
      ##SUDO Changes## #Currently disabled because it shouldnt be required.  Next 9 lines commented out 
      #cp /etc/sudoers /etc/sudoers.new
      #echo "##Allow AD Admins To sudo" >> /etc/sudoers.new
      #echo "%linuxadmins  ALL=(ALL)     ALL" >> /etc/sudoers.new
      #visudo -q -c -s -f /etc/sudoers.new
      #if [ $? = 0 ]
      #then
      #  mv /etc/sudoers /etc/sudoers.old
      #  mv /etc/sudoers.new /etc/sudoers
      #fi
      systemctl restart sssd
    else
      echo $?
      echo "Could not Join domain, Continue Manually"
      exit 1
    fi
    systemctl restart sssd
  else
    echo "Already Joined to Stormontvail.org"
  fi


  #updating system
  echo "updating system"
  yum -y install cifs-utils.x86_64
  yum -y install sysstat 
  yum -y update
fi

exit 0
```

~~~Enjoy~~~
