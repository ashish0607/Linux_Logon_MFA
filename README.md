SOLUTION

The goal is to have more than one mean of establishing identity. And as much as possible, the means have to be distinct in order to reduce the chances of having both mechanisms compromised.

Let’s see how to implement 2FA on an Ubuntu server for SSH. Ubuntu uses PAM (Pluggable Authentication Modules) for SSH authentication among other things. PAM’s name speaks for itself, it’s comprised of many modules that can be added or removed as necessary. And it is pretty easy to write your own module and add it to SSH authentication. After PAM is done with the regular password authentication it already does for SSH, we’ll get it to send an email/SMS with a randomly generated code valid only for this authentication. The user will need access to email/cell phone on top of valid credentials to get in.
What is PAM


Pluggable Authentication Modules (PAM) is an integrated UNIX login framework. PAM is used by system entry components, such as the dtlogindisplay manager of the Common Desktop Environment, to authenticate users logging into a UNIX system.

PAM can also be used by PAM-aware applications for authentication. These applications include the ObjectServer, the process agent, and gateways.

You can set up PAM authentication either by using external authentication sources, such as NIS and LDAP, or by using a single, central ObjectServer.
Interface Used :


NAME

    pam_sm_authenticate - service provider implementation for pam_authenticate

SYNOPSIS

    #include <security/pam_appl.h> #include <security/pam_modules.h> 

    int pam_sm_authenticate( pam_handle_t *pamh, int flags, int argc, const char **argv ); 

DESCRIPTION

    In response to a call to pam_authenticate(), the PAM framework calls pam_sm_authenticate() from the modules listed in the PAM configuration. The authentication provider supplies the back-end functionality for this interface function.

    The function, pam_sm_authenticate(), is called to verify the identity of the current user. The user is usually required to enter a password or similar authentication token depending upon the authentication scheme configured within the system. The user in question is typically specified by a prior call to pam_start(), and is referenced by the authentication handle, pamh.

    If the user is unknown to the authentication service, the service module should mask this error and continue to prompt the user for a password. It should then return the error, [PAM_USER_UNKNOWN].

    Before returning, pam_sm_authenticate() should call pam_get_item() and retrieve PAM_AUTHTOK. If it has not been set before (that is, the value is NULL), pam_sm_authenticate() should set it to the password entered by the user using pam_set_item().

    An authentication module may save the authentication status (success or reason for failure) as state in the authentication handle using pam_set_data(). This information is intended for use by pam_setcred().

Transactions

The lifecycle of a typical PAM transaction is described below. Note that if any of these steps fails, the server should report a suitable error message to the client and abort the transaction.

    If necessary, the server obtains arbitrator credentials through a mechanism independent of PAM—most commonly by virtue of having been started by root, or of being setuid root.

    The server calls pam_start(3) to initialize the PAM library and specify its service name and the target account, and register a suitable conversation function.

    The server obtains various information relating to the transaction (such as the applicant's user name and the name of the host the client runs on) and submits it to PAM using pam_set_item(3).

    The server calls pam_authenticate(3) to authenticate the applicant.

    The server calls pam_acct_mgmt(3) to verify that the requested account is available and valid. If the password is correct but has expired, pam_acct_mgmt(3) will return PAM_NEW_AUTHTOK_REQD instead of PAM_SUCCESS.

    If the previous step returned PAM_NEW_AUTHTOK_REQD, the server now calls pam_chauthtok(3) to force the client to change the authentication token for the requested account.

    Now that the applicant has been properly authenticated, the server calls pam_setcred(3) to establish the credentials of the requested account. It is able to do this because it acts on behalf of the arbitrator, and holds the arbitrator's credentials.

    Once the correct credentials have been established, the server calls pam_open_session(3) to set up the session.

    The server now performs whatever service the client requested—for instance, provide the applicant with a shell.

    Once the server is done serving the client, it calls pam_close_session(3) to tear down the session.

    Finally, the server calls pam_end(3) to notify the PAM library that it is done and that it can release whatever resources it has allocated in the course of the transaction.


IMPLEMENTATION

Let’s do an ls on /lib/security, this is where the pam modules reside in Ubuntu.

Let’s go ahead and create our custom module. First, be very careful, we’re messing with authentication and you risk locking yourself out. A good idea is to keep a couple of sessions open just in case.

Take a look at the code, you’ll see that PAM expect things to be laid out in a certain way. That’s fine, all we care about is where to write our custom code. In our case it starts at line 35. As you can see, the module takes 2 parameters, a URL and the size of the code to generate. The URL will be called and passed a code & username. It is this web service that will be in charge of dispatching the code to the user. This step could be done in the module itself but here we have in mind a centrally managed service in charge of dispatching codes to multiple users.

Deploying the code is done as follows:

gcc -fPIC -lcurl -c 2ndfactor.c

ld -lcurl -x --shared -o /lib/security/2ndfactor.so 2ndfactor.o


If you got errors, you probably need to first:

apt-get update

apt-get install build-essential libpam0g-dev libcurl4-openssl-dev


Do an ls on /lib/security again and you should see our new module, 

# PAM configuration for the Secure Shell service


# Read environment variables from /etc/environment and

# /etc/security/pam_env.conf.

auth       required     pam_env.so # [1]

# In Debian 4.0 (etch), locale-related environment variables were moved to

# /etc/default/locale, so read that as well.

auth       required     pam_env.so envfile=/etc/default/locale
For Applying 2 FA at the Ubuntu console :

Edit the common auth file at the given location sudo nano /etc/pam.d/common-auth and add out module call there.


For Applying 2 FA at the SSH console :

Now let’s edit /etc/pam.d/sshd, this is the file that describes which PAM modules take care of ssh authentication, account & session handling. But we only care about authentication here.

auth       required     2ndfactor.so base_url=http://tokenvalidatorurl.com/ code_size=5

auth requisite pam_deny.so
auth required pam_permit.so
auth optional pam_cap.so


# Standard Un*x authentication.

@include common-auth


The common-auth is probably what takes care of the regular password prompt so we’ll add our module call after this line as such:

auth       required     2ndfactor.so base_url=http://tokenvalidatorurl.com/ code_size=5

 

Or a complex as you can make it for a managed, multi-user, multi-server environment.

Lastly, edit /etc/ssd/sshd_config and change ChallengeResponseAuthentication to yes. Do a quick

/etc/init.d/ssh restart

 

for the change to take effect.

That’s it!
