---
layout: post
title: Extracting Android KeyStores from apps
excerpt: "We love private keys, don't we all?"
tags: [Android, Frida, reverse engineering]
image:
  feature: frida-android-keystore/script_example.jpg
---

> _Questo post è disponibile [in italiano](https://muhack.org/talks/frida-android-keystore/)_

SSL pinning is almost omnipresent in android apps, but can be easily defeated using [Xposed modules][1] or [Frida scripts][2] (more on Frida later).

So, what if a dev wanted to be _extra-safe_ about what went through the net?

# SSL (TLS) and SSL Pinning #
First of all a brief explanation of what SSL and SSL pinning are. If you're already familiar with the topic you can skip to the next chapter, nothing new here.

SSL (Secure Sockets Layer) is a protocol made to guarantee an encrypted connection between user (be it an app or a web browser) and server. SSL dates back to 1995 and has been deprecated since 2015, it was replaced by TLS but TLS encrypted traffic is commonly referred as SSL traffic, I'll do the same from now on. The link formed with SSL ensures that all data passed from the client to the server (and vice versa) remain private and integral.

This protocol itself is safe ([Mostly][3]), but does not solve the underlying [_Key Distribution Problem_][4]. When you want to be sure no one is playing MITM and injecting his own certificates, you pin the public key to the host. If you receive a key different from the expected one, probably someone is between you and the server, so you stop the communication.

{:.image-caption}
<figure><a href="{{ site.url }}/images/frida-android-keystore/burp_alerts_no_unpin.jpg"><img src="{{ site.url }}/images/frida-android-keystore/burp_alerts_no_unpin.jpg" alt="Alerts in Burp when not unpinning certificates"></a><figcaption>Alerts in Burp when not unpinning certificates</figcaption></figure>

Once you used one of the well-known methods linked at the beginning of this post, all the errors should disappear and you'll find clear text traffic in your sniffing tool.

# Client-side certificates #
When you __really__ do not want anyone to read the traffic, you can embed client-side certificates in your app and configure the server to require them while performing the TSL handshake. This procedure is explained thoroughly on [IBM website][5].

You can go to great extent and hide the cert somewhere in the app, maybe already encoded as a Java KeyStore file (.jks) with a password to open it. The password could be hidden with some fancy method such as the (now unmaintained) [Cipher.so][6] project.

All these steps are probably enough to wear an attacker which is not motivated enough, because he/she has to find where the keystore is saved, reverse engineer the password and find a way to extract data in an usable format (PKCS12, probably).

This is where Frida comes into play

# Dynamic code instrumentation #
[Frida][7] is a tool which gives us the possibility to hook to classes and modify methods on runtime.<br>Dynamically.<br>With JavaScript.

Yes, it __*is*__ as magical as it sounds.

With the power of dynamic instrumentation we can do anything we want with code. We can output the parameters a function is called with, we can downgrade security on the fly (think about a signature check function) or misuse proper code (think about signing your tampered message with the proper sign function to get it approved by the server). [Brida][8] is a nice example of how this analysis method could ease your life.

# Bringing it together #
Android KeyStores have a `load` method which is used to load (you don't say) the instantiated keystore with some data. This method is overloaded:
- `load(KeyStore.LoadStoreParameter param)`
- `load(InputStream stream, char[] password)`

The app I had to work used the second one with a jks object passed as a stream, so my code is made for this scenario. As you can see the second method accepts a char array as password, which means that, regardless how bad the password is encrypted in the apk file, it is available in memory in plain text. This is good: no more need to reverse engineer some possibly nasty [white box crypto][9].

Frida helps us and supports a nice `overload` method to let us choose which one of the two `load` we'll modify.

[This is the result script][10].

### How it works ###
I'll ignore python code which is just a wrapper for housekeeping, let's go through the js code which is injected in the app.


```javascript
setTimeout(function() {
    Java.perform(function () {
        keyStoreLoadStream = Java.use('java.security.KeyStore')['load'].overload('java.io.InputStream', '[C');

        /* following function hooks to a Keystore.load(InputStream stream, char[] password) */
        var keyStoreLoadStream.implementation = function(stream, charArray) {

            /* sometimes this happen, I have no idea why, tho... */
            if (stream == null) {
                /* just to avoid interfering with app's flow */
                this.load(stream, charArray);
                return;
            }

            /* just to notice the client we've hooked a KeyStore.load */
            send({event: '+found'});

            /* read the buffer stream to a variable */
            var hexString = readStreamToHex (stream);

            /* send KeyStore type to client shell */
            send({event: '+type', certType: this.getType()});

            /* send KeyStore password to client shell */
            send({event: '+pass', password: charArray});

            /* send the string representation to client shell */
            send({event: '+write', cert: hexString});

            /* call the original implementation of 'load' */
            this.load(stream, charArray);

            /* no need to return anything */
        }
    });
},0);

/* following function reads an InputStream and returns an ASCII char representation of it */
function readStreamToHex (stream) {
    var data = [];
    var byteRead = stream.read();
    while (byteRead != -1)
    {
        data.push( ('0' + (byteRead & 0xFF).toString(16)).slice(-2) );
                /* <---------------- binary to hex ---------------> */
        byteRead = stream.read();
    }
    stream.close();
    return data.join('');
}
```
>`setTimeout`

Is a Frida function to run our code after a 0ms delay
>`Java.perform`

Runs some java code in Frida's Java runtime

>`Java.use('java.security.KeyStore')['load'].overload('java.io.InputStream', '[C');`

We choose which class we want to hook to (`java.security.KeyStore`) and which method of this class (`load`). Also, we specify that we only need the method which takes an InputStream and a Char as inputs. This method will be referenced via the `keyStoreLoadStream` variable.

>`keyStoreLoadStream.implementation = function(stream, charArray)`

Once we have selected the method we specify what we want to do with it: we want to change the implementation with our custom function. `stream` and `charArray` are formal parameters that can be referenced from inside the function.

>`readStreamToHex (stream);`

The parameter `stream` is passed over to `readStreamToHex` which uses the Java function `read()` to read the next byte in the stream until an error is received (`byteRead !=-1`). One by one bytes are converted to their ASCII hex counterpart and pushed into an array that will be joined and returned to the caller on finish.

>```javascript
>send({event: '+type', certType: this.getType()});
>send({event: '+pass', password: charArray});
>send({event: '+write', cert: hexString});
>```

When we're done reading the stream we use `this.getType()` from Java to get the cert type, the result is passed to the client in order to inform the user. If the type is PKCS12 we also set the ext to .jks in client.<br>
Password and the ASCII representation of InputStream are now passed to the client and the stream is written to file.

>`this.load(stream, charArray);`

As a last step the real implementation of `load` is called with the original parameters so that app's flow is not interrupted and the certificate gets actually loaded into the keystore.

# Result #
First you have download and run `./frida-server &` on your target device, then execute my script on your computer and you should get something like this

{:.image-caption}
<figure><a href="{{ site.url }}/images/frida-android-keystore/script_example.jpg"><img src="{{ site.url }}/images/frida-android-keystore/script_example.jpg" alt="Script output example"></a><figcaption>Script output example</figcaption></figure>

We now have a keystore entity saved on client's hard disk. This entity is (probably, with current code) a jks file which should be converted to a pkcs12 binary certificate to be used with other tools. You can perform the conversion with keytool, a part of your Java SDK install (it should be bundled also with the JRE, but I'm not sure).

```bash
keytool -keystore keystore0.jks -list
keytool -importkeystore -srckeystore keystore0.jks -destkeystore dest_pkcs12_crt.p12 -deststoretype PKCS12 -srcalias CERT_ALIAS -deststorepass YOURPASS -destkeypass YOURPASS
```

The first command will give you a list of available aliases in the keystore and you should supply them one by one to the second command to extract all the certificates. You'll need to specify a password for the newly created certificate.

{:.image-caption}
<figure><a href="{{ site.url }}/images/frida-android-keystore/burp_traffic_sniff.jpg"><img src="{{ site.url }}/images/frida-android-keystore/burp_traffic_sniff.jpg" alt="Clear traffic"></a><figcaption>Clear traffic</figcaption></figure>

Profit

[1]:	https://github.com/Fuzion24/JustTrustMe
[2]:	https://techblog.mediaservice.net/2017/07/universal-android-ssl-pinning-bypass-with-frida/
[3]:	https://tools.ietf.org/html/rfc7457
[4]:	https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning#Patient_0
[5]:	https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_7.1.0/com.ibm.mq.doc/sy10660_.htm
[6]:	https://github.com/MEiDIK/Cipher.so
[7]:	https://www.frida.re/
[8]:	https://github.com/federicodotta/Brida
[9]:	http://www.whiteboxcrypto.com/
[10]:	https://gist.github.com/ceres-c/cb3b69e53713d5ad9cf6aac9b8e895d2
