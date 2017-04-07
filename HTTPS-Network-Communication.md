## Using HTTPS for Network Communication

Xenon provides the option to use HTTPS for Network Communication. This includes both, when a client communicates with a Xenon Host, and when one Xenon host communicates with another Xenon host (for Group Membership, Forwarding or Replication). This option is useful when running Xenon in an environment with untrusted clients that should be blocked from having direct access to a Xenon host.

To setup a Xenon host with HTTPS you will need the following:
* An X.509 certificate chain file in PEM format.    
* A PKCS#8 private key file in PEM format.
* The password of the private key file (above), if it's password-protected.

Since each Xenon Host acts as both, a Listener for incoming requests and a Client for making requests to other Xenon-hosts, it is important that the X.509 Certificate(s) used for the rest of the Xenon-Hosts, should be registered with either the default trust-store on the machine, or you can create a custom trust-store and use that at Host start up time.

For prototyping, you can use the certificate files checked-in in Xenon git-repo (below). These files are also used by Xenon tests:
* /xenon/xenon-common/src/test/resources/ssl/trustedcerts.jks
* /xenon/xenon-common/src/test/resources/ssl/server.crt
* /xenon/xenon-common/src/test/resources/ssl/server.pem

Once you have the files created, you can start a Xenon host with HTTPS communication by running the below commands:
```
java \
  -Djavax.net.ssl.trustStore=./xenon-common/src/test/resources/ssl/trustedcerts.jks \
  -Djavax.net.ssl.trustStorePassword=changeit \
  -jar xenon-host/target/xenon-host-*-with-dependencies.jar \
  --peerNodes=https://127.0.0.1:8000,https://127.0.0.1:8001 \
  --port=-1 --securePort=8000 \
  --keyFile=./xenon-common/src/test/resources/ssl/server.pem \
  --certificateFile=./xenon-common/src/test/resources/ssl/server.crt \
  --adminPassword=changeme
```
At a different terminal, start a second host, on a different port, making sure you supply the proper port in the peerNodes argument:
```
java \
  -Djavax.net.ssl.trustStore=./xenon-common/src/test/resources/ssl/trustedcerts.jks \
  -Djavax.net.ssl.trustStorePassword=changeit \
  -jar xenon-host/target/xenon-host-*-with-dependencies.jar \
  --peerNodes=https://127.0.0.1:8000,https://127.0.0.1:8001 \
  --port=-1 --securePort=8001 \
  --keyFile=./xenon-common/src/test/resources/ssl/server.pem \
  --certificateFile=./xenon-common/src/test/resources/ssl/server.crt \
  --adminPassword=changeme
```

Please refer to [Starting Xenon Host](Start-Xenon-Host) page for details on other available options/arguments to start the Xenon host.