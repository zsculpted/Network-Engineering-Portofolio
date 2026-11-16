# FortiGate Administrator Configuration and Security

## Project Introduction

To start the FortiGate projects, I feel like it's only right for me to begin with exploring FortiGate administrator configuration and security. Before moving into more advanced firewall configurations.

I will focus on configuring administrator accounts, managing usernames and passwords, setting up authentication security, implementing Two-Factor Authentication (2FA), and configuring email settings for authentication and notifications.

Throughout this project, I will document each configuration and verify the results through practical testing using both the FortiGate GUI and CLI or either one.

## Project Objectives

1. Configure administrator accounts, including creating usernames and passwords.
2. Configure administrator profiles and manage access privileges.
3. Implement password security policies and login restrictions.
4. Configure Two-Factor Authentication (2FA) for administrator accounts.
5. Configure email settings for authentication and system notifications.
6. Configure trusted hosts to restrict administrative access.
7. Secure administrative management access through HTTPS and SSH.

## Network Topology

<img width="1127" height="646" alt="Screenshot 2026-11-14 161159" src="https://github.com/user-attachments/assets/10b43b23-4fd9-4df5-8823-febeeab43092" />

Today, I'll focus on the initial administrator configuration of **Fortinet2** (yellow highlight). As shown in the output below, the device is in its initial state, with only the default `admin` account present. 

<img width="936" height="327" alt="image" src="https://github.com/user-attachments/assets/9d35a51a-acb5-48d3-87a5-4153b40d6521" />

I'll use this as my starting point before proceeding with the administrator account setup and additional security configurations.

## Configuration and Implementation

### 1. Administrator Account Configuration

For this objective, I'll create two additional administrator accounts on the FortiGate device: `raja` and `ratu`. Each account will have a different level of administrative privileges. Raja will be assigned the `prof_admin` profile, while Ratu will receive the `super_admin` profile.

The purpose of this configuration is to understand how FortiGate manages multiple administrator accounts and how administrator profiles determine the level of access granted to each user.

**Configuration:**

```bash
config system admin
    edit "raja"
        set accprofile "prof_admin"
        set password "raja"
    next
    edit "ratu"
        set accprofile "super_admin"
        set password "ratu"
    next
end
```

**Testing:**

Successfully able to access the FortiGate GUI using both account.

  <img width="602" height="392" alt="image" src="https://github.com/user-attachments/assets/4e1dd9ab-ea10-457a-917d-e2a1ce1b7b7b" />
  
<img width="634" height="477" alt="image" src="https://github.com/user-attachments/assets/3123f896-57ba-42a6-9866-dd0d7fbc40f4" />


**Troubleshooting:**

No issues encountered during the configuration.

### 2. Administrator Profiles and Access Privileges

Will be continued...
