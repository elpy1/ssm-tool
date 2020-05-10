# ssm-tool
Toolkit to complement ssh-over-ssm.


## Info and requirements
It's been a while since I uploaded and started using ssh-over-ssm and it has really had an impact in making life easier for sysadmins/engineers needing access to AWS EC2 instances.  Over time, I've been making small changes to `ssm-tool` (which began as a script to simply list SSM instances) to simplify both SSH access and configuration.


### Requirements
- `python3` with some pip modules (see `requirements.txt`)
- `awscli`
- `ssh-ssm.sh` from ssh-over-ssm
- a config snippet added to the end of your SSH config (`~/.ssh/config`)
- `ssh-ssm.sh` and `ssm-tool` in your `$PATH`
- ssm-agent installed and updated on remote instances
- necessary AWS IAM permissions for your user and the remote instance (SSM/EC2)


### Installation and usage
1) Clone this repository and `ssh-over-ssm`
2) Place scripts in a familiar local directory (e.g. `~/bin/`) and add to PATH -> `echo "export PATH="$HOME/bin${PATH:+:${PATH}}""| tee -a ~/.bashrc`
3) Install required python modules  -> `pip3 install --user -r {ssm-tool repo}/requirements.txt`
4) Add SSH config (see below)
5) Verify tools work

### SSH config
To make use of the functionality provided by `ssh-over-ssm` and `ssm-tool` you will need to place the following snippet (example config provided in repo) at the end of your SSH config file (`~/.ssh/config`):
```
Match exec "grep -qs '^Host.*%n' %d/.ssh/ssmtool-*"
  Include ssmtool-*

Host i-*
  ProxyCommand ssh-ssm.sh %h %r
  IdentityFile ~/.ssh/ssm-ssh-tmp
  StrictHostKeyChecking no
  PasswordAuthentication no
  ChallengeResponseAuthentication no
  ```

## How it works
### ssh-over-ssm
`ssh-over-ssm` is simply a small bash script wrapper which performs some checks on execution and then runs two AWS commands:
- `ssm send-command` (with ssm document `AWS-RunShellScript`)
- `ssm start-session` (with document `AWS-StartSSHSession`).

The difference between this and the `ProxyCommand` provided by AWS in their [documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html) is that `ssh-over-ssm` automates placing your local SSH key (and generating one if need) on the remote server prior to starting the SSH session and connecting. The public key is removed after 15 seconds. Without this your SSH public key must exist on the instance, under the correct user's directory before you attempt to connect.

### ssm-tool
`ssm-tool` is a python script which acts as a toolkit for SSM. It was created and designed to work in conjunction with `ssh-over-ssm`. Its' main functionality is:
- list/search SSM instances (filter by windows/linux, output as formatted table or plain text)
- update SSM agent on instances if need
- access instances (via SSM session or SSH)
- generate SSH config fragments to simplify SSH access over SSM

### why both?
I'd prefer to keep `ssh-over-ssm` as a small bash wrapper script which can continue to be used without `ssm-tool` if preferred.
If you wish to make use of `ssm-tool` to generate SSH config for you or access instances directly using SSH, `ssh-over-ssm` knows to check for `ssm-tool` configuration when it is executed. 

**NOTE:** If your SSH public key does not exist on the remote server you **must** use `ssh-over-ssm` for seamless SSH access.

## Example usage
For the below examples:
- I am assuming role to accounts using AWS CLI profiles configured in `~/.aws/{config,credentials}`.
- I have configured a region for my `default` and other configured awscli profiles.
- I am using the example SSH configuration file from this repo.
- I do not have a SSH key on my machine locally (and obviously there's no public key on the remote server).


**Listing instances and searching/filtering results by platform OS, IP and tag:name:**
```
[elpy@testbox ~]$ ssm-tool --profile home-dev
+--------------------------+---------------------+---------------+------------+--------------+
| tag[name]                | instance            | ip address    | ssm-agent* | platform     |
+--------------------------+---------------------+---------------+------------+--------------+
| home-dev-jumpbox-01      | i-0xxxxxxxxxxxx79d6 | 10.xxx.24.9   | True       | Amazon Linux |
| home-dev-confluenceasg   | i-0xxxxxxxxxxxx9007 | 10.xxx.24.1xx | True       | CentOS Linux |
| home-dev-bambooasg       | i-0xxxxxxxxxxxx29b9 | 10.xxx.24.2xx | True       | CentOS Linux |
| home-dev-jiraasg         | i-0xxxxxxxxxxxxc331 | 10.xxx.24.2xx | True       | CentOS Linux |
+--------------------------+---------------------+---------------+------------+--------------+
 * ssm-agent column refers to whether the agent is up-to-date

[elpy@testbox ~]$ ssm-tool --profile home-dev amazon
+-----------------------+---------------------+-------------+------------+--------------+
| tag[name]             | instance            | ip address  | ssm-agent* | platform     |
+-----------------------+---------------------+-------------+------------+--------------+
| home-dev-jumpbox-01   | i-0xxxxxxxxxxxx79d6 | 10.xxx.24.9 | True       | Amazon Linux |
+-----------------------+---------------------+-------------+------------+--------------+

[elpy@testbox ~]$ ssm-tool --profile home-dev 24.2
+----------------------+---------------------+---------------+------------+--------------+
| tag[name]            | instance            | ip address    | ssm-agent* | platform     |
+----------------------+---------------------+---------------+------------+--------------+
| home-dev-bambooasg   | i-0xxxxxxxxxxxx29b9 | 10.xxx.24.2xx | True       | CentOS Linux |
| home-dev-jiraasg     | i-0xxxxxxxxxxxxc331 | 10.xxx.24.2xx | True       | CentOS Linux |
+----------------------+---------------------+---------------+------------+--------------+

[elpy@testbox ~]$ ssm-tool --profile home-dev jira
+-------------------+---------------------+---------------+------------+--------------+
| tag[name]         | instance            | ip address    | ssm-agent* | platform     |
+-------------------+---------------------+---------------+------------+--------------+
| home-dev-jiraasg  | i-0xxxxxxxxxxxxc331 | 10.xxx.24.2xx | True       | CentOS Linux |
+-------------------+---------------------+---------------+------------+--------------+
```

**Connecting to an instance using a SSM session:**
```
[elpy@testbox ~]$ ssm-tool --profile home-dev --session i-0xxxxxxxxxxxx29b9

Starting session with SessionId: example123-0e467c6bf9f9ae39d
sh-4.2$ sudo -i
[root@ip-10-xxx-24-2xx ~]# logout
sh-4.2$ exit

Exiting session with sessionId: example123-0e467c6bf9f9ae39d.
```

**Connecting to an instance over SSH using `ssm-tool` and instance id:**
```
[elpy@testbox ~]$ ssm-tool --profile home-dev --ssh centos@i-0xxxxxxxxxxxx29b9
Last login: Fri May  8 10:54:38 2020 from localhost
[centos@ip-10-xxx-24-2xx ~]$ sudo -i
[root@ip-10-xxx-24-2xx ~]#
[root@ip-10-xxx-24-2xx ~]# logout
[centos@ip-10-xxx-24-2xx ~]$ logout
Connection to i-0xxxxxxxxxxxx29b9 closed.
```

**Using ssm-tool to generate SSH config and then using ssh directly to connect:**
```
[elpy@testbox ~]$ ssm-tool --profile home-dev --ssh-conf

ssh config fragment generated and saved to -> /home/elpy/.ssh/ssmtool-home-dev
[elpy@testbox ~]$ ssh home-dev-jumpbox-01
Last login: Sun May 10 07:15:35 2020 from localhost

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
74 package(s) needed for security, out of 154 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-10-xxx-24-9 ~]$ logout
Connection to i-0xxxxxxxxxxxx79d6 closed.
```
**Connecting to a different instance (same AWS profile) via its' IP:**
```
[elpy@testbox ~]$ ssh 10.xxx.24.1xx
Last login: Sun May 10 07:18:48 2020 from localhost
[centos@ip-10-xxx-24-1xx ~]$ logout
Connection to i-0xxxxxxxxxxxx9007 closed.
```

