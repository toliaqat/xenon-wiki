# Gerrit Workflow

#### Gerrit Workflow v Github Workflow

This page describes the process for using Gerrit to submit changes to Xenon, initiate automated testing and perform code reviews.  Please see [Github Workflow](github-workflow) for details regarding working with Xenon via Github.
 
## Pre-Conditions

Before you may begin, you'll need to have an account on the Gerrit server hosting Xenon builds:

#### Gerrit Account
* Visit https://review.ec.eng.vmware.com and click the Sign In link at the top-right corner of the page. Log in with your LDAP account (just username, no ‘@vmware.com’ required).
* Register your public SSH key by clicking on your name in the upper right-hand corner, go to Settings, SSH Public Keys or by following [this link](https://review.ec.eng.vmware.com/#/settings/ssh-keys).
* See [this page](generating-and-configuring-ssh-keys) for more information on how to generate and configure SSH keys for use with Gerrit on your machine.

#### Local Configuration
**SSH Key Configuration.** If you have generated a new SSH key, make sure the ssh-agent is running and add your key to the ssh agent as follows:

```
# make sure the agent is started:
$ eval "$(ssh-agent -s)"
Agent pid 59566

# add your key:
$ ssh-add ~/.ssh/id_rsa
```

To list the installed keys, use the following commands:
```
$ ssh-add -l -E md5
```

**Git Configuration.** Ensure that you have told 'git' who you are as this will be required for access to Gerrit when using the 'git-review' plugin.

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

#### Installing and Configuring git-review
We recommend using the git-review tool which is a git subcommand that handles all the details of working with Gerrit, making this process much much easier.  This component was developed by the folks at Openstack Infrastructure and used in the OpenStack code review system. It is widely available and included in the standard distros for Ubuntu, SUSE, Fedora and RHEL (see below).  A nice documentation page for git-review [is provided here](https://www.mediawiki.org/wiki/Gerrit/git-review).  More information and documentation is available w/in the [Openstack repository here](http://docs.openstack.org/infra/git-review/installation.html#installing-git-review).
 
Steps to perform for a few common platforms are provided below with more complete instructions available in the documentations links above.
 
**Ubuntu Precise (12.04) and later:** git-review is included in the distribution, so install it as any other package:

```bash
apt-get install git-review
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

Now, you'll need to configure your repository to know about Gerrit.  To do so, manually set up an origin named 'gerrit' in your local git configuration as shown in the following example.  Then tell git-review to configure itself with the ```git review -s``` command.

```
cd <xenon-project-directory>
git remote add gerrit ssh://<username>@review.ec.eng.vmware.com:29418/xenon
git config gitreview.username yourgerritusername
git review -s
```

Documentation for git-review can be found [here](http://docs.openstack.org/infra/git-review/usage.html) but here are some of the more common commands:

Hack on some code and then issue the normal 'git commit' command (well, presuming you collected your changes with ```git add``` first, but we're not going to detail the complete ```git``` workflow here, just the gerrit interaction.

```
git commit -m "your comment detailing your changes according with the guidelines"
```

Then, you're ready to submit your changes for review:

```
git review
```

git-review is pretty instructive and will (usually) coach you along the process. Once your changes are reviewed and the unit tests are successful, the change will be ready to 'submit' for merging.  This is done through the Gerrit UI by clicking the 'submit' button next in your change description screen.  For a complete rundown of the Gerrit UI and more Gerrit documentation, follow the [Gerrit User Guide](https://review.ec.eng.vmware.com/Documentation/intro-user.html) and [Gerrit UI](https://review.ec.eng.vmware.com/Documentation/user-review-ui.html#download).

A great diagram of the gerrit workflow of a change was included in the [Android Project](http://source.android.com/source/life-of-a-patch.html)

Congratulations, you just submitted your first patch!

Move ahead to the Developer Setup Guide and Project Tutorials.

## Troubleshooting ##

Git-review normally sets up the necessary git remotes and configurations for you.  In the event that ```git review -s``` fails, it may be because ```git-review``` was unable to determine the location of your project.  If that's the case, you can create the gerrit remote manually and re-run ```git-review```

Additionaly information on git-review commands can be found in the [Git Review Usage Guide](http://docs.openstack.org/infra/git-review/usage.html) or by typing ```git review -h```.

See the Git SCM Book for more information on [Working With Remotes](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes).

The [Advanced Gerrit Usage Guide](https://www.mediawiki.org/wiki/Gerrit/Advanced_usage) at Media Wiki also details a number of useful scenarios you may encounter with Git, Gerrit, and Git-Review.

