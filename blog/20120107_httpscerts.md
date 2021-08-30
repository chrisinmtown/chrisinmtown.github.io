# Server and Client Certificates in HTTPS for Apache Client

### 7 Jan 2012

I was working on a client to access a RESTful (representation
state transfer AKA http) web service.  Maybe that's just the buzzword
of choice these days, but the system seems to conform to Wikipedia's
list of 
[REST architecture](http://en.wikipedia.org/wiki/Representational_state_transfer)
I'm using version 4.1.2 libraries
provided by the [Apache HttpComponents](http://hc.apache.org)
project. All straightforward so far, right?

Two complicating factors made this a bit interesting.  First, the
server requires access via HTTPS, and for that it uses a self-signed
server certificate.   This requirement can be met in a couple of
ways: either the HttpClient can be told to trust all servers no matter
what, or the server certificate can be cached locally for comparison.
Apache offers 
[example code](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientCustomSSL.java)
to demonstrate caching a self-signed certificate so that was no sigificant problem. 

The second requirement, presenting a user certificate to the server,
was a bit tricker.  I found example code at the Apache site, but it was
for version 3 and no longer works in v4.  Stackoverflow offered
[pieces of code](http://stackoverflow.com/questions/3375121/mutual-authentication-with-x509-certificates-using-httpclient-4-0-1)
but not the full solution. A
[blog post](http://drumcoder.co.uk/blog/2011/oct/18/httpclient-client-side-certificates)
by Tim Sawyer was extremely helpful in pointing out that
this scenario requires both a *keystore* and a *truststore*, but I
still struggled to get the keystore and truststore files set up
appropriately.  I find the Java keytool fairly inscrutable but that's
prolly because I'm not a crypto person.  And just to make it fun, the
javadoc for the critical constructor in the
[SSLSocketFactory class](http://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/conn/ssl/SSLSocketFactory.html)
is utterly free of any description, and the parameter names are barely helpful. 

I had to save the server's certificate in a Java keystore file.
Because this file holds the server info, the proper term is a
*truststore*, which is the term used in the Apache HttpClient javadoc.
The keystore must show that it has a "trustedCertEntry."  This is the
incantation I used to build a server truststore file in Java Keystore
("JKS") format using the keytool command that comes with Java; after
keytool shows the details you have to tell it to trust the
certificate:

    keytool -importcert -alias "server" -file server.crt -keystore server.keystore -deststorepass password1

I had to save the client's private key in a Java keystore file, and it
must appear in that keystore as a "PrivateKeyEntry" (not a certificate
entry).  The client key was available in a PKCS12 (".p12") format and
that was critical.  I learned from googling that keytool can read a
PKCS12 file and import its contents appropriately.  This is the
incantation I used to build a client keystore file in JKS format using
the keytool command; again you have to approve import of the data:

    keytool -v -importkeystore -srckeystore user.p12 -srcstoretype PKCS12 -srcstorepass changeit -destkeystore user.keystore -deststoretype JKS -deststorepass password2


Along the way I hit various stumbling blocks of course.  

Initially I supplied the wrong server certificate, and I hit this
exception: 

    javax.net.ssl.SSLPeerUnverifiedException: peer not authenticated

At least once I gave the wrong password for a keystore and this
exception is what happens:

    java.io.IOException: Keystore was tampered with, or password was incorrect

While I knew that the private key is protected by a password, I didn't
quite grasp that this protection is preserved when it's imported into
the destination keystore, which of course is protected by a different
password.  So I supplied the correct password to load the keystore,
but not the right password to decrypt the private key within the
keystore.  That yielded the following exception.

    java.security.UnrecoverableKeyException: Cannot recover key

When I botched the user private key certificate by supplying a
keystore file with the wrong content, I hit this exception:

    org.apache.http.impl.client.DefaultRequestDirector handleResponse
    WARNING: Authentication error: Unable to respond to any of these
    challenges: {}

I used Java version 1.6.0_20, and apparently this VM has an SSL
implementation that blocks SSL renegotation, which presented itself as
this exception:

    javax.net.ssl.SSLException: HelloRequest followed by an unexpected handshake message

I frankly don't understand all the ramifications of this, but again
asking the cloud yielded the answer.  Supposedly other versions don't have
this problem but I have not yet tested them. Launching the program
with this additional VM argument turns this off.  But note that his
only appears *if some other problem is also present*; it's not
necessary when all the keystores and passwords are correct.

    -Dsun.security.ssl.allowUnsafeRenegotiation=true

Putting all the pieces together yields the following demonstration
code. It traps all the exceptions that I hit and tries to give helpful
messages :).  Forgive me for assuming UTF-8 encoding for the server
response!  Please drop me a line if it helps you. 

```
package io.github.chrisinmtown.httpscerts;

import java.io.File;
import java.io.FileInputStream;
import java.net.URI;
import java.security.KeyStore;
import java.security.UnrecoverableKeyException;

import javax.net.ssl.SSLException;
import javax.net.ssl.SSLPeerUnverifiedException;

import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.conn.scheme.Scheme;
import org.apache.http.conn.ssl.SSLSocketFactory;
import org.apache.http.impl.client.DefaultHttpClient;
import org.apache.http.util.EntityUtils;

/**
 * Demonstrates use of the Apache HTTP Client version 4 to access a web site via
 * HTTPS, with special conditions:
 * <OL>
 * <LI>The server presents a self-signed certificate (not signed by a trusted
 * certificate authority). The certificate is available in a .crt file (x509
 * format?). To allow this, the caller must supply a truststore file containing
 * the expected server certificate.
 * <LI>The user must supply a private key to the server for authentication. The
 * key is available in PCKS12 format. To enable this, the caller must supply a
 * keystore file containing the expected user certificate.
 * </OL>
 * Built and tested using Apache HTTP Components version 4.1.2.
 * 
 * Used Java's keytool to create the server truststore from a .crt file:
 * 
 * <PRE>
 *   keytool -importcert -alias "server" -file server.crt -keystore server.keystore -deststorepass password1
 * </PRE>
 * 
 * Used Java's keytool to creaet the client keystore from a .p12 file:
 * 
 * <PRE>
 *    keytool -v -importkeystore -srckeystore user.p12 -srcstoretype PKCS12 -srcstorepass changeit -destkeystore user.keystore -deststoretype JKS -deststorepass password2
 * </PRE>
 * 
 * Note that JDK versions 1.6.0_19 thru JDK 1.6.0_23 prohibit SSL renegotiation.
 * Reopen this possible security hole by supplying this JVM argument:
 * -Dsun.security.ssl.allowUnsafeRenegotiation=true
 * 
 * This is a revised version of ClientCustomSSL, an example program contributed
 * to the Apache Http Client project, and available here:<BR>
 * http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/
 * http/examples/client/ClientCustomSSL.java
 *
 */
public class ServerClientCustomSSL {

    public final static void main(String[] args) throws Exception {
    if (args.length != 6) {
        System.err
        .println("Usage: server-truststore-file-name server-truststore-password client-keystore-file-name client-keystore-password client-key-password target-URI");
        return;
    }

    File truststoreFile = new File(args[0]);
    if (!truststoreFile.exists() || !truststoreFile.isFile()) {
        System.err.println("Not found or not a file: "
                   + truststoreFile.getPath());
        return;
    }
    System.out.println("Truststore file with server cert is "
               + truststoreFile.getPath());

    String truststorePassword = args[1];
    if (truststorePassword.length() == 0) {
        System.err.println("Empty truststore password, giving up");
        return;
    }
    System.out.println("Truststore password is " + truststorePassword);

    File keystoreFile = new File(args[2]);
    if (!keystoreFile.exists() || !keystoreFile.isFile()) {
        System.err.println("Not found or not a file: "
                   + keystoreFile.getPath());
        return;
    }
    System.out.println("Keystore file with client private key is "
               + keystoreFile.getPath());

    String keystorePassword = args[3];
    if (keystorePassword.length() == 0) {
        System.err.println("Empty keystore password, giving up");
        return;
    }
    System.out.println("Keystore password is " + truststorePassword);

    String privateKeyPassword = args[4];
    if (privateKeyPassword.length() == 0) {
        System.err.println("Empty private key password, giving up");
        return;
    }
    System.out.println("Private key password is " + privateKeyPassword);

    URI targetURI = new URI(args[5]);
    String protocol = targetURI.getScheme();
    if (!"https".equals(protocol)) {
        System.err
        .println("URI does not begin with expected protocol name");
        return;
    }
    System.out.println("URI to fetch is " + targetURI.toString());

    DefaultHttpClient httpclient = new DefaultHttpClient();
    try {
        // The expected server certificate must be in a *trust* store.
        // keytool must report "trustedCertEntry" when listing the contents.
        KeyStore trustStore = KeyStore.getInstance(KeyStore
                               .getDefaultType());
        FileInputStream trustStream = new FileInputStream(truststoreFile);
        try {
            System.out.println("Loading server truststore from file "
                    + truststoreFile.getPath());
            trustStore.load(trustStream, truststorePassword.toCharArray());
            System.out.println("Truststore certificate count: "
                    + trustStore.size());
        } catch (Exception ex) {
            System.err.println("Failed to load truststore: "
                    + ex.toString());
            return;
        } finally {
            try {
                trustStream.close();
            } catch (Exception ignore) {
            }
        }

        // The required user key must be in a *key* store.
        // keytool must report "PrivateKeyEntry" when listing the contents.
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        FileInputStream keyStream = new FileInputStream(keystoreFile);
        try {
            System.out.println("Loading client keystore from file "
                    + keystoreFile.getPath());
            keyStore.load(keyStream, keystorePassword.toCharArray());
            System.out.println("Keystore certificate count: "
                    + keyStore.size());
        } catch (Exception ex) {
            System.err.println("Failed to load keystore: " + ex.toString());
            return;
        } finally {
            try {
                keyStream.close();
            } catch (Exception ignore) {
            }
        }

        // Create and register a socket factory for all HTTPS connections
        SSLSocketFactory socketFactory = null;
        try {
        // http://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/index.html
        // This constructor has zero words of documentation in the
        // version 4.1.2 javadoc; I only figured it out by googling.
        socketFactory = new SSLSocketFactory(keyStore,
                             privateKeyPassword, trustStore);
        } catch (UnrecoverableKeyException ke) {
            System.err
                .println("Failed to create SSLSocketFactory, possible wrong password on client private key");
            return;
        }
        // This is the default port number only; others are allowed
        Scheme sch = new Scheme("https", 443, socketFactory);
        httpclient.getConnectionManager().getSchemeRegistry().register(sch);

        HttpGet httpget = new HttpGet(targetURI);
        System.out.println("Executing request " + httpget.getRequestLine());

        HttpResponse response = null;
        try {
            response = httpclient.execute(httpget);
        } catch (SSLPeerUnverifiedException ex) {
            // Message "peer not authenticated" means the server presented
            // a certificate that was not found in the local truststore.
            System.err
                .println("Get failed, possible missing or invalid certificate: "
                    + ex.toString());
            return;
        } catch (SSLException sx) {
            // Renegotiation must be allowed in certain JDK versions via the
            // JVM argument -Dsun.security.ssl.allowUnsafeRenegotiation=true
            System.err
                .println("Get failed, possible missing JVM argument: "
                    + sx.toString());
            return;
        } catch (Exception x) {
            // Something I have not seen (yet)
            System.err.println("Get failed unexpectedly: " + x.toString());
            return;
        }

        HttpEntity entity = response.getEntity();
        System.out.println("----------------------------------------");
        System.out.println("Response status: " + response.getStatusLine());
        if (entity != null) {
            System.out.println("Response content length: "
                    + entity.getContentLength());
            if (entity.getContentLength() > 0) {
                byte[] first100 = new byte[100];
                int howMany = entity.getContent().read(first100);
                // Hope that the encoding is sane
                String hundred = new String(first100, 0, howMany, "UTF-8");
                System.out.println("First " + howMany
                        + " characters of the response:");
                System.out.println(hundred);
            }
        }
        // read the rest and close the stream
        EntityUtils.consume(entity);

    } finally {
        // When HttpClient instance is no longer needed,
        // shut down the connection manager to ensure
        // immediate deallocation of all system resources
        httpclient.getConnectionManager().shutdown();
    }
}
```
