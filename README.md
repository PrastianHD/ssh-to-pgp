# ssh-to-pgp

Requirements: [OpenPGP-Python](https://github.com/singpolyma/OpenPGP-Python), [pycrypto](https://pypi.python.org/pypi/pycrypto)

This tool converts an SSH key in to an OpenPGP compatible authentication key. This can be helpful if you use the `gpg-agent` as your SSH agent, or if you want to migrate an existing SSH private key in to an OpenPGP compatible smartcard.

One of the main benefits of using an OpenPGP compatible smartcard for SSH authentication is that the key cannot be cloned or stolen once it's stored in the smartcard. You should ask yourself whether migrating a crusty old SSH key that's been copied to every laptop you've owned in the last five years is a smarter choice than generating a new, secure key directly on the card. I'll leave that decision up to you.

This process has been tested with the [Yubikey 4](https://www.yubico.com/products/yubikey-hardware/yubikey4/) from Yubico. Once the key has been copied to the card (but as of January 2017, ideally before the card PIN has been set) it is further possible to enable touch-to-authenticate with [yubitouch.sh](https://github.com/a-dma/yubitouch/blob/master/yubitouch.sh)

## Example key conversion

First a new key is created for use with the demo.

    fincham@laptop:~/Documents/Development/ssh-to-pgp$ ssh-keygen -b 2048
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/fincham/.ssh/id_rsa): /home/fincham/.ssh/demo_id_rsa
    Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Your identification has been saved in /home/fincham/.ssh/demo_id_rsa.
    Your public key has been saved in /home/fincham/.ssh/demo_id_rsa.pub.
    The key fingerprint is:
    46:76:10:9e:7b:b6:50:28:1a:5a:0c:e5:70:d6:e9:23 fincham@laptop

Next `ssh-to-pgp` is used to create a new OpenPGP message containing the same RSA key, and this is imported directly in to GnuPG. It is important to note that the key will currently be imported without a passphrase (and therefore will not be encrypted when it is stored on disk).

    fincham@laptop:~/Documents/Development/ssh-to-pgp$ python ./ssh-to-pgp --ssh-key-path=/home/fincham/.ssh/demo_id_rsa import
    Reading SSH private key from /home/fincham/.ssh/demo_id_rsa...
    writing RSA key

    Generated PGP key overview
    --------------------------
    :secret key packet:
        version 4, algo 1, created 1486713682, expires 0
        skey[0]: [2048 bits]
        skey[1]: [17 bits]
        skey[2]: [2048 bits]
        skey[3]: [1024 bits]
        skey[4]: [1024 bits]
        skey[5]: [1022 bits]
        checksum: 38eb
        keyid: B51FF6CA0D8822BB
    :user ID packet: "Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)"
    :signature packet: algo 1, keyid B51FF6CA0D8822BB
        version 4, created 1486713682, md5len 0, sigclass 0x10
        digest algo 8, begin of digest a4 c4
        hashed subpkt 2 len 4 (sig created 2017-02-10)
        hashed subpkt 27 len 1 (key flags: 20)
        hashed subpkt 16 len 8 (issuer key ID B51FF6CA0D8822BB)
        data: [2048 bits]

    Import this key in to GnuPG without a passphrase? [y/N] y
    gpg: key 0xB51FF6CA0D8822BB: secret key imported
    gpg: key 0xB51FF6CA0D8822BB: public key "Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)" imported
    gpg: Total number processed: 1
    gpg:               imported: 1  (RSA: 1)
    gpg:       secret keys read: 1
    gpg:   secret keys imported: 1
    fincham@laptop:~/Documents/Development/ssh-to-pgp$ gpg2 --list-secret-keys 0xB51FF6CA0D8822BB
    sec   2048R/0xB51FF6CA0D8822BB 2017-02-10
        Key fingerprint = 9B5D 73C6 07F1 0854 3B9B  D47F B51F F6CA 0D88 22BB
    uid                            Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)

At this point it is possible to use the `keytocard` functionality of GnuPG to copy the newly made PGP key in to an Authentication key slot of a compatible smartcard.

    fincham@laptop:~/Documents/Development/ssh-to-pgp$ gpg2 --edit-key 0xB51FF6CA0D8822BB
    gpg (GnuPG) 2.0.26; Copyright (C) 2013 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  2048R/0xB51FF6CA0D8822BB  created: 2017-02-10  expires: never       usage: CA  
                                trust: unknown       validity: unknown
    [ unknown] (1). Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)

    gpg> toggle

    sec  2048R/0xB51FF6CA0D8822BB  created: 2017-02-10  expires: never     
    (1)  Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)

    gpg> keytocard
    Really move the primary key? (y/N) y
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]

    Please select where to store the key:
    (1) Signature key
    (3) Authentication key
    Your selection? 3

    sec  2048R/0xB51FF6CA0D8822BB  created: 2017-02-10  expires: never     
                        card-no: 0003 06041672
    (1)  Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)

    gpg> save

When this operation is complete the card status should show the migrated subkey, and if `gpg-agent` is configured as an SSH agent, `ssh-add` will show the same SSH key fingerprint as the original key on disk.

    fincham@laptop:~/Documents/Development/ssh-to-pgp$ gpg2 --card-status
    Application ID ...: D5163134375775820003060416720000
    Version ..........: 2.1
    Manufacturer .....: Example
    Serial number ....: 06041672
    Name of cardholder: [not set]
    Language prefs ...: [not set]
    Sex ..............: unspecified
    URL of public key : [not set]
    Login data .......: [not set]
    Signature PIN ....: not forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 0 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: 9B5D 73C6 07F1 0854 3B9B  D47F B51F F6CA 0D88 22BB
        created ....: 2017-02-10 08:01:22
    General key info..: pub  2048R/0xB51FF6CA0D8822BB 2017-02-10 Imported from SSH private key (/home/fincham/.ssh/demo_id_rsa)
    sec>  2048R/0xB51FF6CA0D8822BB  created: 2017-02-10  expires: never     
                        card-no: 0003 06041672
    fincham@laptop:~/Documents/Development/ssh-to-pgp$ ssh-add -l
    2048 46:76:10:9e:7b:b6:50:28:1a:5a:0c:e5:70:d6:e9:23 cardno:000306041672 (RSA)

Once the card is confirmed working the private part of the generated key can be removed from GnuPG's secret keyring (though the public part must remain).

## Limitations

The decrypted private key will be kept in memory by the `ssh-to-pgp` script. Python currently provides no effective way to ensure the safe handling of this key material, nor will it be encrypted when it is saved to disk by GnuPG during import. For this reason it may be preferable to perform this process while booted from a live CD, then rebooting afterwards. Remember to export and retain the public part of the key even after it has been migrated, as this is needed for SSH authentication to take place (e.g. `gpg2 --export-key ...`).

The `ssh-to-pgp` script has very little in the way of error checking. Any problems encountered during the process will likely result in cryptic error messages.

Only RSA SSH keys may currently be converted by this tool, and some smart cards may place a limitation on the bit size of stored keys.
