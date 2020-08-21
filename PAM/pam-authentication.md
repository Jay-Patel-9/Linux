# PAM Authentication 

## Formal Introduction
PAM stands for Pluggable Authentication Modules and it was first proposed by Sun Microsystems in an Open Software Foundation Request for Comments (RFC) 86.0. PAM first appeared in Red Hat Linux 3.0.4 in August 1996 in the Linux PAM project. PAM is currently supported in the AIX operating system, DragonFly BSD,FreeBSD, HP-UX, Linux, macOS, NetBSD and Solaris.

#### In simple words
Whenever you ask system to make authentication, the request is processed by PAM; The software program which acts as a mediator between OS API and the application to dynamically perform authentication and authorization activities, PAM reads the config file defiened by the application(ex: login) and validates the details based results it allow authentication or logs an error message.

------
Before PAM applications were solely relied on /etc/passwd and /etc/shadow file to validate the user authentication in other simple words (anyone with valid user and password) was in whitelist. To make any change in authentication was required source code change in the application.

After PAM, applications stopped direct authentication with OS and asked PAM to perform and validate the authentication.

![alt](https://github.com/Jay-Patel-9/Linux/blob/master/PAM/before-after-pam.png)

Now you might have the basic idea of PAM in the picture. in simple words, PAM provides common authentication scheme with great flexibility and control, which can be used with variety of applications.

#### To check whether the application is PAM Aware use following command:
> ldd /usr/bin/login | grep libpam

![pam-aware-application](https://github.com/Jay-Patel-9/Linux/blob/master/PAM/ldd-example.png)

## How PAM is working?

PAM provides validation support over following mechanism:

1. Authentication
2. Account
3. Password
4. Session

### PAM control flags
|Flag |Description |
|---|---|
|Required | Result must be successful for authentication to continue. If the test fails at this point, the user is not notified until the results of all module tests that reference that interface are complete.|
|Requisite | Result must be successful for authentication to continue. However, if a test fails at this point, the user is notified immediately with a message reflecting the first failed required or requisite module test. |
|Sufficient |Result is ignored if it fails. However, if the result of a module flagged sufficient is successful and no previous modules flagged required have failed, then no other results are required and the user is authenticated to the service.|
|Optional |Result is ignored. A module flagged as optional only becomes necessary for successful authentication when no other modules reference the interface. |
|Include | This flag pulls in all lines in the configuration file which match the given parameter and appends them as an argument to the module.|
|Substack | Include all lines of give type from the config file but evolution of done and die does not cause skipping the rest of the modules. |

### PAM files on system
 1. /etc/pam.conf - PAM Configuration directory
 2. /etc/pam.d/ - Linux PAM configuration directory, If this directory exists then /etc/pam.conf will be ignored.
----------
###### **PAM has enormous control over authentication mechanism in the System, Editing or removing files from /etc/pam.d/ may permanent lock your system. If you're doing it just for testing it is recommended to try in VirtualBox or Docker environment.**
----------
### PAM Config example
| module_interface |     control_flag |  module_name module_arguments |
|---|---|---|

example:
|auth|sufficient|pam_rootok.so|
|---|---|---|

> Get detail about any PAM module using: man {pam_module}. Ex: `man pam_rootok`

#### PAM config file for "SU" binary (/etc/pam.d/su) :

```python
# %PAM-1.0
auth            sufficient      pam_rootok.so
# Uncomment the following line to implicitly trust users in the "wheel" group.
#auth           sufficient      pam_wheel.so trust use_uid
# Uncomment the following line to require a user to be in the "wheel" group.
#auth           required        pam_wheel.so use_uid
auth            substack        system-auth
auth            include         postlogin
account         sufficient      pam_succeed_if.so uid = 0 use_uid quiet
account         include         system-auth
password        include         system-auth
session         include         system-auth
session         include         postlogin
session         optional        pam_xauth.so
```

###### Simplified config

```python
# %PAM-1.0 # PAM Version
Module          Control_flag    Module_argument
----------------------------------------------
auth            sufficient      pam_rootok.so # Checking for UID=0
# Uncomment the following line to implicitly trust users in the "wheel" group.
#auth           sufficient      pam_wheel.so trust use_uid # Allow every user resides in wheel group.
# Uncomment the following line to require a user to be in the "wheel" group.
#auth           required        pam_wheel.so use_uid #User must be in wheel group to succeed in authentication.
auth            substack        system-auth # Provides authentication libraries
auth            include         postlogin 
account         sufficient      pam_succeed_if.so uid = 0 use_uid quiet # succeed or fail the authentication based on characteristic of user account or based on the values of other PAM items. 
account         include         system-auth 
password        include         system-auth
session         include         system-auth
session         include         postlogin # Providing pam modules which are required after authentication.
session         optional        pam_xauth.so #Forwards xauth keys (Also referred to as "cookies") between users.
```

![pam-config-flow](https://github.com/Jay-Patel-9/Linux/blob/master/PAM/pam-config-flow.png)


#### Handling exception with pam.d/others
##### What happens when PAM doesn't find the config file for application in /etc/pam.d/?
All the files in /etc/pam.d/ contains the configurations for specific application or service, The exception(when PAM didn't file config for application) is handled by /etc/pam.d/other. This file contains the configuration for any services which don't have their on configuration file.
For instance: if the service abc attempted authentication, PAM will look for /etc/pam.d/abc file. failure to find that config the authentication for service abc will be determined by /etc/pam.d/others file.

/etc/pam.d/others has two secure configurations one is paranoid and another is quite gentle.

1. Paranoid (/etc/pam.d/other)

```python
auth        required	pam_deny.so
auth        required	pam_warn.so
account     required	pam_deny.so
account     required	pam_warn.so
password    required	pam_deny.so
password    required	pam_warn.so
session	    required	pam_deny.so
session	    required	pam_warn.so

```
- pam_deny.so is used to deny the access using PAM framework. 
- pam_warn.so is logs user,service, terminal and remote host to syslog, it always returns PAM_IGNORE which means that it doesn't make any impact on authentication process.

With this configuration when any application tried to make authentication PAM deny the request using pam_deny module and then logs a syslog using pam_warn module.

2. Gentle ( /etc/pam.d/other)
```python
auth   	  required	pam_unix.so
auth      required	pam_warn.so
account	  required	pam_unix.so
account	  required	pam_warn.so
password	 required	pam_deny.so
password	 required	pam_warn.so
session	  required	pam_unix.so
session	  required	pam_warn.so
```
- pam_unix.so is uses a traditional authentication by using system's library to retrieve and set account information and authentication. Usually (/etc/passwd & /etc/shadow).

This configuration will allow unkown service to authenticate using pam_unix.so module, However it will not allow user to change password. It also logs entry to syslog whenever a service attempts authentication.

#### PAM on OS Hardening
PAM plays vital role when it comes to User Authentication and Management. PAM can be configured in following ways to enhance Operating System's security.

1. Password creation requirements. (Password complexity)
2. Lockout user after x number of failed attempts.
3. Password reuse limitation.
4. Strong password hashing algorithm.
5. Write own PAM module for [custom](http://www.linux-pam.org/Linux-PAM-html/Linux-PAM_MWG.html) requirements.

Reference: [Configure PAM for OS Hardening](http://www.itsecure.hu/library/image/CIS_Red_Hat_Enterprise_Linux_7_Benchmark_v2.1.1.pdf)
