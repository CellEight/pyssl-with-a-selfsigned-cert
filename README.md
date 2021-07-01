# Using SSL in Python with A Self-Signed Certificate
Often times when you're creating a secure networked application in python you will wish to encrypt the traffic being sent between client and server using ssl.
The best way of going about this is in general to use the standard ssl library included in the standard python distribution.
The ssl library has great documentation that covers almost everything you need to know.
However if you are trying to just run your application on your local system rather than on a web server with a certificate purchased from a certificate authority it leaves you somewhat out in the cold.
To get ssl working on your local machine or local network you are going to require to generate a self signed certificate and to then configure your client a server to use it.
This process, if undertaken without a guide of some description, is surprisingly infuriating!
As far as I can tell no such guide exists so as I have just spent an several hours scouring the internet to work out how to do this I figured it would be best to write one myself.
Hopefully to help others but also so that when I inevitably forget how it is done I can return to this guide and avoid spending another afternoon tearing my hair out in a fit of incandescent rage at the python devs.

## Generating a Self Signed Certificate

The first step in the process which should be undertaken before all else is the generation of a self-signed certificate.
To do this you will need openssl installed, create a directory with a name such as `certs` in your project directory, navigate there and being by creating a file called `ssc.cnf` and using your preferred text editor place the following configuration information into it:
```
[req]
default_bits  = 4096 
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
countryName = XX
stateOrProvinceName = N/A
localityName = N/A
organizationName = Self-signed certificate
commonName = 120.0.0.1: Self-signed certificate
[req_ext]
subjectAltName = @alt_names
[v3_req]
subjectAltName = @alt_names
[alt_names]
IP.1 = <YOUR SERVER'S IP>
```
Replace `<YOUR SERVER'S IP>` with the ip `127.0.0.1` if you will just be running your server on your local machine or the network ip of your server if you are testing your software on a LAN. 
You could also use this for testing over the internet but at that point perhaps you might as well just get a certificate authority to issue a real cert.
Next run the following command:

```bash
openssl req -nodes -x509 -newkey rsa:4096 -keyout test_key.pem -out test_cert.pem -days 365 -config san.cnf
```

This command generates your self-signed certificate as well as the private key your server will use to prove its identity to clients.
For some reason I am not privy to, the ssl library does not let you pass the certificate and the private key to your server as two separate files.
They must instead be combined into a single file consisting of the private key immediately followed by the cert.
This, while somewhat non-obvious on the basis of the ssl library's documentation, if fortunately easy to do:

```bash
cat ./test_cert.pem >> test_key.pem
```

## Adding SSL to an Application

Now that we have our self-signed certificate set up correctly we can actually get to the configuration of our application.
The ssl library works by creating a `SSLContext` object which encapsulates all the ssl configurations we might wish to make and the using the wrap\_socket method of this object to convert a ordinary python socket object into a `sslsocket\_class` object which has an interface much like an ordinary socket but through which all traffic is encrypted.
The configuration of the client and the server are similar but differ in important ways.
We will cover each in turn.

## Configuring the Client

To configure the client we first import the ssl library and create our `SSLContext` instance, pointing it towards the certificate we generated earlier.
```python
import ssl
import socket

context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
context.options |= ssl.OP_NO_TLSv1 | ssl.OP_NO_TLSv1_3
context.load_verify_locations('path/to/cert/test_cert.pem')
```

With our context created and with it configured to use our certificate we may now make a socket and convert it to and ssl socket using that context.
```python
ssl_socket = self.context.wrap_socket(socket.socket(socket.AF_INET, socket.SOCK_STREAM),server_hostname=<YOUR SERVER'S IP>)
```

This ssl socket instance can then be used just like any other socket to connect to your server which we will configure now.

## Configuring the Server

