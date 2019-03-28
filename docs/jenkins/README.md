Jenkins Configuration for FreeBSD HW CI
=======================================
*~8 minute read*

Objective and Background
------------------------
This post explains how to configure Jenkins for the FreeBSD Hardware CI Lab. The work done here is based on earlier work using an Intel NUC and Rasberry PI <sup>[[1]](#LwhsuTestbedGit)[[2]](#RPITestbed)</sup>  along with work pioneered by the main Continuous Integration server, https://ci.freebsd.org/.<sup>[[3]](#JenkinsWiki)[[4]](#JenkinsSetupWiki)[[5]](#LwhsuCIGit)[[6]](#FreeBSDCIGit)</sup>

Pre-requisites
--------------
 * [Netbooting Setup](/docs/netboot)
 * [Power Controller Setup](/docs/devpower)

Tools and Requirements
----------------------
### Packages
 * `lang/python3`
 * `devel/jenkins`
 * `devel/py-jenkins-job-builder`
 * `devel/git`
 * `security/sudo`
 * `lang/groovy` (optional)
     * May be useful for creating/testing configurations used by Jenkins.
 * `lang/expect` (optional)
     * Can be useful for testing expect scripts (or autogenerating scripts using autoexpect) but requires some TCL knowledge.


Jenkins Initial Setup
---------------------
Setup `rc.conf` to enable Jenkins and then start the Jenkins server.
```bash
sudo sysrc local_unbound_enable=YES

sudo sysrc jenkins_enable=YES
sudo sysrc jenkins_home=/usr/local/jenkins
sudo sysrc jenkins_args='--webroot=${jenkins_home}/war --httpPort=8180'

sudo service jenkins start
```

Then navigate to `localhost:8180` or `<hostname>:8180` where `<hostname>` is the hostname of the machine running Jenkins. This will be referred to as `<jenkins_url>` from here on.

You should then be prompted to input the initial admin password. Choose "Select plugins to install" and then click "None" at the top to deselect all plugins and continue (you can install all necessary plugins later). Continue to follow the prompts to create the initial admin account (in my case the user is called `admin`) and setup Jenkins.

Install Plugins
---------------
Plugins can be installed at `<jenkins_url>/pluginManager/available`. And install the following plugins:
 * [Git](https://plugins.jenkins.io/git)
 * [Subversion](https://plugins.jenkins.io/subversion)
 * [Build Timeout](https://plugins.jenkins.io/build-timeout)
 * [Clang Scan-Build](https://plugins.jenkins.io/clang-scanbuild)
 * [Timestamper](https://plugins.jenkins.io/timestamper) 
 * [Parameterized Trigger](https://plugins.jenkins.io/parameterized-trigger)
 * [EnvInject](https://plugins.jenkins.io/envinject)
 * [Warnings Next Generation](https://plugins.jenkins.io/warnings-ng)
 * [Copy Artifact Plugin](https://plugins.jenkins.io/copyartifact)
 * [PostBuildScript](https://plugins.jenkins.io/postbuildscript) 
 * [Conditional Buildstep](https://plugins.jenkins.io/conditional-buildstep) 
 * [Matrix-Based Security](https://plugins.jenkins.io/matrix-auth) (optional - matrix based authentication for users)
 * [Embeddable Build Status](https://plugins.jenkins.io/embeddable-build-status) (optional - get status buttons)
 * [Email Extension](https://plugins.jenkins.io/email-ext) (optional - mailing after builds)
 * [Green Balls](https://plugins.jenkins.io/greenballs) (optional - green for success)
 * [Blue Ocean](https://plugins.jenkins.io/blueocean) (optional - prettier interface)


Jenkins Config Setup
--------------------
### Configure Workspaces
Create folders for Jenkins to work in.

```bash
sudo mkdir -p /jenkins/jails
sudo mkdir -p /jenkins/workspace
chown jenkins:jenkins /jenkins /jenkins/*
```

### Configure Permissions
Create a sudoers file for jenkins `/usr/local/etc/sudoers.d/jenkins`. This will allow jenkins to run the commands it needs to run in superuser without a password.
```
Cmnd_Alias CI_COMMANDS = /usr/sbin/jail, /usr/sbin/jexec, /sbin/mount, /sbin/umount, /sbin/devfs, /bin/chflags, /bin/rm, /usr/sbin/pkg, /usr/bin/tar, /sbin/ifconfig, /usr/bin/tee, /bin/mkdir, /sbin/mdconfig
Defaults:jenkins !env_reset
jenkins ALL=(root) NOPASSWD: CI_COMMANDS
```


### Define builds and build configurations
Clone the [the HW CI repository](https://github.com/GeraldNDA/freebsd-ci) into an easily accessible location. You may want to fork this repository before checking it out.

```bash
git clone https://github.com/GeraldNDA/freebsd-ci
```
> **NOTE**
> 
> This repository contains configurations and build scripts used on each build. By default, scripts are cloned from the repository any build that requires it. As a result, if you make any changes don't forget to push them (`git push`) after your commits.


#### JJB File Description
 * `default.yaml` defines default definitions for all jobs
 * `macro.yaml` defines various custom build and post-build tasks
 * `device-template.yaml` defines templates for building FreeBSD for devices and testing them
 * `device-test.yaml` defines the branches/target architectures/devices that should be built and tested. This is where the majority of your changes will go.
 * `jail.yaml` defines macros related to working in a jail
 * `mail-notify.yaml` defines macros related to getting email notifications 
 * `scm.yaml` defines macros related to SCMs. This is where support for new repositories and branches should be added.

Job configurations are in the `jjb` directory and managed by Jenkins Job Builder (JJB). 

In order to modify any configurations, you should first make a `jenkins_jobs.ini` file by copying the sample. You'll want to change the username to match the username used. As well you can create an API Token for yourself at `<jenkins_url>/user/<username>/configure` and use that as JJB's configured password.
> **WARNING**
> 
> While you could use your admin password instead of the API Token, it is not suggested to use a plaintext password with JJB for security reasons.


If you have forked the repository, you'll want to modify `default.yaml` and change `ci-scripts-git-path` to match the new repo. 
> **NOTE**
> 
> If the 'master' node is being used, you can also use a local path. This is done by modifying `device-template.yaml` and uncommenting the lines `checkout-scripts-local` and commenting out the `checkout-scripts`.


To update Jenkins with the configurations, run the following:
```bash
jenkins-jobs --conf jenkins_jobs.ini update --delete-old .
```

Jailer Node Setup
-----------------

Navigate to `<jenkins_url>/configureSecurity/` and under the "Agents" section, enable "TCP port for JNLP agents". If you don't have a firewall, "Random" is a good option.

This will enable you to launch nodes via Java Web Start

### Create a new node
Go to `<jenkins_url>/computer/new`

Ensure the following configurations are set:
 * `Name` can be anything you want
 * `Labels` should include `jailer` if this is to be where jails are created
 * `Usage` should be `use this node as much as possible`
 * `Launch method` should be `launch agent via Java Web Start`
 * `Environment Variables` should have the following values:
     * `BUILDER_0_IP4` should be some IP address in the same subnet as the IP address used to access the internet. 
         > **WARNING**
         > 
         > If it isn't defined correctly, `pkg`  will fail when trying to build FreeBSD. 

         > **NOTE**
         > 
         > Also define  `BUILDER_n_IP4` for every `n`th additional executor. Use `BUILDER_n_IP6` if IP6 IP addresses are used instead.

     * `BUILDER_JFLAG` is the number of cores build jobs should use. Use `syctl kern.smp.cpus` to choose this number.
     * `BUILDER_MEMORY` is the amount of memory build jobs should use (e.g. `8G`). This should be based on the amount of memory the node/computer has.
     * `BUILDER_NETIF` is the network interface the build jails should use. This should match the network interface to the internet (e.g. `em0`)
     * `BUILDER_ZFS_PARENT` is the parent directory of jails. This should be `/jenkins/jails`. The build scripts do not use ZFS.
         > **NOTE**
         > 
         > If you are using ZFS you may want to use the jail scripts for: https://github.com/freebsd/freebsd-ci/blob/master/scripts/jail instead of the included ones.



The following setup is tested on the master machine (that does not use ZFS). If you are using a distributed machine that does use ZFS you may benefit from looking at the steps used in https://wiki.freebsd.org/Jenkins/Setup. These steps should also be applicable on a distributed machine (with Jenkins installed).

First, enable jails:
```bash
sudo sysrc jail_enable=YES
sudo service jail start
```

The following tasks should be run as the `jenkins` user:
`sudo su jenkins`

Load the Jenkins scripts in the `/jenkins` directory
```bash
cd /jenkins
git clone https://github.com/GeraldNDA/jenkins-agent-scripts
cd jenkins-agent-scripts
```

Copy `agent.conf.sample` into `agent.conf`. You'll need to update `jenkins_url`, `agentname` and `secret` values to match the value what's given at `<jenkins_url>/computer/${agentname}/`. At this url, you should see "Run from agent command line" which gives a command like this: `java -jar agent.jar -jnlpUrl ${jenkins_url}/computer/${agentname}/agent-agent.jnlp -secret ${secret}`.

Configure the crontab to keep the instance running:
```bash
crontab crontab
```

To start the agent, run 
```bash
daemon -cf keepalive.sh
```


You should see at `<jenkins_url>/computer/${hostname}/` that the node is running successfully.

Modifying Devices
-----------------
Given the following `jobs` in `FreeBSD-device-tests` (in `device-test.yaml`)
```yaml
  - 'FreeBSD-{branch}-aarch64-device_tests':
          disable_build: true
          target: arm64
          target_arch: aarch64
          warnscanner: clang
          device_name:
            - pinea64

      # Each 32-bit arm device has it's own CONF file so they have their own build job.
      - 'FreeBSD-{branch}-beaglebone-device_tests':
          target: arm
          target_arch: armv7
          warnscanner: clang
          device_name: beaglebone
             conf_name: BEAGLEBONE
```

Let's say you wanted to add another 32-bit device like the `jetsontk1` and another aarch64 device like the `rockpine64` and two amd devices (`desktop1` and `performancepc1`) (Assuming they are already configured for net booting). After doing so your configuration file would like so:
```yaml
  - 'FreeBSD-{branch}-aarch64-device_tests':
          disable_build: true
          target: arm64
          target_arch: aarch64
          warnscanner: clang
          device_name:
            - pinea64
            - rockpine64
  - 'FreeBSD-{branch}-amd-device_tests':
          disable_build: true
          target: amd
          target_arch: amd
          warnscanner: clang
          device_name:
            - desktop1
            - performancepc1

  - 'FreeBSD-{branch}-beaglebone-device_tests':
          target: arm
          target_arch: armv7
          warnscanner: clang
          device_name: beaglebone
             conf_name: BEAGLEBONE
             
  - 'FreeBSD-{branch}-jetsontk1-device_tests':
          target: arm
          target_arch: armv7
          warnscanner: clang
          device_name: beaglebone
             conf_name: JETSON-TK1
```


 Additionally, add the following build groups for each `job` to add device/architecture specific builds as necessary. 
```yaml
- job-group:
    name: 'FreeBSD-{branch}-amd-device_tests'
    jobs:
      - 'FreeBSD-{branch}-{target_arch}-build_devtest'
      - 'FreeBSD-device-{branch}-{device_name}-test'
- job-group:
    name: 'FreeBSD-{branch}-jetsontk1-device_tests'
    jobs:
      - 'FreeBSD-{branch}-{device_name}-build_devtest'
      - 'FreeBSD-device-{branch}-{device_name}-test
```

Also, add these devices to the `conserver.cf` file so that they can be accessed by the power control and serial scripts. A template `conserver.cf` file is described [here](/WaraH18ZQK2LgxyFFQxJOA).

Additionally, you'll have to add jobs for each of these. At the time of writing the repository includes a sample device specific (beaglebone) and architecture specific (pinea64) build job and test job. Thus it should be sufficient to just copy them. And then follow the steps for modifying tests to ensure they will work. For example:
```bash
cp -r jobs/FreeBSD-device-head-beaglebone-test jobs/FreeBSD-device-head-jetsontk1-test
cp -r jobs/FreeBSD-device-head-beaglebone-build_devtest jobs/FreeBSD-device-head-jetsontk1-build_devtest

cp -r jobs/FreeBSD-device-head-pinea64-test jobs/FreeBSD-device-head-desktop1-test
cp -r jobs/FreeBSD-device-head-pinea64-test jobs/FreeBSD-device-head-performancepc1-test
cp -r jobs/FreeBSD-head-aarch64-build_devtest jobs/FreeBSD-device-head-amd-build_devtest
```

Modifying Tests
---------------
### Device Test Builds
For architecture-specific builds, the defaults should work as expected. However, for device-specific builds you will need to update some files.

Each build job contains:
- `job/<jobname>/build.sh` - the script that runs the build
- `scripts/build/build-world-kernel-head.sh` - the script which performs the build and creates the artifacts used for installing the build (and NFS booting the devices)
- `job/<jobname>/jail.conf` - File which contains configurations for jail environment
- `job/<jobname>/src.conf` - File which contains additional build configurations. For device-specific builds, it should contain `KERNCONF=<DEVICE>-NFS`.
- `GENERIC-NFS` used for architecture specific builds. Contains a Kernel configuration based on `GENERIC` but adds NFS support
- `<DEVICE>-NFS` used for device-specific builds. Contains a kernel configuration based on `<DEVICE>` (or whichever kernel config is correct for this device).

### Device Tests
For device tests, the default tests simply install the files and turn on the device. They expect FreeBSD to boot successfully and will then power off the device. 

Each test job depends on the following files:
 - `job/<jobname>/build.sh` - the script that runs the tests
 - `scripts/test/extract-artifacts.sh` - the script that extracts the built files and installs them in a directory
 - `scripts/test/run-device-tests.sh` - runs the device scripts and loads any overrides to these scripts
 - `scripts/test/device_tests/test_device.py` - a boilerplate  test script that powers on the device and verifies that FreeBSD boots successfully. Will also source any additional tests if specified.
 - `job/<jobname>/device_tests/tests.py` - the sourced python file for specifying additional tests

Example `job/<jobname>/device_tests/tests.py`:
```python
# Either (or both) of these lists should be defined. 
# The lists should be iterables of functions as specified below.
to_run = []
panic_actions = []

class TestError(Exception):
    pass
    
def check_status(ch, msg):
    res = ch.expect(["0", r"\d+"])
    if res == 1:
        raise TestFailure(f"Failed to do task '{msg}'")
    ch.console.expect(ch.CMDLINE_RE)

def example_test(ch):
    """
    `ch` is the console handler
     - before running this function, the device has already been booted and logged into.
    
    `ch.timeout` is the boot timeout 
    - default value is 5 minutes, otherwise every expect timeout is 1 minute.
    
    `ch.console` is the Pexpect instance for running commands 
     - see Pexpect.spawn documentation for more info
    
    `ch.device` is the device instance. 
    
    `ch.device.name` is the device name

    `ch.device.turn_on/turn_off` are methods for controlling the power to the device.

    `ch.CMDLINE_RE` is the regex that matches the command line PS1
    """
    
    try:
        ch.console.sendline("ls /")
        ch.console.expect(ch.CMDLINE_RE)
        check_status(ch, "Look for files at root")
        
        ch.console.sendline("dmesg")
        ch.console.expect(ch.CMDLINE_RE)
        check_status(ch, "Check boot messages")
        
    except TestFailure as e:
        return (False, str(e))
    # Can return (status, message) or nothing.
    # return (True, "All's good!")

def bt_on_panic(ch):
    # before running this function, the device panicked during boot
    ch.console.sendline("bt")

# You can add multiple functions to either list. The will be run in the order of the list. 
to_run.append(example_test)
panic_actions.append(bt_on_panic)
```


You can specify additional tests to run after logging in or after a panic occurs (to extract debug information).


Troubleshooting
---------------
### SVN Checkouts Fail
> **NOTE**
>
> [Patches to this issue](https://issues.tmatesoft.com/issue/SVNKIT-740) have been submitted and it should be resolved soon.

In the `<jenkins_home>` directory (i.e. `/usr/local/jenkins`) remove any JNA files. You can use the following to do so.
```bash
sudo find war -name "*jna*" -exec sudo rm {} +  
sudo find plugins -name "*jna*" -exec sudo rm {} +  
```
Also delete any lingering workspaces (these may contain lock files which will prevent further checkouts)
```bash
sudo rm -rf /jenkins/workspace/*
```
And then restart jenkins
```bash
sudo service jenkins restart
```
Checking out from SVN should work properly now.

### Pkg Fails in Build jobs
`BUILDER_0_IP4` must be on the same subnet as the ip address used to access the interface. You can use a tool like this: http://meridianoutpost.com/resources/etools/network/two-ips-on-same-network.php to verify that your IP is on the same subnet i.e. if `em0` uses ip address `inet 192.168.32.164 netmask 0xffffff00 broadcast 192.168.32.255` then any address from `192.168.32.1` to `192.168.32.255` would be on the same subnet. Avoid using values at the highest range, because those are usually used as the "broadcast address" and avoid values too low as they are usually used as the "router address".

References
----------

<a name="RPITestbed">[1]</a>: https://ci-dev.freebsd.org/job/rpi3-test/

<a name="LwhsuTestbedGit">[2]</a>: https://github.com/lwhsu/testbed 


<a name="JenkinsWiki">[3]</a>: https://wiki.freebsd.org/Jenkins

<a name="JenkinsSetupWiki">[4]</a>: https://wiki.freebsd.org/Jenkins/Setup

<a name="LwhsuCIGit">[5]</a>: https://github.com/lwhsu/freebsd-ci

<a name="FreeBSDCIGit">[6]</a>: https://github.com/freebsd/freebsd-ci
