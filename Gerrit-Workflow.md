# Gerrit Workflow

#### Gerrit Workflow v Github Workflow

This page describes the process for using Gerrit to submit changes to Xenon, initiate automated testing and perform code reviews.  Please see [Github Workflow](github-workflow) for details regarding working with Xenon via Github.
 
## Pre-Conditions

Before you may begin, you'll need to have an account on the Gerrit server hosting Xenon builds:

#### Gerrit Account
* Visit https://review.ec.eng.vmware.com and click the Sign In link at the top-right corner of the page. Log in with your LDAP account (just username, no ‘@vmware.com’ required).
* Register your public SSH key by clicking on your name in the upper right-hand corner, go to Settings, SSH Public Keys or by following [this link](https://review.ec.eng.vmware.com/#/settings/ssh-keys).
* See [this page](generating-and-configuring-ssh-keys) for more information on how to generate and configure SSH keys for use with Gerrit on your machine.

#### Local Git Configuration
Ensure that you have told 'git' who you are as this will be required for access to Gerrit when using the 'git-review' plugin.

```sh
git config --global user.name "Firstname Lastname"
git config --global user.email "your_email@youremail.com"
```

If you use multiple credentials for your projects, please see [this guide](ssh-configuration-file) for information on how to setup an SSH configuration file.

#### Testing your Configuration
The following command will list your local Git configuration settings.  Local settings in a project may over-ride global settings, so be sure to test this from within the project directory if you are trouble-shooting an exiting project.  

```bash
git config --list
```

Once you have validated that your user/email are correct, you can verify that your SSH key has been properly configured with Gerrit by issuing the commands below.

```bash
ssh -v -T -p 29418 username@review.ec.eng.vmware.com
```

#### Installing git-review
We recommend using the git-review tool which is a git subcommand that handles all the details of working with Gerrit, making this process much much easier.  This component was developed by the folks at Openstack Infrastructure and used in the OpenStack code review system. It is widely available and included in the standard distros for Ubuntu, SUSE, Fedora and RHEL (see below).  A nice documentation page for git-review [is provided here](https://www.mediawiki.org/wiki/Gerrit/git-review).  More information and documentation is available w/in the [Openstack repository here](http://git.openstack.org/cgit/openstack-infra/git-review/tree/README.rst).
 
 Before you start work, make sure you have git-review installed on your system.  Steps to perform for several platforms are provided below with more complete instructions available in the documentations links above.
 
**Ubuntu Precise (12.04) and later:** git-review is included in the distribution, so install it as any other package:

```bash
apt-get install git-review
```

**Fedora 16 and later:** git-review is included into the distribution, so install it as any other package

**Red Hat Enterprise Linux:** you must first enable the EPEL repository, then install it as any other package:

```bash
yum install git-review
```

**OpenSUSE 12.2 and later:** git-review is included in the distribution under the name python-git-review, so install it as any other package:

```bash
zypper in python-git-review
```

**Mac OS X:** or most other Unix-like systems, you may install it with pip, or if you prefer the brew package manager:

```bash
pip install git-review
```

For more information on installing the [brew](http://brew.sh/) package manager, please see [the documentation](https://github.com/Homebrew/homebrew).  Installation instructions are provided at the [brew home page](http://brew.sh/).

```bash
brew install git-review
```

## Starting Work on Xenon (New Project)

You can clone the repository in the usual way

```bash
git clone ssh://<username>@review.ec.eng.vmware.com:29418/xenon
```

If you're having trouble, be sure that you have uploaded your public SSH key to Gerrit: https://review.ec.eng.vmware.com/#/settings/ssh-keys and that your username / password are correct.  Additionally, you may find it helpful to use an SSH config file if you use multiple SSH keys for different projects:

```bash
Host review.ec.eng.vmware.com
    Hostname review.ec.eng.vmware.com
    User <your gerrit username>
    PreferredAuthentications publickey
    IdentityFile <path/to/ssh_key.pub
```

Next, you'll want git-review to configure your repository to know about Gerrit.  If you don't, it will do so the first time you submit a review but you will need the git-hook installed beforehand which git-review can do for you

```bash
cd <xenon-project-directory>
git config --global gitreview.username yourgerritusername
git review -s
```

Git-review normally sets up the necessary git remotes and configurations for you.  In the event that ```git review -s``` fails, it may be because ```git-review``` was unable to determine the location of your project.  If that's the case, you can create the gerrit remote manually and re-run ```git-review```

```
git remote add gerrit https://<username>@review.openstack.org/<umbrella repository name>/<repository name>.git
git review -s
```

Congratulations, you are now ready to submit changes for review on Xenon! 

Move ahead to the Project Workflow discussion.
