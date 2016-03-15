# Overview
The xenon-client module provides a simple command line interface to Xenon
services.

# Command Usage

```
xenonc [OPTIONS] METHOD SERVICE [FLAGS]...

Options:
  -i string
    	Body input
  -n	Do not print a trailing newline character
  -s string
    	Output field selector
  -t string
    	Enables trace dump of all HTTP data, to the given output file
  -x	Execute body template and print request to stdout
  -xenon string
    	Root URI of xenon node

```

For detailed information on examples, how to build and use `xenonc` see [Readme](https://github.com/vmware/xenon/blob/master/xenon-client/src/main/go/README.md)  
