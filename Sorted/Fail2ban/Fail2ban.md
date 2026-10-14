**Fail2ban** is a security tool/service for Linux systems commonly used by system admins to protect devices from brute-force attempts. The software monitors logs on the host systems and automatically prevents further access attempts if logs reveal several failed attempts at access. Fail2ban is typically used on servers to harden SSH access but at its core it can lock down any service that writes a log file if configured to do so.

## How it works

- Fail2ban primary operates via a background server/daemon which tracks log files such as `/var/log/auth.log`

- The software filters logs for any pattern that resemble malicious activity such as repeated failed password attempts

- After a set amount of failed attempts the IP address of the machine that attempted to access will be blocked via iptables, nftables or firewalld.

- Fail2ban will by default only temporarily ban the assaulting IP but it can be configured to permanently ban said IP. This ban process is known as **Jailing**.

## Setup and Usage

#### Install and enable

- Firstly we need to install fail2ban via our respective package manger.

 Ubuntu and Debian
`sudo apt update`
`sudo apt install fail2ban -y`

Arch Linux
`sudo pacman -Rns`
`sudo pacman -S fail2ban`

- Next we need to enable the fail2ban service so it starts on machine boot

`sudo systemctl enable --now fail2ban` -> this will enable the service to launch on boot as well as start it install with the `--now` flag

we can check whether the service began by entering `sudo systemctl status fail2ban`
![[fail2ban_enable.png]]

#### Configuration 

When it comes to comes to defining the rules fail2ban follows we must first take note of its configuration files and where they live.

|                  File                  | Use                                                                                                                                                                                                                                              |
| :------------------------------------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|     `/etc/fail2ban/fail2ban.conf`      | The main configuration file that handles the bulk of the software such as the log it reads and settings                                                                                                                                          |
|   <br>`/etc/fail2ban/fail2ban.local`   | Intended for user defined configurations. We use `.local` files as a way to prevent fail2ban from returning to default settings after an update                                                                                                  |
|       `/etc/fail2ban/jail.conf`        | The main jail config that  handles jail definitions and default ban behavior                                                                                                                                                                     |
|       `/etc/fail2ban/jail.local`       | User defined Jail configurations.                                                                                                                                                                                                                |
| <br><br>`/etc/fail2ban/jail.d/*.local` | Special files stored in fail2bans jail directory. These hold service specific jail overrides. For exampled `/etc/fail2ban/jail.d/sshd.local` could be used to define the jail conditions for ssh specifically separate from other jail settings. |
Note that it is recommended to use `.local files` over fail2bas `.conf` files to define our setup. We can customize our fail2ban service by doing the following.

- `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local` -> This will copy all default parameters in our `.conf` to our new `.local` file.

- After wards we can edit `etc/fail2ban/jail.local` with a text editor like `nano` or `vim`
  
- A decent set up goes as follows though variables can be adjusted 

![[fail2ban_jaillocal.png]]

#### Jails
After enough failed login attemps fail2ban will flat out refuse to allow futher attempts by an IP to gain access by banning that specifc IP in a process call **"Jailing"**. Based on the variables set in the `jail.local` fail2ban will take different approaches when it comes to banning an IP but when the ban takes place it is rather noticable via the stall when attemping to log in. After the delay, fail2ban will deny futher attempts but this does not mean the service, ssh in this example, is disabled but rather our IP has been blacklisted from attempting to log into ssh.

![[fail2ban_ban.png]]

In order to see our active jails we can run `sudo fail2ban status` command which will pull up all defined jails in the example below only ssh has been configured as seen our `jail.local` file.

![[fail2ban_status.png]]

#### Manual Unbanning and Banning

Typically we would have to rely on our config file to automatically free an IP from jail but we can do so manually by first checking what IPs are banned.  `sudo fail2ban-client status *jail_name*` is the command used to check the status of a specifc jail and all IPs it banned alongside other facts. If a IP resides in the banned IP list we can free it by entering `sudo fail2ban-client set *jail_name* unbanip *IP_address*` alternatively we ban an ip by swapping out `unbanip` -> `banip` to added it to the banned IP list.

![[fail2ban_status_ssh.png]]

#### Logs

Fail2ban stores its logging information at `/var/log/fail2ban.log`  which we can read to track all prior actions taken by the service. We may also follow live updates to fail2ban by running the comman `sudo tail -f /var/log/fail2ban.log` to follow new lines as they are added.

