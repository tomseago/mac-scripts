There are 3 keychains on 10.7 (Lion), "Login", "System", and "System Roots".

By Catalina at 10.15 there is also a separate "iCloud" keychain visible in Keychain Access. My main / old iMac that has been upgraded over the years also has a couple other keychains that have been added for various reasons.

It's not clear to me that the "System Roots" can actually be modified. However, it should be possible to put certs into the system keychain and have them be sufficiently trusted. Let's keep digging though...

The mac system keychain is stored in 

    /Library/Keychains/System.keychain

while the login keychain is in some variation of `~/Library/Keychains`

While Keychain Access is the GUI program, `security` is the roughly equivalent and hopefully more powerful command line program. There are also `certtool` and `systemkeychain` commands, although these appear to be older. The entire functionality may not be present in `security` but I'm not sure what is missing.

From the `certtool` man page we learn that

    /System/Library/Keychains/X509Anchors

is the root keychain file. However, this directory also holds `SystemRootCertificates.keychain` and `SystemTrustSettings.plist` though so maybe those are what we _actually_ want to be looking at.

This holds true for both the iMac and the laptop that upgraded to Catalina, although the laptop is significantly newer. Basically, it's not clear to me what the difference is between X509Anchors vs SystemRootCertificates.keychain - time for a google.

Apparently on 10.4 and earlier it was X509Anchors but by 10.5 we moved to the SystemRootCertificates.keychain location. Thus, that's going to be the one we focus on.

There is a minor issue in the man page that the dump-keychain command does not list the `[keychain]` parameter, but it's there.

To see all our system roots we can

    security dump-keychain /System/Library/Keychains/SystemRootCertificates.keychain

To export all system root we would do

    security export -k /System/Library/Keychains/SystemRootCertificates.keychain -o SystemRoots.pem

To import a file to the system root it seems we will

    security import SystemRoots.pem -k /System/Library/Keychains/SystemRootCertificates.keychain

The downside being - if the item already exists, then it seems to stop the entire import process. Thus, one could create a new `SystemRoots.keychain` file, import things into that, and then slip it into place and maybe reboot or something like that? Let's try...

First gotta create a keychain though (as root, the leading / is important for the keychain name or else it ends up in `~/Library`)

    security create-keychain /tmp/NewSystemRoots.keychain

when prompted for the password, none. No password.

Then import the things into it

    security import SystemRoots.pem -k /tmp/NewSystemRoots.keychain

Move the old one out of the way and the new one into position

    mv /System/Library/Keychains/SystemRootCertificates.keychain /System/Library/Keychains/SystemRootCertificates-old.keychain
    mv /tmp/NewSystemRoots.keychain /System/Library/Keychains/SystemRootCertificates.keychain

If keychain access was open, it must be restarted to get the new system roots.

Now the task is to make all the roots actually be trusted. **This should be scripted.** However, for fun, it can also be done manually now because the certs are there, they will just be untrusted in the UI.

To export the system trust settings we can

    security trust-settings-export -s SystemTrustSettings.plist

And then as root import them as at least "system"

    security trust-settings-import -d SystemTrustSettings.plist

That will end us up not entirely exactly where we want, because it's really the defaults that we wanted to have be trusted, but it works.




