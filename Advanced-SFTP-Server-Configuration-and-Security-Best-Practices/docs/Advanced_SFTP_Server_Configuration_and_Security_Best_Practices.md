# Secure SSH & SFTP Chroot Environment Project (Advanced Technical Documentation)

## 1. Project Description
This project demonstrates how to configure a secure multi-user SSH/SFTP environment on Linux using:
- User and group management
- Chroot isolation
- SFTP-only access
- Bind mounts
- Advanced permissions (ACL, setgid, sticky bit, chattr)

The goal is to simulate a real-world enterprise scenario where different users have different access levels and all users can access a shared directory automatically according to group membership.

---

## 2. Scenario / Objective

### Groups
- **shellgroup**  
  Users allowed SFTP access inside a chroot environment.

- **sftpgroup**  
  Users restricted to SFTP-only access inside a chroot environment.

- **admingroup**  
  Administrative users with full system access (no chroot).

### Users
- `user1` → member of `shellgroup`
- `user2` → member of `sftpgroup`
- `user3` → member of `admingroup`

### Security Goals
- Restrict SFTP users to `/sftp/%u`
- Prevent access outside chroot
- Share data securely using a single shared directory with automatic access for all group members
- Keep admin users unrestricted

---

## 3. Chroot Concept (Explanation)
`chroot` is an isolation mechanism that limits a user or process to a specific directory tree, preventing access to the system’s real root (`/`).

**Important SSH Rule:**  
The `ChrootDirectory` must be owned by `root:root` and must not be writable by any other user, otherwise SSH will refuse the connection.

---

## 4. Create Groups
sudo groupadd --gid 1001 shellgroup
sudo groupadd --gid 1002 sftpgroup
sudo groupadd --gid 1003 admingroup

## 5. Create Uusers
sudo useradd -m --uid 2001 user1
sudo useradd -m --uid 2002 -s /usr/sbin/nologin user2
sudo useradd -m --uid 2000 user3

## Set Passwords
sudo passwd user1
sudo passwd user2
sudo passwd user3

## Add Users to Groups
sudo gpasswd -a user1 shellgroup
sudo gpasswd -a user2 sftpgroup
sudo gpasswd -a user3 admingroup

## Verify Group Membership
getent group shellgroup
getent group sftpgroup
getent group admingroup

## 6. Create Shared Data Directory
sudo mkdir -p /data/sftpproject
sudo chown root:sftpgroup /data/sftpproject
sudo chmod 3770 /data/sftpproject

## Apply ACL Permissions
sudo setfacl -m g:sftpgroup:rwx /data/sftpproject
sudo setfacl -m g:shellgroup:rwx /data/sftpproject
sudo setfacl -d -m g:sftpgroup:rwx /data/sftpproject
sudo setfacl -d -m g:shellgroup:rwx /data/sftpproject

## Append-Only Attribute (Files Only)
sudo touch /data/sftpproject/file1.txt /data/sftpproject/file2.txt
sudo chattr +a /data/sftpproject/file1.txt

## Verify Permissions and Attributes
getfacl /data/sftpproject
lsattr /data/sftpproject

## 7. Create Chroot Directory Structure
sudo mkdir -p /sftp/user1 /sftp/user2 /sftp/share
sudo chown root:root /sftp /sftp/user1 /sftp/user2 /sftp/share
sudo chmod 755 /sftp /sftp/user1 /sftp/user2 /sftp/share

## 8. Permanent Bind Mount Shared Directory
Add the following line to /etc/fstab:

/data/sftpproject /sftp/share none bind 0 0

## Apply the config immediately without reboot
sudo mount -a

## Create Symlinks Inside Each User Chroot
sudo ln -s /sftp/share /sftp/user1/share
sudo ln -s /sftp/share /sftp/user2/share

## 9. SSH Server Installation
sudo apt install openssh-server
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

## 10. SSH ConfigurationEdit
Edit: nano/etc/ssh/sshd_config and add:

Subsystem sftp internal-sftp

Match Group shellgroup
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
    PermitTTY no

Match Group sftpgroup
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
    PermitTTY no

Match Group admingroup
    ForceCommand none

## ctrl+s & ctrl+x

## Restart SSH
sudo sshd -t
sudo systemctl restart ssh

## 11. Testing
sftp user1@localhost
sftp user2@localhost
ssh user3@localhost

## | User  | Group      | Access Result        |
| ----- | ---------- | ----------------------- |
| user1 | shellgroup | SFTP inside chroot      |
| user2 | sftpgroup  | SFTP-only inside chroot |
| user3 | admingroup | Full system access      |


## 12. Security Notes

Each user belongs to only one access group

Chroot directories are root-owned

No shell inside chroot

ACL + setgid + sticky bit enforce shared access

Append-only attribute prevents modification


## 13. Conclusion
This project demonstrates a secure and structured SSH/SFTP environment using best practices suitable for junior Linux sysadmins and DevOps roles, with automatic shared folder access for group members and easy onboarding of new users.



## 14. Adding New Users (Automatic Permission Inheritance)

This environment is designed for scalable and low-maintenance user management.
When a new user is added, no manual permission or directory changes are required.

Access is controlled entirely by group membership.

## 14.1 Create a New User
sudo useradd -m -s /usr/sbin/nologin newuser
sudo passwd newuser

Use /usr/sbin/nologin for SFTP-only users

Use /bin/bash if shell access is required

## 14.2 Assign the User to a Group
Add the user to one group only, based on the required access level:

## SFTP-only access inside chroot
sudo usermod -aG sftpgroup newuser

## SFTP access inside chroot (shellgroup scenario)
sudo usermod -aG shellgroup newuser

## Full system access (administrator)
sudo usermod -aG admingroup newuser


## 14.3 Automatic Behavior (No Manual Configuration)

After the user is added to a group:

✅ Access to the shared directory is granted automatically

✅ Permissions are inherited via ACL and default ACL

✅ New files inherit correct group ownership (setgid)

✅ Chroot isolation is applied automatically (if applicable)

✅ SSH configuration does not need to be modified

## 14.4 Verify User Access

## Test SFTP access
sftp newuser@localhost

## Verify group membership
id newuser

## Design Principle:

This setup follows a group-based access control model:

Groups define access rules

ACLs enforce permissions

Chroot enforces isolation

Administrators only manage group membership

Adding a user = adding the user to a group
All permissions and access rules are applied automatically.
