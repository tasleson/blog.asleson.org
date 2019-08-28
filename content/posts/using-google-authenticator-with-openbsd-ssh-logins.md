---
title: 'Using google authenticator with OpenBSD SSH logins'
date: Fri, 11 Apr 2014 19:54:38 +0000
draft: false
tags: [OpenBSD, TFA]

aliases:
   - /index.php/2014/04/11/using-google-authenticator-with-openbsd-ssh-logins/
---

For an introduction on two factor authentication please see: 
[http://en.wikipedia.org/wiki/Two-step\_verification](http://en.wikipedia.org/wiki/Two-step_verification "http://en.wikipedia.org/wiki/Two-step_verification") 

**NOTE: Make sure you leave a terminal with root access if you are using a 
remote system until you have tested that you can indeed authenticate to it!** 

Steps tested on OpenBSD 5.4, used tools on EL6 client to generate QR code. 
I’m mainly documenting this here so I can remember how to do this again. 

1. Install: login\_oauth-.tgz eg.

    ```
    # pkg_add -v http://mirror.planetunix.net/pub/OpenBSD/5.4/packages/i386/login_oath-0.8p1.tgz
    ```
    
    Make sure to look over the readme: /usr/local/share/doc/pkg-readmes/login_oath-version 

2. Add a new login class by editing /etc/login.conf and add change user(s) to use it.

    ```
    # The TOTPPW login class requires both TOTP and passwd; the user must
    # supply the password in the format OTP/password, e.g. 816721/axlotl.
    
    totppw:\\
            :auth=-totp-and-pwd:\\
            :tc=default:
    
    ```
    Change user(s) login class eg.
    
    ```
    # usermod -L totppw username
    ```

3. Generate a random key in the users home directory

    ```
    $ openssl rand -hex 20 > ~/.totp-key
    ```

    Note: Make sure your home directory and this file are not world readable! 
    Otherwise you will be prevented from logging in. 

4. Convert hex string key to base32 documents shows perl, I have supplied some 
code in python.

    ```
    import binascii
    import base64
    import sys
    
    if __name__ == '__main__':
        if len(sys.argv) != 2:
            print 'Syntax: %s ' % ( sys.argv[0])
            sys.exit(1)
    
        print base64.b32encode(binascii.unhexlify(sys.argv[1]))
    
    ```

5. Generate QR code so you don’t have to type in a big random sequence on your 
smart phone. Free web based ones exist, but only use if you trust.

    ```
    $ qrencode -o ~/.totp-key.png 
    "otpauth://totp/?secret=BASE 32 SECRET&issuer=Your name, etc." 
    ```

    More info: http://code.google.com/p/google-authenticator/wiki/KeyUriFormat 

6. Using google authenticator create a new software key using your created 
QR code. If you don’t want to create a QR code, then create a new entry, 
set to time based and enter the 64 characters. I’m guessing it may take a 
few tries to get it correct. 

7. Test that google authenticator and 
    ```
    $ oathtool –totp `cat ~/.totp-key`
    ```
    match. If they don’t make sure both the phone and the computer dates and 
    time match. 

8. Using a separate terminal, ssh to remote system with just the password, 
this should fail. Then try using OTP/password for your password and you 
should get authenticated.