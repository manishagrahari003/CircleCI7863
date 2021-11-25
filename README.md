## Prerequisite:
The assumptions are detailed in the sparkler-setup.sh. Running that will show requirements before continuing.

## Usage:
1. Download the script: [sparkler-setup.sh](https://git.corp.adobe.com/WebPA/sparkler-chef-recipes/blob/main/sparkler-setup.sh)<br>
**Note:** if you want to run provisioning recipes from a branch other than main, make sure to get the sparkler-setup.sh from that branch.
2. chmod +x the downloaded script
3. Run the script (note the options below):<br>
<code>./sparkler-setup.sh -b <branch_to_run> -e [xd-stage|xd-pipeline] -r [mac_dev|mac_test|mac_build|mac_sonarqube] -d [debug|info] -k <path_to_ssh_key_file></code>

* -b: Name of branch. Optional, defaulting to `main` if not provided. If you want to run some recipes from a development/test branch, specifying this option will let you run the branch before it is merged to main.<br>
* -e: Environment for chef-client to choose. Required only if provisioning an agent. One of `xd-stage` or `xd-pipeline`<br>
* -r: Role for chef-client to choose. Required. One of `mac_dev`, `mac_test`, `mac_build` or `mac_sonarqube`.<br>
* -d: Optional. If you don't specify anything, the default is `info`. You might want to use `-d debug`, because this option will not delete the temporary working directory containing all the recipes, allowing the chance to debug the provisioning process.<br>
* -k: Optional, defaulting to `$HOME/.ssh/id_rsa`. You can provide absolute/relative path if you want to use other custom ssh key file.
* -l: Optional, defaulting to false. Ignored if -b is specified. This will use the current directory instead of fetching from git. Use this for development.
* --useVault: Required for agent provisioning. Used to fetch credentials from Vault when provisioning a Jenkins agent.
```diff
- Use this option only for agents and not for development machines.
```
* --vaultRoleId=[ROLE_ID]: Specify the Vault Role ID. This option should be used alongside with `--useVault`
* --vaultSecretId=[SECRET_ID]: Specify Vault Secret ID. This option should be used alongside with `--useVault`

## Setting up a Jenkins agent
Wiki : [HOW-TO Set up XD Jenkins Agent](https://wiki.corp.adobe.com/display/UXDesign/%5BMAC%5D+HOW-TO+Set+up+XD+Jenkins+Agent) <br>
<code>./sparkler-setup.sh -r mac_agent -e xd-pipeline --useVault</code>

OR
<code>./sparkler-setup.sh -r mac_agent -e xd-pipeline --useVault --vaultRoleId=[ROLE_ID] --vaultSecretId=[SECRET_ID]</code>
* -b was not specified, so this means, you are simply running the provisioning recipes from the main
* -r mac_agent: provisioning a build machine as a Jenkins agent.
* -e xd-pipeline: And thus, this build machine is provisioned as an agent on the xd-pipeline Jenkins server
* -d debug: Want to keep the working files after finishing/failing the provisioning process, still there are things to be ironed out. Also, this parameter increases the log level.
* --useVault : Fetch credentials from Vault

## Setting up a development machine
<code>./sparkler-setup.sh -r mac_dev</code>

If you are a developer trying to update/prepare your dev machine for XD development, this is what you want to run.
* -b was not specified, so this means, you are simply running the provisioning recipes from the main
* -r mac_dev: provisioning a development machine. This machine is not meant to become Jenkins agent.


## Implementation:
#### High Level
Use chef in local mode: <br>
This helps in executing the cookbooks and recipes available on the local mode without interfacing to server recipes are implemented with following 2 major steps:

#### Flow of Setup

sparkler-setup.sh does following before the control

- If `--useVault` parameter was not specified the script checks what type of machine you are going to provision `agent` or `developer`. If `agent` option is entered `--useVault` option is enabled, otherwise you will be prompted to enter the local password.
- Make sure the script is not running as root (sudo). If you are running this script as root/sudo, the script will inform you and exit.
- Check the OS version to make sure the OS is supported. (Currently the minimum version is 10.13.6)
- If`--useVault` parameter is specified the script will login into Vault using the [Vault_AppRole_Credentials](https://vault-amer.adobe.net/ui/vault/secrets/dme_adobe_xd/show/Sparkler/Automation/Vault_AppRole_Credentials)(which can be passed as parameters or at the script runtime ) and fetch the LDAP credentials, Artifactory credentials, Jenkins API credentials, SSH keys and local credentials.
	- **Note** this option should be used only when provisioning Jenkins agents.
- If a development machine is provisioned this script will try to fetch `ldap user`, `ldap password`, `artifactory user`, `artifactory api key`, and `local computer's login password` from environment variable(LDAP_USER, LDAP_PASSWORD, ARTIFACTORY_USER, ARTIFACTORY_API_KEY, BUILD_ACCOUNT_PASS). If not found in the environment, it prompts you to enter them.
- Performs check on correctness of ldap credentials so that you find out early that you entered the wrong password - How nice, isn't it?:-)
- Performs check that your ssh key file exists
- Installs Mac OS X command line tools if it didn't exists already.
- Installs homeBrew if it was not already installed.(homeBrew installer prompts for sudo password). Then, updates the packagelist.
- Install chefdk - Chef SDK. This chef sdk is installed from brew cask : https://github.com/caskroom/homebrew-cask
- Install a brew cask for local brew versions: https://git.corp.adobe.com/WebPA/xd-homebrew-versions
- Install a cask for other versions: ssh://git.corp.adobe.com/WebPA/xd-homebrew-versions
- Create a temporary directory for clone and usage of chef recipes (and change directory to it).
   You can find out what the directory is by looking at /tmp/flags file.
- clones chef recipe repo there, and check out to the branch if non-main branch was specified.
- Now, chef-client begins to run the recipes.
	- A new user will be created while provisioning a new agent from scratch (not for development machines)

#### Debugging Recipe on Mac: How To run an individual recipe

```
export LDAP_PASSWORD=your_ldap_password
export LDAP_USER=your_ldap
export ARTIFACTORY_USER=your_artfy_user
export ARTIFACTORY_API_KEY=your_artfy_apikey
export BUILD_ACCOUNT_PASS=local_computer_password
export LAUNCH_DIR=~/Desktop/sparkler-chef-recipe
```
- At LAUNCH_DIR,
   `berks vendor -b $LAUNCH_DIR/cookbooks/Sparkler/Berksfile`
- go to `$LAUNCH_DIR/cookbooks` directory, then
```
sudo chef-client --local-mode --config ../solo.rb -E xd-pipeline --override-runlist Sparkler::jenkins-node -l debug
```

* -E may not be needed if you are running other recipe (`-e xd-pipeline` of `sparkler-setup.sh` is mapped to this)
* If you have Chef::log.debug() in your recipe, `-l debug` will print it.
* If you have Chef::log.info(), then you won't need `-l debug`


#### Note for VM users

If you are using VM to run these recipes, here are few things you might run into.

1. Make sure you change your network settings of your VMs to `Thunderbolt Ethernet` or `Autodetect`. From the experience, `Share with my Mac` seems to make your VM's network connection flaky, causing a download failure, failing the remaining recipe.

2. Check the size of your VM before proceeding (the default 30GB is not enough), and your vm is running OS X El Capitan, you will find it frustrating not being able to increase the size of partition within Mac OS X even if the new size is showing in Disk Utility.

For OSX 10.11 and earlier:
The solution was found in this link:
https://themacwrangler.wordpress.com/2015/10/21/resizing-el-capitan-mac-volumes-under-vmware-fusion/

In a nutshell, run following command in terminal:

`sudo diskutil resizeVolume / R`

For OSX 10.12 and later:
https://derflounder.wordpress.com/2017/10/18/resizing-a-macos-vms-apfs-boot-drive-to-use-all-available-disk-space/
