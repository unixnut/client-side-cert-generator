Client-side cert generator
==========================

Generates PKCS#12 certificates (.pfx files) with or without a
passphrase, for use in a browser.

Installation
------------
Install Make and OpenSSL.

Either copy the files `Makefile`, `openssl.cnf` and `vars` into a directory,
and modify `vars`; OR

Use [Cookiecutter](https://github.com/cookiecutter/cookiecutter) with
the template in the repo by running
  
  **`cookiecutter gh:unixnut/client-side-cert-generator`**

...which will prompt you for a directory to create and the name of the
organisation to use in the certificates.

Usage
-----
Edit `vars` if you haven't already, to set the organisation.  You may
wish to edit `openssl.cnf` to set the suburb and e-mail defaults.

Run the following commands for a certificate without a passphrase:

    . vars
    make zip CLIENT=Firstname

*Note*: If the CA hasn't been generated already, it will be generated
first, and so the first set of OpenSSL prompts are for this, not the
cert.

*Note*: Don't just press enter when asked to commit the cert as
OpenSSL presents a y/n prompt and the default is no.

Nginx setup
-----------
Copy `ca.crt` to the server and put this in your vhost file (with the
actual path):

    ssl_verify_client on;
    ssl_client_certificate /path/to/ca.crt;
    if ($ssl_client_verify != "SUCCESS") {
        return 403;
    }

Apache setup
------------
Copy `ca.crt` to the server and put this in your vhost file (with the
actual path):

***TBA***
