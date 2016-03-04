# Gerrit Workflow

#### Gerrit Workflow v Github Workflow

This page describes the process for using Gerrit to submit changes to Xenon, initiate automated testing and perform code reviews.
 
## Account Setup

Before you may begin, you'll need to have an account on the Gerrit server hosting Xenon builds:

* Visit https://enatai-gerrit.eng.vmware.com/ and click the Sign In link at the top-right corner of the page. Log in with your LDAP account (just username, no ‘@vmware.com’ required)
* Register your public SSH key by clicking on your name in the upper right-hand corner, go to Settings, SSH Public Keys or by following [this link](https://enatai-gerrit.eng.vmware.com/#/settings/ssh-keys).

Ensure that you have told 'git' who you are as this will be required for access to Gerrit when using the 'git-review' plugin.

```sh
git config --global user.name "Firstname Lastname"
git config --global user.email "your_email@youremail.com"
```

If you use multiple credentials for your projects, we presume you know how to switch (TODO: reference to using git with multiple logins)

Test your configuration now by issuing the following commands 

```sh
git config --list
```

## Installing git-review
We recommend using the git-review tool which is a git subcommand that handles all the details of working with Gerrit, making this process much much easier.  This component was developed by the folks at Openstack Infrastructure and used in the OpenStack code review system. It is widely available and included in the standard distros for Ubuntu, SUSE, Fedora and RHEL (see below).  A nice documentation page for git-review [is provided here](https://www.mediawiki.org/wiki/Gerrit/git-review).  More information and documentation is available w/in the [Openstack repository here](http://git.openstack.org/cgit/openstack-infra/git-review/tree/README.rst).
 
 Before you start work, make sure you have git-review installed on your system.  Steps to perform for several platforms are provided below with more complete instructions available in the documentations links above.
 
On Ubuntu Precise (12.04) and later, git-review is included in the distribution, so install it as any other package:

```sh
apt-get install git-review
```

On Fedora 16 and later, git-review is included into the distribution, so install it as any other package

On Red Hat Enterprise Linux, you must first enable the EPEL repository, then install it as any other package:

```sh
yum install git-review
```

On openSUSE 12.2 and later, git-review is included in the distribution under the name python-git-review, so install it as any other package:

```sh
zypper in python-git-review
```

On Mac OS X, or most other Unix-like systems, you may install it with pip, or if you prefer the brew package manager:

```sh
pip install git-review
```

For more information on the [brew](http://brew.sh/) package manager, please see [the documentation](https://github.com/Homebrew/homebrew).  Installation instructions are provided at the [brew home page](http://brew.sh/).

```sh
brew install git-review
```

## Starting Work on Xenon (New Project)

You can clone the repository in the usual way

```sh
git clone ssh://<username>@review.ec.eng.vmware.com:29418/xenon
```

If you;re having trouble, be sure that you have uploaded your public SSH key to Gerrit: https://review.ec.eng.vmware.com/#/settings/ssh-keys.  Additionally, you may find it helpful to use an SSH config file

```sh
Host review.ec.eng.vmware.com
    Hostname review.ec.eng.vmware.com
    User <your gerrit username>
    PreferredAuthentications publickey
    IdentityFile <path/to/ssh_key.pub
```

Next, you'll want git-review to configure your repository to know about Gerrit.  If you don't, it will do so the first time you submit a review but you will need the git-hook installed beforehand which git-review can do for you

```
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

#### Bugs

```
Pivotal: 
https://www.pivotaltracker.com/n/projects/1471320
```
