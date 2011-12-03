========
Features
========

This project includes a collection of scripts to help you
run j2me MIDlets of your choosing on j2me-capable phones
without restrictions.

----------
Background
----------

It seems that most firmwares on nokia s40 phones do not allow you to install
your own code-signing certificates using the phone's provided user interface.

I discovered this after I bought a nokia phone with a TMobile-USA customized
firmware on eBay. Furthermore, according to `this page on the nokia developer website
<http://www.developer.nokia.com/Community/Wiki/T-Mobile_U.S._Java_security_domains>`_,
T-Mobile does not even allow you to run MIDlets signed by code-signing certificates
that are in-turn signed by popular certificate authorities (Thawte, VeriSign, etc).

It is possible to install your own MIDlets on these devices, but only MIDlets
signed with the T-Mobile root certificates on the devices are allowed to access
all the useful, protected j2me APIs that require specific permissions.

This project provides scripts to help you work around these restrictions.

----------------------
provided scripts
----------------------

**alter_ext_info_file**:
    This will modify an ``ext_info.sys`` file, taken from the user certificates
    directory on your device, to give your user-installed x509 v3 CA
    certificate full code-signing permissions, thus enabling you to run MIDlets
    signed with a certificate of your choosing on your nokia phone.


**gen_ca_v3_cert**:
    A utility script to help you generate a certificate and a java keystore
    that you can use to sign MIDlets on your phone.

======================================
Documentation
======================================

-------------------------------------------
How to install Certificates on Nokia Phones
-------------------------------------------

~~~~~~~~~~
Disclaimer
~~~~~~~~~~

I have no idea if this works on all nokia S40 phones or firmwares. I think it
probably works on many S40 phones and possibly some S60 phones (?). I have only
tested this with a Nokia 5130 XM running T-Mobile USA-customized firmware.

~~~~~
Steps
~~~~~

- Use gammu to print out the entire filesystem of your phone::

    gammu getfilesystem -flatall

- Find the location of the user certificates directory (grep for ``ext_info.sys``
  in the above). On my phone, this directory is (all commands below will assume
  this location)::

    d:/predefhiddenfolder/certificates/user

- Generate your own DER-formatted x509 code signing certificate. If this is a
  v3 certificate, it *MUST* include CA:true in the x509 v3 constraints
  (critical is optional) . You can verify that your cert is compliant by running::
  
    openssl x509 -inform DER -in your_cert.cer -noout -text
  
  you should see this in the output::

    X509v3 Basic Constraints: critical
        CA:TRUE

You can use the ``gen_ca_v3_cert`` script to generate your x509 cert. Run it with
no arguments for usage information.

- Use gammu to copy your cert to your user cert dir::

    gammu addfile d:/predefhiddenfolder/certificates/user yourcert.cer

- reboot your phone

- Download the updated ``ext_info.sys`` file from your phone with gammu, and
  rename it::

    gammu getfiles d:/predefhiddenfolder/certificates/user/ext_info.sys
    mv ext_info.sys ext_info.sys.orig

- Use ``alter_ext_info_file`` to escalate the permissions of your certificate,
  as defined in the ``ext_info.sys`` file::

    alter_ext_info_file ext_info.sys.orig ext_info.sys

- Replace the ext_info.sys on the phone with your new file::

    gammu deletefiles d:/predefhiddenfolder/certificates/user/ext_info.sys
    gammu addfile d:/predefhiddenfolder/certificates/user ext_info.sys

- Reboot your phone

- Copy your jad + jar to the phone (to the SD card or install them with ``gammu
  nokiaaddfile APPLICATION AppName``)

You can now run your MIDlet with any permissions that you declare in your jad
file.

======================================
Acknowledgements
======================================

----------------
Acknowledgements
----------------

When I started experimenting with my nokia mobile and discovered gammu, I was
quickly able to find the core certificates installed on the phone. It was
immediately apparent that the ``ext_info.sys`` file contained metadata that
defined, among other things, the permissions granted to the certificates.

A web search for ``nokia certificate ext_info.sys`` brought me to `this page
authored by Thomas Zell <http://www.tzell.mynetcologne.de/cert.html>`_, in
which he wrote up his notes on installing certificates on nokia
phones. Zell cites that his notes are based on reverse engineering done
by some Russian guy who goes by the handle *exp*.

The solution posted by Zell to the nokia-certificate restriction, along with
the cert from exp, did not work on my mobile. I believe this is due to the
additional T-Mobile USA restrictions described above. I had to further study
the permissions associated in the core ``ext_info.sys`` file that are
associated with the T-Mobile certificates and encode those in my script that
fixes ext_info.sys. If you are using a nokia phone that does not have a
T-Mobile USA firmware, chances are you can use the solution described by Zell
along with exp's cert instead of generating your own cert and fixing the
``ext_info.sys`` file with ``alter_ext_info_file``.

So, thanks to Thomas Zell, exp.

Thanks to all the `gammu-project <http://wammu.eu/>`_ developers for writing
some awesome software to interface with these nokia handsets.

I'd also like to thank the developers of the nokia S40 mobile firmware. Aside
from the annoying MIDlet lockdown, this is really a great platform for
running j2me applications. I'm impressed with the obex support and lots of
little miscellaneous capabilities of my nokia 5130 XM.

==================================
Other Miscellaneous Examples
==================================

Generate a self-signed x509 v3 CA/code-signing certificate with the default
configuration options. **NOTE**: *to use a secure private key, you should copy
the cert_config, edit the copy, change the private key password in the copy,
and run gen_ca_v3_cert against your copy*.::

    out_dir=/tmp/certs
    mkdir -p "$out_dir"
    bin/gen_ca_v3_cert config/cert_config config/openssl.conf "$out_dir"

You can sign the certificate with something like this::
from `this mynetcologne.de blog entry
<http://www.tzell.mynetcologne.de/cert.html>`_::

    alias=dustbowl
    KEYSTORE_PASS='mys3cr3tk3st0r3p@ss'
    KEY_PASS='mypr1v@t3k3yp@ss'

    java -jar JadTool.jar -addjarsig -alias "$alias" \
        -storepass "$KEYSTORE_PASS" -keystore dustbowl.jks \
        -keypass "$KEY_PASS" \
        -inputjad midlet.jad -outputjad midlet.jad \
        -jarfile midlet.jar

I personally use `the excellent antenna ant plugin
<http://antenna.sourceforge.net/>`_ with `apache ant <http://ant.apache.org/>`_
to build and sign my MIDlet jad file. You can use the `wtksign ant task
<http://antenna.sourceforge.net/wtksign.php>`_ to sign your jad with something
like this::

    <target name='sign' description='Sign the jar and jad files with our code signing certificate' if="${sign.app}">
        <wtksign keystore="${keystore.file}"
                 jarfile="${jarfile}"
                 jadfile="${jadfile}"
                 storepass="${keystore.pass}"
                 certpass="${code_signing_key.pass}"
                 certalias="${code_signing_cert.alias}"/>
    </target>
