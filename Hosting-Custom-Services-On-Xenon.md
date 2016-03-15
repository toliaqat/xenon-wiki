# Hosting custom Xenon services

The Xenon framework comes with a set of core services but it is meant as general hosting framework, where authors create a set of services, package them in a JAR, then start them using either a custom host, invoked from the command line, or through dynamic loading.

Please first take a look at the [example service tutorial](./Example-Service-Tutorial) to get an idea on how to build and interact with Xenon services.

# Creating a custom host JAR

To start a Xenon process, with custom services, you need to follow these steps

 1. create a new JAR that represents your host.
 1. add the Xenon jar as a dependency
 1. add a custom host class, that derives from ServiceHost
 1. Override the start() method and start the core services
 1. Start any additional services you created.

To start your host invoke it using the java 8 jvm:

java --jar <custom-host-name>.jar --port <portnumber>

More details on starting one or more Xenon hosts, see the [debugging page](./Debugging-and-Troubleshooting#starting-a-host)

## Derived Host example

Here is a snippet from the source, showing the few lines needed in creating a custom host:

```
class ExampleServiceHost extends ServiceHost {

    public static void main(String[] args) throws Throwable {
        ExampleServiceHost h = new ExampleServiceHost();
        h.initialize(args);
        h.start();
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            h.log(Level.WARNING, "Host stopping ...");
            h.stop();
            h.log(Level.WARNING, "Host is stopped");
        }));
    }

    @Override
    public ServiceHost start() throws Throwable {
        super.start();

        startDefaultCoreServicesSynchronously();

        // start the example factory
        super.startService(new ExampleFactoryService());

        return this;
    }
}

```

The following are needed:
 1. a static main() that creates an instance of the custom host class (which derives from ServiceHost)
 1. call to the super class initialize(args) method, with the command line arguments
 1. call to the super class start() method
 1. Override of the start() method
 1. starting the core services
 1. starting any additional, custom services (in this case, we start the provisioning services)

# Packaging services

The custom host jar can include service classes or services can be packaged in their own jar so they can be re-used by other projects.

# Automatic service enumeration and loading
See the xenon-loader package, it provides a service for finding and loading xenon services from jar files, dynamically.
