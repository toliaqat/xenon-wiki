# Overview
This page will talk in detail about external authentication support in xenon and how a xenon based application can be coded to authenticate against an external authentication source.

# Prerequisites
It's recommended to go through these topics below if you have not already done, these will give you an understanding of existing basic authentication and authorization in xenon

* [Authentication and Authorization Tutorial](./Authentication-And-Authorization-Tutorial)

* [Authentication and Authorization Design](./Authentication-and-Authorization-Design)

# External authentication in xenon

Xenon provides an adapter based solution for authenticating with any external authentication provider. The external authentication provider could be a LDAP based system, an authorization server or even a plain credential lookup service that could be abstracted as a xenon authentication adapter adhering to the contracts established by the framework.

Once your application is configured with a xenon authentication adapter all the requests will be routed through the adapter. It's the functionality of the adapter to

* authenticate any un-authenticated requests - LOGIN
* verify an authenticated request - TOKEN VERIFICATION
* handle logout requests- LOGOUT

Post authentication, the framework expects the authentication adapter to set 'xenon-auth-cookie' cookie or 'x-xenon-auth-token' header with a token string so that the subsequent requests will be considered as authenticated and will be routed to the authentication adapter only for verification of the token.

## Pre-requisites for external authentication

You will need to have the following pre-requisites for enabling external authentication in your
application.

1) Similar to enabling basic authentication, external authentication will come into play only when the authorization is enabled.

2) You will need to write a xenon authentication adapter for your external authentication provider. The framework will need an instance of this adapter to be provided to the ServiceHost before starting your service host. 

There could be multiple implementations of the authentication adapter based on the external authentication provider, however the framework currently supports using only one external authentication adapter at a time.

## Writing a xenon authentication adapter
This section will talk in detail on how to go about writing an example xenon authentication adapter, focussing mostly on the contracts between the framework and the adapter.

There is already an example authentication adapter in the xenon-samples module, which is mostly self explanatory and clearly calls out the functional expectations from the adapter. Please take a look.
_code: [SampleAuthenticationService.java](https://github.com/vmware/xenon/blob/master/xenon-samples/src/main/java/com/vmware/xenon/services/samples/SampleAuthenticationService.java)_

As you see its just a stateless xenon service, however a real implementation of the adapter will internally talk to configured external authentication provider.

### Handling un-authenticated requests in authentication adapter
As said above the framework will route all un-authenticated requests to the adapter, if you want all un-authenticated requests to be redirected to a different url which could be your own custom login screen or the the login page of the external authentication provider you could do that by overriding the queueRequest(Operation op) method of the stateless service.

See the example code snippet below which is just doing a redirection to VMware site, actual implementations might redirect a login screen of an external authenticaion provider or login screen of the application.

```java
public class ExampleAuthenticationService extends StatelessService {
    /**
     * Override this method to specify an URL to redirect any un-authenticated requests.
     */
    @Override
    public boolean queueRequest(Operation op) {
        // its important to only redirect un-authenticated requests for this service
        if (op.getUri().getPath().equals(getSelfLink())) {
            return false;
        }

        // the sample says redirect to vmware.com, it could should be the url of the external
        // authentication provider or your application's login screen

        // the code to remember the actual url requested before redirection can go in here,
        // so that the authentication service can respond with a redirect after
        // authentication later

        op.addResponseHeader(Operation.LOCATION_HEADER, "http://www.vmware.com");
        op.setStatusCode(Operation.STATUS_CODE_MOVED_TEMP);
        op.complete();
        return true;
    }
}
```
Redicrection will follow with a request for login which could be initiated from the external authentication provider or from your login screen once you have the creentials.

### Handling login requests in authentication adapter
Login requests could be made to the authentication adapter directly or as a callback from the external authentication provider. While generally the login requests are POSTs, in case of callbacks the adapter might have to support login requests on GET also.

The framework expects the following behaviour for login requests in the adapter:
* Default behaviour of handleGet() and handlePost() will be for login, unless the request has pragma headers which tell to do token verification or logout. However requests can mandate login by specifying the Operation.PRAGMA_DIRECTIVE_AUTHENTICATE pragma header

* After successful login, get a token from the external authentication provider or generate one locally based on the authentication type

* Create a user document in xenon if one does not exist already representing the logged in user in an external source. During creation of the user make sure the user is assigned to the right UserGroupService for managing the authorization

* Build a xenon Claims object based on the information obtained by cracking the token retrieved from the external authentication provider or based on information available locally. Make sure to set the subject as the userLink of logged in user.

* Use the Claims object and token to build a xenon AuthorizationContext and set it on the service, which will eventually set it on the Operation. Make sure to set propagateToClient as true on the AuthorizationContext to make sure NettyClientRequestHandler will propagate the generate token to the clients as 'xenon-auth-cookie' cookie or 'x-xenon-auth-token' header

See the example code snippet below which is showing how to handle login requests in the xenon adapter.

```java
public class ExampleAuthenticationService extends StatelessService {
    // this example service will use the following hard coded string as access token
    public static final String ACCESS_TOKEN = "valid_token_string";

	/**
     * GET method needs to be implementing the LOGIN requests only, its useful to expose
     * LOGIN in GET to support redirection from external authentication providers after user
     * authenticates against them. Since we need to support external redirections we don't
     * rely on any pragma here.
     */
    @Override
    public void handleGet(Operation op) {
        // actual implementation will expect this to be called from an external
        // authentication provider and use any kind of code shared by it to
        // get an accessToken and then use the access token to create an authorization
        // context and use it.

        // just use the predefined ACCESS_TOKEN to create an authorization context
        // and set it.
        associateAuthorizationContext(this, op, ACCESS_TOKEN);
        op.complete();
    }

	@Override
    public void handlePost(Operation op) {
        // the example code just focusses on showing how to handle login requests
        // since they are default behaviour no need to check for Pragmas
        associateAuthorizationContext(this, op, ACCESS_TOKEN);
        op.complete();
    }

    private void associateAuthorizationContext(Service service, Operation op, String token) {
        // actual implementations will need to generate the Claims object
        // by decoding the access token provided from the external authentication provider

        // the sample service is using a locally created Claims object
        Claims claims = getClaims();

        AuthorizationContext.Builder ab = AuthorizationContext.Builder.create();
        ab.setClaims(claims);
        ab.setToken(token);

        // this is required for the NettyClientRequestHandler to propagate the
        // access token as headers to the clients.
        ab.setPropagateToClient(true);

        // associate resulting authorization context with operation.
        service.setAuthorizationContext(op, ab.getResult());
    }

    private Claims getClaims() {
        Claims.Builder builder = new Claims.Builder();
        builder.setIssuer(AuthenticationConstants.DEFAULT_ISSUER);

        // the claims object has to be associated with a valid userLink as subject,
        // in case of actual implementation this user docuemnt will need to be created if
        // one does not exist
        builder.setSubject(SystemUserService.SELF_LINK);
        return builder.getResult();
    }
}
```

### Handling token verification requests in authentication adapter
Token verification requests will be POST requests from the framework, with the Operation.PRAGMA_DIRECTIVE_VERIFY_TOKEN pragma in the header. Generally verification requests are made to the adapter when the application recieves a request with a token, but it did not find a AuthorizationContext corresponding to the token

The framework expects the following behaviour for token verification requests in the adapter:
* Token verification will be a POST request, adapter will look for the Operation.PRAGMA_DIRECTIVE_VERIFY_TOKEN pragma header in the request before proceeding with verification

* After token verification adapter has to clear off the Operation.PRAGMA_DIRECTIVE_VERIFY_TOKEN pragma

* Look for the token to be verified in 'xenon-auth-cookie' cookie or 'x-xenon-auth-token' header. You can use BasicAuthenticationUtils.getAuthToken(Operation) to read the token from the request

* Crack the token and check if it has not expired. If the token was provided by an external authentication provider contact the provider to check if the token has not been revoked

* Create a user document in xenon if one does not exist already representing the logged in user in an external source. During creation of the user make sure the user is assigned to the right UserGroupService for managing the authorization. This is required in a service to service communication using the same authentication provider

* Build a xenon Claims object based on the information obtained by cracking the token. Make sure to set the subject as the userLink of logged in user. Send the Claims object as a response body

See the example code snippet below which is showing how to handle token verification requests in the xenon adapter.

```java
public class ExampleAuthenticationService extends StatelessService {
    @Override
    public void handlePost(Operation op) {
        // the example service is just doing a token verification based on pragma header

        if (!op.hasPragmaDirective(Operation.PRAGMA_DIRECTIVE_VERIFY_TOKEN)) {
            op.fail(new IllegalStateException("Invalid request"));
            return;
        }
        op.removePragmaDirective(Operation.PRAGMA_DIRECTIVE_VERIFY_TOKEN);
        String token = BasicAuthenticationUtils.getAuthToken(op);
        if (token == null) {
            op.fail(new IllegalArgumentException("Token is empty"));
            return;
        }

        // Actual implementation will crack the token, check expiry and if needed also
        // check for token revocation
        if (token.equals(ACCESS_TOKEN)) {
        	// look for getClaims()
            Claims claims = getClaims();
            op.setBodyNoCloning(claims);
            op.complete();
            return;
        }
        op.fail(new IllegalArgumentException("Invalid Token!"));
    }
}
```
### Handling logout requests in authentication adapter
Logout will be a POST request, you can route the logout requests with Operation.PRAGMA_DIRECTIVE_AUTHN_INVALIDATE pragma header. Handling logout in the adapter will involve a logout request to the external authentication provider if any.

The framework expects the following behaviour for logout requests in the adapter:

* Do an actual logout with an external authentication provider if any

* Check if there is an AuthorizationContext set on the request, if so set a new AuthorizationContext with a Claims object set with expirationTime as ZERO for the corresponding token.


An example code snippet of handling logout is shown below.

```java
public class ExampleAuthenticationService extends StatelessService {
    @Override
    public void handlePost(Operation op) {
        // the example service is just doing logout based on the pragma header

        if (!op.hasPragmaDirective(Operation.PRAGMA_DIRECTIVE_AUTHN_INVALIDATE)) {
            op.fail(new IllegalStateException("Invalid request"));
            return;
        }
        op.removePragmaDirective(Operation.PRAGMA_DIRECTIVE_AUTHN_INVALIDATE);

        // actual implementation will involve a logout request to the
        // external authentication provider, followed by setting the expiration time
        // to ZERO in the loal AuthorizationContext

        // check if there is an AuthorizationContext set, if not return
        if (op.getAuthorizationContext() == null) {
            op.complete();
            return;
        }

        // get a claims object with ZERO as expiration time
        Claims claims = getClaims();
        String token = BasicAuthenticationUtils.getAuthToken(op);
        if (token == null) {
            op.fail(new IllegalArgumentException("Token is empty"));
            return;
        }

        // set the new AuthorizationContext with new Claims object

        AuthorizationContext.Builder ab = AuthorizationContext.Builder.create();
        ab.setClaims(claims);
        ab.setToken(token);
        ab.setPropagateToClient(true);

        // Associate resulting authorization context with operation.
        service.setAuthorizationContext(op, ab.getResult());
        op.complete();
    }

    private Claims getClaims() {
        Claims.Builder builder = new Claims.Builder();
        builder.setIssuer(AuthenticationConstants.DEFAULT_ISSUER);

        // the claims object has to be associated with a valid userLink as subject,
        // in case of actual implementation this user docuemnt will need to be created if
        // one does not exist
        builder.setSubject(SystemUserService.SELF_LINK);

        // set ZERO as expirationTime
        builder.setExpirationTime(0);
        return builder.getResult();
    }
}
```

## Consuming a xenon authentication adapter
This section talks about consuming an implementation of authentication service in your custom service host.

The developer of the custom service host will need to set the authentication service on ServiceHost.java during initialization of the service host. You can find the sample code below
on how to do it.

```java
public class SampleHost extends ServiceHost {

    public static void main(String[] args) throws Throwable {
        SampleHost h = new SampleHost();
        h.initialize(args);

        // set the authentication service here
        // lets use the SampleAuthenticationService as an example
        h.setAuthenticationService(new SampleAuthenticationService());

        // your implementation of authentication service might take some arguments
        // in it's constructor specific to its external authetication provider

        // start the core and custom services
        // you don't have to start the authentication service explicitly
        // the framework will take care of starting it
        h.start();

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            h.log(Level.WARNING, "Host stopping ...");
            h.stop();
            h.log(Level.WARNING, "Host is stopped");
        }));
    }
}
```

## Co-existence of basic and external authentication
Whether an external authentication service is provided or not, the basic authentication will be present. So all the previosly generated tokens from basic authentication will continue to work. You could also generate new access tokens on local user authentication.


