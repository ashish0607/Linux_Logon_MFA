Description

The goal is to implement 2FA on an unix server for Console/SSH access. Unix uses PAM (Pluggable Authentication Modules) for SSH authentication among other things. PAM’s name speaks for itself, it’s comprised of many modules that can be added or removed as necessary. And it is pretty easy to write your own module and add it to SSH /Logon authentication. After PAM is done with the regular password authentication it already does for SSH, we’ll get it to send an http message/email/SMS with a randomly generated code valid only for this authentication.
You can set up PAM authentication either by using external authentication sources, such as NIS and LDAP, or by using a single, central ObjectServer.


IMPLEMENTATION

Let’s do an ls on /lib/security, this is where the pam modules reside in unix system(ubuntu).
Note: We’re trying to change the authentication mechanism and you  can get locked out. A good idea is to keep a couple of sessions open just in case such situation came up.

Take a look at the code, you’ll see that PAM expect things to be laid out in a certain way. That’s fine, all we care about is where to write our custom code. In our case it starts at line 50. As you can see, the module takes 2 parameters, a Server URL and the size of the code to generate. The URL will be called and passed a code & username. It is this web service that will be in charge of dispatching the code to the user. This step could be done in the module itself but here we have in mind a centrally managed service in charge of dispatching codes to multiple users.

Pre-Requisite 

You need to first update the libraries and require some dev package onto your system:
    apt-get update

    apt-get install build-essential libpam0g-dev libcurl4-openssl-dev

Deploying the code is done as follows:

    gcc -fPIC -lcurl -c Auth_Module.C.c

    ld -lcurl -x --shared -o /lib/security/Auth_Module.C.so Auth_Module.C.o

Do an ls on /lib/security again and you should see our new module, 

For Applying 2 FA at the Ubuntu console :

Edit the common auth file at the given location sudo nano /etc/pam.d/common-auth and add out module call there.

    auth       required     2ndfactor.so base_url=http://serverurl.com/ code_size=5

For Applying 2 FA at the SSH console :

Now let’s edit /etc/pam.d/sshd, this is the file that describes which PAM modules take care of ssh authentication, account & session handling. But we only care about authentication here.

    auth       required     2ndfactor.so base_url=http://serverurl.com/ code_size=5

auth requisite pam_deny.so
auth required pam_permit.so
auth optional pam_cap.so


# Standard Un*x authentication.

@include common-auth


The common-auth is probably what takes care of the regular password prompt so we’ll add our module call after this line as such:

auth       required     2ndfactor.so base_url=http://serverurl.com/ code_size=5


Lastly, edit /etc/ssd/sshd_config and change ChallengeResponseAuthentication to yes. 

/etc/init.d/ssh restart

Try to Logon Using SSH/Console.
