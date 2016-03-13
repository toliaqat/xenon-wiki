# Contributing to Xenon

Hi, and thanks for your interest in helping out with Xenon.  This page will guide you through a few important concepts you need to keep in mind when contributing to Xenon as well as specifics regarding the setup of your development environment, how to test and submit your changes.  

## Getting Started

Prior to 'diving in', there are a few book-keeping things that need to get taken care of:  

> If you haven't already, now is the time to fill out our [Contributor License Agreement](http://vmware.github.io/photon/assets/files/vmware_cla.pdf) and return a copy to [osscontributions@vmware.com](mailto:osscontributions@vmware.com) as you will need to do so before we can merge your contribution. 

#### Gerrit Workflow v Github Workflow

Currently there are two processes for submitting changes to Xenon: an internal process and an external process.  We are working to fix this - the internal process will become the external process as soon as we can scale it.  Just so we're not hiding anything, the only difference is around the use of a Gerrit workflow vs submitting Pull requests through Github.  So if you are external to VMWare please follow the [Github Workflow](github-workflow) model for now.  Folks inside VMWare, please follow and help us test the [Gerrit Workflow](gerrit-workflow).

#### Reporting Issues

Our public Pivotal Project at https://www.pivotaltracker.com/n/projects/1471320 is open to everyone, so please help us out by reporting issues you encounter.

#### Editing Guidelines

The team uses Eclipse or IntelliJ. Formatting style settings for both these editors can be found in the
[contrib](https://github.com/vmware/xenon/tree/master/contrib) folder.

#### Building and Testing Changes

Our developer guide has up to date information on how to set up your development environment to build and test xenon as well as a number of topics related to setting things up in different configurations and environments.

#### A Note about Commit Messages

After you have made your changes, added the relevant test cases, and made sure they pass, it is time to commit!

Please observe the following guidelines for writing a comprehensive commit message: http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html
