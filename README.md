# How to use it

## Ansible Setup

1. Install `>= ansible2.5`
    - I used `sudo pip install ansible` on Ubuntu Mate 18.04 and it worked
2. Fill in inventory file `prod` with ssh connection information to a server
    - If you need RSA key, or want to make one just for ansible, run ssh-keygen
    - Put your username and key file name in the `prod` file with the IP of your server
3. Fill out and encrypt the host_vars/vault.yml file
    - You will need to track down your PIA account details and make a NAS samba user for the media server. See below for how to make a new samba user.
    - Run ansible-vault encrypt on the vault.yml file...keep the password you used somewhere safe!
4. Create ~/.vault_pass.txt with password for decrypt during provision
    - This is just a file containing only your password you just made. It seems common that it goes into your home directory. `chmod` the permissions to 600, at least.
5. Make sure you have the angstwad.docker_ubuntu role in your `~/.ansible/roles/` path

        ansible-galaxy install angstwad.docker_ubuntu

6. run

        ansible-playbook -i prod site.yml --vault-password-file ~/.vault_pass.txt

    - --ask-sudo-pass also likely to be needed

__Ansible will now run, but a lot of things will be wrong__


## Fix the config files

In a few places, there are hardcoded usernames (andrew) and network addresses in 10.0.0.0/24. After discovering they existed I found all of them with
    
    find ./ -type f -exec bash -c "grep -i "andrew" {} && echo {} && echo " ';'

and

    find ./ -type f -exec bash -c "grep -i "10\.0" {} && echo {} && echo " ';'


The username needs to be changed in
- roles/ec3/tasks/main.yml: //10.0.0.62/data/home/andrew/docker should be set to an area on your NAS the containers can use
- roles/ec3/vars/main.yml: Change the NAS address, change the username you want made on the host (and the password too)
- prod and host_vars/vault.yml (but these have already been changed)


The network range/addresses need to be changed in
- roles/ec3/tasks/main.yml: As above, make the IP right for your NAS
- roles/ec3/tasks/transmission.yml: Change the subnet range of your local network in LOCAL_NETWORK
- roles/ec3/vars/main.yml: Change the NAS address



The default SMB mount will need its server address changed, but leaving it as /docker is easier as that's what all the docker containers expect in their exports. 
I also made another mount that mounts to a more general directory for me. Add the mount in roles/ec3/tasks/main.yml if you want it to be done automatically, otherwise modify the media server's fstab yourself. It will be something like:

    - name: Mount media on NAS
      mount: 
        state: "mounted" 
        fstype: "cifs" 
        opts: "username={{nas_user}},password={{nas_pass}},uid={{user.name}},gid={{user.name}},file_mode=0755,dir_mode=0755,exec" 
        src: //10.0.0.105/media/
        name: "/media/NAS"



In roles/ec3/tasks/transmission.yml, make volume exports so that the complete and incomplete dirs used by transmission are somewhere good (for me, this means both on my NAS).
 - /docker/incomplete:/data/incomplete
 - /docker/complete:/data/completed
 - Remember that volume exports are \<HOST_DIR\>:\<DIR_INSIDE_CONTAINER\> and it means that in the docker container, that directory is exactly what it is on the host.


I also had to  add a mount for the general NAS media directory in the Plex container
- /media/NAS:/data/nas under volumes: in roles/ec3/tasks/plex.yml


When testing transmission I ran into issues of it totally filling up the OS drive since it used it for the incomplete directory. However, another part of the problem is that the default transmission web GUI is very bare and does not let you only download parts of a torrent. After some searching I found that seemingly the only way to fix this is to use a [custom UI made by someone in China for a google summer of code](https://github.com/ronggang/transmission-web-control/wiki).
- Enter your transmission container with `sudo docker exec -it transmission /bin/bash` on the media server hosting your containers
- [Download the install script and run it, and when you refresh the transmission page you should have a much better UI](https://github.com/ronggang/transmission-web-control/wiki/Linux-Installation)

### Small notes: 
- Container configs are stored in /opt/amc (that's where the ansible/docker combo puts them)
- The PIA server is set to Sweden by default in transmission.yml. I didn't get any speed issues, but depending on your location your may want something more local (or not!)
- To reach transmission's control server, connect to \<media-server-ip-or-hostname\>:9091/transmission
- To reach plex, connect to \<media-server\>:32400



## Containers used

Below is a list of the containers used with links to their source. All credit to them, I stand on the shoulders of giants.

NOTE: Due to the way docker interacts with the file system, all static config directories should be on the docker host and not from a Network Drive. Media files are ok, but no config files.

### Plex

@linuxserver/docker-plex

Your friendly-neighborhood media player

### Transmission/Open-VPN

@haugene/docker-transmission-openvpn

A combination of a bit torrent client and openvpn client.

This will require you to have a VPN subscription. Highly recommend Private Internet Access www.privateinternetaccess.com.

### Radarr

@linuxserver/docker-radarr

An automation tool to track movies.

### Sonarr

@linuxserver/docker-sonarr

An automation tool to track TV

### Jackett

@linuxserver/docker-jackett

A proxy server that standardizes communication for sonarr and radarr to search.

### Heimdall

It seems the project used to use muximux but is uses heimdall now. You'll find it, once it's setup it's what you get from just a normal web connection to the server. 

## Other Stuff

### Glances

This playbook installs glances on the host and runs its web application as a service to help monitor OS metrics. Also keeps a list of active containers.

### .kitchen.yml file

It is useful to have a quick way to test the playbook and not change the server at all. Test-kitchen has a plugin to provision vagrant boxes with Ansible. Makes testing really easy.  

## Things to Fix

## Improvements

Simple nginx proxy server to route traffic between containers with pseudo-DNS names.

Add nginx reverse proxy to ansible setup so that transmission web page is accessible from outside the OS running the transmission container

Post system requirements.

## Legal Stuff

This code is posted exclusively for private use and should not be used for any illicit activity.
