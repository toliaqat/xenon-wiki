AuthN and AuthZ is now built into Xenon and helps control access to services. This feature is turned off by default at this time and can be turned on by passing in the flag —-isAuthorizationEnabled=true to the ServiceHost during startup (or invoking the method serviceHost.setAuthorizationEnabled(true) before starting the service).

Once authorization is turned on, any request to a stateful Xenon service will be subject to two additional checks:
 
 1. Is the request on behalf of a valid user principal 
 2. Is that user authorized to perform the desired action the service.

Any unauthorized access will result in a 403 (Forbidden) response from Xenon.

Note that we do not have any authorization checks on stateless services at this time (it is in the works). Stateless services typically interact with other stateful services and that access is subject to authorization checks.

Also, FactoryService instances are stateless services which can be invoked by any user. POSTs to factories result in a stateful service instance being created and that operation will complete successfully only if the user who invoked the factory service has the right privileges on the resulting stateful service.

This document is intended to be an example of how to interact with and build services when authz is enabled. Please review the [authn/authz design document](./authn-authz) for details about the model.

For this document,  we need to consider the following authz related services in Xenon before we get started:

- UserService - Represents a valid user in the system
- UserGroupService - Represents a group of users - backed by a query that resolves to a set of UserService instances
- ResourceGroupService - Represents a group of resources - backed by a query that resolves to a set of services
- RoleService - Associates a UserGroupService with a ResourceGroupService and lets us control what level of visibility a UserGroupService instance can have on a ResourceGroupService instance.

Before we jump into examples of seeing authz in action, we need to know about one other key concept - privileged services. These are services(typically stateless)that can execute operations in a ’system context’. When running in the system context we bypass the regular security checks and allow access to all services - think of running in the system context as running as root on unix and a privileged service as a process with the setuid bit set. 

We obviously need to control what services can run as privileged and this is enforced by marking a service privileged explicitly before it is started. For example if we need to start the ‘UserInitializationService’ as privileged, we would invoke the following sequence in ServiceHost:

      	 host.addPrivilegedService(UserInitializationService.class);
         host.startService(
                Operation.createPost(UriUtils.buildUri(host,
                        UserInitializationService.class)),
                new UserInitializationService()); 


addPrivilegedService() is a protected method and hence can be invoked only by the ServiceHost instance, giving application builders control on what can be deemed privileged.

With that background, let us now look at example of authz in action - when authz is turned on we need to make sure we have a process to setup the various user groups, resource groups and roles governing access. 

The UserInitializationService aims to be such a service. It does the following:

- Sets up a UserService instance and a UserGroupService with just that user
- Creates a AuthCredentialsService instance for the user - this is required for identity assertion using the BasicAuthenticationService. Moving forward we will have integration with external authentication providers
- Creates a ResourceGroupService that will contain all services whose documentAuthPrinicipalLink is set to the user. documentAuthPrinicipalLink is a field in every ServiceDocument that is automatically set to the user who is creating or updating the service. The assignment happens automatically and cannot be explicitly set by users. ResourceGroupService instances can obviously be constructed in other ways based on how the application wants to control access to resources
- Creates a RoleService that ties the UserGroupService to the ResourceGroupService

Let us look at snippets of code which create the various services

### Creation of a UserService instance: 

Users will be able POST to the UseerInitializationService without any authentication. The POST body is a PODO(UserInitializationData) that has a user email(the identity of the user in this case) and a password(for authentication). 

The code will look like this:

    private void handlePost(Operation op) {
        if (!op.hasBody()) {
            op.fail(new IllegalStateException("No body specified"));
            return;
        }
        UserInitializationData userData = op.getBody(UserInitializationData.class);
        UserState userState = new UserState();
        userState.email = userData.email;
        userState.documentSelfLink = userData.email;

        Operation userOp = Operation.
                createPost(UriUtils.buildUri(getHost(), UserFactoryService.SELF_LINK)).
                setBody(userState).
                setCompletion((o, e) -> {
                    // if the user already exits, return the appropriate error to the user
                        if (o.getStatusCode() == Operation.STATUS_CODE_CONFLICT) {
                            op.fail(Operation.STATUS_CODE_CONFLICT);
                            return;
                        }
                        if (e != null) {
                            op.fail(e);
                            return;
                        }
                        createAuthServicesForUser(op);
                    });
        setAuthorizationContext(userOp, getSystemAuthorizationContext());
        sendRequest(userOp);

    }

This is a standard Xenon interaction pattern where a stateless service is issuing a POST to a FactoryService. The key thing to watch here is the line at the bottom that sets the authorization context of the operation to the system authorization context :  setAuthorizationContext(userOp, getSystemAuthorizationContext());

This service can do this only because it has been marked as a privileged service. Invoking the POST to the UserFactoryService in the system authorization context lets us add new users to the system - essentially create new UserService instances without having to be authenticated. 


Once we have the UserService instance in place we call createAuthServicesForUser() in the completion handler and that kicks off a chain of other services that need to be created.

### Creation of a UserGroupService, ResourceGroupService, RoleService instances: 

   private void createAuthServicesForUser(Operation op) {
        UserInitializationData userData = op.getBody(UserInitializationData.class);

        // completion handler to be invoked when any of the operation below complete
        // when all operations are complete, we mark the parent op as having completed
        int expectedNumberOfTasks = 4;
        AtomicInteger notificationCount = new AtomicInteger(0);
        CompletionHandler creationCompletion = (o, ex) -> {
            if (ex != null) {
                op.fail(ex);
                return;
            }
            if (notificationCount.incrementAndGet() == expectedNumberOfTasks) {
                // mark the parent task as complete
                op.complete();
                return;
            }
        };

        // create an auth credentials service for user - used for identify assertion
        AuthCredentialsServiceState auth = new AuthCredentialsServiceState();
        auth.userEmail = userData.email;
        auth.privateKey = userData.password;
        Operation authOp = Operation.
                createPost(UriUtils.buildUri(getHost(), AuthCredentialsFactoryService.SELF_LINK)).
                setBody(auth).
                setCompletion(creationCompletion);
        setAuthorizationContext(authOp, getSystemAuthorizationContext());
        sendRequest(authOp);

        // create a user group for user - this is a group of 1 at this time
        UserGroupState userGroupState = new UserGroupState();
        userGroupState.documentSelfLink = userData.email;
        userGroupState.query = new Query();
        userGroupState.query.setTermPropertyName(UserState.FIELD_NAME_EMAIL);
        userGroupState.query.setTermMatchType(MatchType.TERM);
        userGroupState.query.setTermMatchValue(userData.email);

        Operation userGroupOp = Operation.
                createPost(UriUtils.buildUri(this.getHost(), UserGroupFactoryService.SELF_LINK)).
                setBody(userGroupState).
                setCompletion(creationCompletion);
        setAuthorizationContext(userGroupOp, getSystemAuthorizationContext());
        sendRequest(userGroupOp);

        // create a resource group for all docs that belong to this user
        String userResourceGroupSelfLink = userData.email;
        String userResourceGroupPropertyName = ServiceDocument.FIELD_NAME_AUTH_PRINCIPAL_LINK;
        String userResourceGroupPropertyValue = UriUtils.buildUriPath(
                UserFactoryService.SELF_LINK, userData.email);
        ResourceGroupState userResourceGroupState = new ResourceGroupState();
        userResourceGroupState.documentSelfLink = userResourceGroupSelfLink;
        userResourceGroupState.query = new Query();

        userResourceGroupState.query.setTermPropertyName(userResourceGroupPropertyName);
        userResourceGroupState.query.setTermMatchValue(userResourceGroupPropertyValue);
        userResourceGroupState.query.setTermMatchType(MatchType.TERM);
        Operation userResourceGroupOp = Operation.
                createPost(UriUtils.buildUri(getHost(), ResourceGroupFactoryService.SELF_LINK)).
                setBody(userResourceGroupState).
                setCompletion(creationCompletion);
        setAuthorizationContext(userResourceGroupOp, getSystemAuthorizationContext());
        sendRequest(userResourceGroupOp);


        // create role for the user
        String userGroupLink = UriUtils.buildUriPath(UserGroupFactoryService.SELF_LINK,
                userGroupState.documentSelfLink);
        String userResourceGroupLink = UriUtils.buildUriPath(ResourceGroupFactoryService.SELF_LINK,
                userResourceGroupSelfLink);
        RoleState roleState = new RoleState();
        roleState.userGroupLink = userGroupLink;
        roleState.resourceGroupLink = userResourceGroupLink;
        // give complete access - control can obviously be limited
        roleState.verbs = new HashSet<>();
        roleState.verbs.add(Action.GET);
        roleState.verbs.add(Action.POST);
        roleState.verbs.add(Action.DELETE);
        roleState.verbs.add(Action.PUT);
        roleState.verbs.add(Action.PATCH);
        roleState.policy = Policy.ALLOW;

        Operation roleOp = (Operation.
                createPost(UriUtils.buildUri(getHost(), RoleFactoryService.SELF_LINK)).
                setBody(roleState).
                setCompletion(creationCompletion));
        setAuthorizationContext(roleOp, getSystemAuthorizationContext());
        sendRequest(roleOp);

    }


Once the above sequence of operations are complete we have created a UserGroupService(with one user) a ResourceGroupService(backed by a query that resolves to all documents created by the user) and a role that gives the user complete access to those services. Note that that role dictates what verbs the user group can invoke.


The next step is to login as the user. The code for that will look like this:

       String userPassStr = new String(Base64.getEncoder().encode(
                new StringBuffer(userName).append(":").append(password)
                        .toString().getBytes()));
        String headerVal = new StringBuffer("Basic ").append(userPassStr).toString();
        sendRequest(Operation
                .createPost(UriUtils.buildUri(getHost(), BasicAuthenticationService.SELF_LINK))
                .setBody(new Object())
                .addRequestHeader(BasicAuthenticationService.AUTHORIZATION_HEADER_NAME, headerVal)
                .setCompletion(
                        (o, e) -> {
                            if (o.getStatusCode() != Operation.STATUS_CODE_OK) {
                                // error logging in
                                return;
                            }
                            // at this point the user is logged in and the response has a set-cookie header containing a JWT token
                        }));
 
You can alternately invoke a POST on the BasicAuthenticationService endpoint using a REST client and pass in a base64 encoded string containing the username/password as a authorization header.

The BasicAuthenticationService will check the credentials passed in against the credentials it stores as AuthCredentialsService instances. If the credentials match a JWT token encoding the user principal (and claims) is created and passed back as a set-cookie response header. Any subsequent request on behalf of the user will need to have this cookie passed in as a request header. Xenon will decode the cookie and create the appropriate authorization context and subject the request to the appropriate security checks.

