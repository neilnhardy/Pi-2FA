# Pi-2FA

Bash scripts supporting 2-factor authentication (2FA) for a range of services on Pi OS. 

To access a 2FA supported service a user has to first enter their system password, then a 6-digit TOTP (Time-based One-Time Password) which is  typically generated by a smartphone based app such as 'Authy' or 'Google Authenticator'.

Baseline support is provided for `lightdm`, `login`, `sshd`, `sudo` and `su`, however other `/etc/pam.d` services can be easily added. 

By default, each user has individual control of which services are 2FA enabled, however a service can be configured so that 2FA is enforced for all users.

##  Quick Start

### Administrator 
1. Clone this repository. 
2. From the repo directory execute `sudo ./2fa-install`.

_Optional_
- Execute `sudo 2fa-svc-enf` to manage which services have 2FA enforced and view which users have yet to run `2fa-register`
- Execute `sudo 2fa-add-service  <service>` to add a service to be managed by Pi-2FA scripts.
- Execute `sudo 2fa-del-service  <service>` to delete a service from being managed by Pi-2FA scripts.

### User
1. Ensure you have a TOTP generator app (e.g. 'Authy' or 'Google Authenticator') loaded on your mobile device.
2. Execute `2fa-svc-mng`. (Note: `2fa-install` adds this command to each /home user's `.bashrc` in order to encourage them to register).
- Initially this will call `2fa-register` which will guide you through the process of creating a barcode that can be scanned by your TOTP generator app, thus generating TOTP's every 30s.
-  Provided `2fa-register` completes successfully, you will progress to selecting which services are 2FA enabled
3. `2fa-svc-mng` can be executed subsequently to manage which services are Enabled/Disabled.
4. `2fa-register` can be executed subsequently to create a new private key. This might be considered necessary after losing/changing your mobile device.

### Help
For a more detailed description of the above, or any other scripts, execute the script with a single -h option. e.g:
`2fa-svc-mng -h`
`2fa-add-service -h`


## How it works

Each 2FA supported service configuration file in `/etc/pam.d` has additional config inserted by `2fa-add-service` after the line that enables system password entry. When a user authenticates to a 2FA supported service this additional config determines if their `$USER` name is present in the relevant `/etc/2fa/<service>-users` file. If present, after entering their password, the user is challenged to enter a 6-digit verification code which should be obtained from the user's smartphone TOTP generator app, i.e 2FA is applied.

The `2fa-svc-mng` script is executed by a user to view/change which services require 2FA. Enabling a service results in the service name being added to the user's `$HOME/.2fa-services` file. Disabling a service results in the service name being removed from the user's `$HOME/.2fa-services` file.

When a user's `$HOME/.2fa-services` file is changed, the systemd `2fa-svc.service`, executing `2fa-svc-mon` as root, updates each `/etc/2fa/<service>-users` file, ensuring the user's $USER name is present where the user has 2FA enabled,  and is not present where 2FA is disabled.

The systemd `2fa-svc.service` detects changes in users preferences by using a list of `$HOME/.2fa-services` paths in `/etc/2fa/user-svc-files`. The systemd `2fa-new-user.service` detects an addition to the `/home`  directory (typically as a result of `adduser`) then creates a `$HOME/.2fa-services` file for the new user and adds its file path to `/etc/2fa/user-svc-files`

2FA can be enforced for a service, overriding individual user control.  `sudo 2fa-svc-enf` manages service enforcement. When a service is enforced  the service configuration in `/etc/pam.d` is altered so that the verification code challenge is issued immediately after entering the system password. When enforcement is disabled the service configuration in `/etc/pam.d` is reverted to individual user control.

## Notes   
- Users that have yet to run `2fa-register` will **bypass enforcement of services.** This is necessary as they may require access to an enforced service to be able to register and hence create a TOTP. Administrators can identify these users by running `sudo 2fa-svc-enf` and if required, remove their bypass by deleting their usernames from `/etc/2fa/except_usrs`. It should be understood that this will deny access to all enforced service(s) for those users. 

- Before executing  `sudo 2fa-add-service` it is strongly recommended that the `/etc/pam.d/<service>` file be reviewed. In some cases there are explicit warnings against editing the file.


## Acknowledgements

This project was inspired by the following articles: 

- https://www.raspberrypi.org/blog/setting-up-two-factor-authentication-on-your-raspberry-pi/
- https://dopensource.com/2016/12/30/scripting-one-time-ssh-access-to-allow-for-google-authenticator-setup/
- https://askubuntu.com/questions/819265/bash-script-to-monitor-file-change-and-execute-command

