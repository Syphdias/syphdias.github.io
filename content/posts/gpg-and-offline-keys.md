---
title: "GPG and Offline Keys"
date: 2025-01-14T23:21:30+01:00
draft: false
---

It is time again for me to renew my GPG keys and I wanted to write something
about GnuPG/GPG and YubiKey for a while now. I want to go over some things I
think someone should know, if they want to use GPG and a YubiKey for GPG. If I
think there are good resource for learning about certain aspects, I will link to
them.

## What this is not

- This does not explain basics of cryptography. I assume you know about
  asymmetric and symmetric encryption, and signing (basically [reverse
  asymmetric encryption of the contents hash]). It also is not a guide on how to
  use GPG on a normal day. You probably already know how to that. Take a look at
  signing your git commits and using it for SSH authentication!
- This does not recommend any hardware.
- This does not take a look at signing git commits with SSH (though that is a
  interesting topic imo).
- This does not go over installation of GPG or tooling around using a smartcard
  (e.g. `pcscd` and `pcsc_scan`, `ykman`, `kdf-setup`, etc.) — maybe later.
- This also does not go over thread modeling. You need to know if an
  intelligence service is after you, if data corruption is a risk, etc.

## Be Serious

If you are serious about using GPG, you should understand more than just how to
give your key to an application to use it to sign/(de)crypt for you. I would
recommend you never set the expiration of more than one year. Get comfortable
with generating and renewing GPG keys. _Never_ create a non-expiring key; this
is because in case you loose access to the secret key (e.g. forgetting the
password), the key will still be invalidated eventually.  
I would also recommend understanding more about the anatomy and though behind
the workings of GPG. I really liked Neal Walfield's [Advanced Intro to GnuPG].

## Playground

If you just run a GPG command like `gpg -K` you usually use your `~/.gnupg/` (on
Linux). To use another, temporary, and clean configuration directory you can set
`GNUPGHOME`. You can export it like in the following or provide the `--homedir`
argument for every call to GPG.

```sh
export GNUPGHOME=$(mktemp -d -t gnupg_$(date +%Y%m%d%H%M)_XXX)
```

## Actions of Keys

You might guess that there are at least two actions a key can be used for:
signing (a message) and encrypting a message.  
But GPG defines two more actions: certifying and authentication  
You can have separate keys for all four actions or you can combine sign, certify
(signing keys), and authenticate (signing for SSH authentication). Signing and
certifying are often combined, but I prefer to have a dedicated primary/certify
key, to be able to move it off my devices. This is called "offline"; more on
that later.

```sh
# generating primary key, valid for 1 year, this could also be 2006-01-02
# use a secure passphrase
gpg --quick-gen-key your@email.example ed25519 cert 1y
# generate sub key for signing and encryption (requires primary passphrase)
gpg --quick-add-key 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A ed25519 sign 1y
gpg --quick-add-key 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A cv25519 encr 1y
```

You can not now take a look a your key. The normal way or with more details:

```console
❯ gpg -K your@email.example
sec   ed25519 2025-01-05 [C] [expires: 2026-01-05]
      8CE453902E8E810DB46B8A5550ED5BF8E4042B5A
uid           [ultimate] your@email.example
ssb   ed25519 2025-01-05 [S] [expires: 2026-01-05]
ssb   cv25519 2025-01-05 [E] [expires: 2026-01-05]
```

There are a few things to talk about. The new key only has one `uid`, you can
however add more, for multiple email addresses. This also shows that you
trust this key ultimate-ly. Only trust your own keys to that level. But more to
that later.

The `sec` shows the private primary key, `ssb` the private subkey.

```console
❯ gpg -k your@email.example
pub   ed25519 2025-01-05 [C] [expires: 2026-01-05]
      8CE453902E8E810DB46B8A5550ED5BF8E4042B5A
uid           [ultimate] your@email.example
sub   ed25519 2025-01-05 [S] [expires: 2026-01-05]
sub   cv25519 2025-01-05 [E] [expires: 2026-01-05]
```

For this key and for imported public keys you can also see `pub` for the primary
key and `sub` for the public subkey.

```console
❯ gpg --keyid-format 0xlong -K --with-subkey-fingerprint --with-keygrip your@email.example
sec   ed25519/0x50ED5BF8E4042B5A 2025-01-05 [C] [expires: 2026-01-05]
      8CE453902E8E810DB46B8A5550ED5BF8E4042B5A
      Keygrip = 2BFDFFC57B771DCE7F009C3BEF37142EBCF4B8E5
uid                   [ultimate] your@email.example
ssb   ed25519/0xB654F428066CB8A3 2025-01-05 [S] [expires: 2026-01-05]
      149A02287C20580A7CF01CAAB654F428066CB8A3
      Keygrip = 82887047B2E87C91CC4EA9915D22A6C4D5B23006
ssb   cv25519/0x587AE7468688C837 2025-01-05 [E] [expires: 2026-01-05]
      3782DF5CA1038377F172D881587AE7468688C837
      Keygrip = A57B578C56165F6EFFBC1249DDE4E69F0553D433
```

When using your GPG key you usually reference the primary key and GPG selects
the most recent subkey to do the actual work. If you have only one sub key of a
kind, there cannot be any confusion, but if you have multiple subkeys you can
also reference a specific subkey with `SUBKEYFINGERPRINT!`.  
Also note that the `0xlong` format uses the ending of the full keyid, not the
start. The `long` version is often what you see when referring to keys.  
The keygrip is what you will see on the file system. In
`${GNUPGHOME:-$HOME/.gnupg}/private-keys-v1.d/` you can find `${keygrip}.key`.

## List Packets

GPG was designed to process messages in one pass and not require loading the
entire messages into memory. It works on streams of packages in a specific
order. This means, if you want decrypt something, the information about which
key needs to be used comes first.

### Message

Looking at a simple message (a new line), encrypting it to our newly created
key, and then inspecting its packets.

```console
❯ echo |gpg -aer 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A |gpg --list-packets
gpg: encrypted with cv25519 key, ID 587AE7468688C837, created 2025-01-05
      "your@email.example"
# off=0 ctb=84 tag=1 hlen=2 plen=94
:pubkey enc packet: version 3, algo 18, keyid 587AE7468688C837
        data: [263 bits]
        data: [392 bits]
# off=96 ctb=d4 tag=20 hlen=2 plen=70 new-ctb
:aead encrypted packet: cipher=9 aead=2 cb=16
        length: 70
# off=117 ctb=a3 tag=8 hlen=1 plen=0 indeterminate
:compressed packet: algo=2
# off=119 ctb=cb tag=11 hlen=2 plen=7 new-ctb
:literal data packet:
        mode b (62), created 1736115332, name="",
        raw data: 1 bytes
```

The first two lines are printed to stderr and show information about the
encrypted message. Then there follow four packets.  

- The first one is information about the public key used — note that the keyid
  is not the primary key but the id of the subkey for encryption.  
- The second one is a asymmetrically encrypted packet containing the symmetric
  session key to decrypt the message. If you "asymmetrically" encrypt a message
  to multiple recipients, all it does is encrypt the symmetric key multipel
  times. This way the message does not have to be duplicated.  
  There is debate about if you should use AEAD due to a split between GnuPG and
  the OpenPGP specification it is based on. TL;DR: [disable it for now].
  - Side note: Should you ever be forced to to decrypt data encrypted to you,
    you should [never give up] your entire secret key, but only decrypt relevant
    session keys. This way your key is not compromised.
- The third packets contains the fourth packet, as far as I know. I am not an
  expert on GPG but as far as I understood it, there is a hierarchy to packets,
  but I could not find an easy way to show that.
- The fourth packet is the actual encrypted data that can be decrypted with the
  decrypted symmetric key from earlier.

Note: Also try the command from above with `-vv`.

Now try the same with `echo | gpg -s | gpg --list-packets`. You will three
packets: onepass_sig, literal data, and signature.  
The onepass_sig packet is useful for checking the signature in one pass. This
way the hash for for the signature can be calculated while the message is
already in memory.

### Key

You can do the same on public and secret keys.

```console
❯ gpg --export your@email.example -a |gpg --list-packets
# off=0 ctb=98 tag=6 hlen=2 plen=51
:public key packet:
        version 4, algo 22, created 1736114781, expires 0
        pkey[0]: [80 bits] ed25519 (1.3.6.1.4.1.11591.15.1)
        pkey[1]: [263 bits]
        keyid: 50ED5BF8E4042B5A
# off=53 ctb=b4 tag=13 hlen=2 plen=18
:user ID packet: "your@email.example"
# off=73 ctb=88 tag=2 hlen=2 plen=153
:signature packet: algo 22, keyid 50ED5BF8E4042B5A
        version 4, created 1736114781, md5len 0, sigclass 0x13
        digest algo 10, begin of digest 13 2e
        hashed subpkt 33 len 21 (issuer fpr v4 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A)
        hashed subpkt 2 len 4 (sig created 2025-01-05)
        hashed subpkt 27 len 1 (key flags: 01)
        hashed subpkt 9 len 4 (key expires after 1y0d0h0m)
        hashed subpkt 11 len 4 (pref-sym-algos: 9 8 7 2)
        hashed subpkt 34 len 1 (pref-aead-algos: 2)
        hashed subpkt 21 len 5 (pref-hash-algos: 10 9 8 11 2)
        hashed subpkt 22 len 3 (pref-zip-algos: 2 3 1)
        hashed subpkt 30 len 1 (features: 07)
        hashed subpkt 23 len 1 (keyserver preferences: 80)
        subpkt 16 len 8 (issuer key ID 50ED5BF8E4042B5A)
        data: [254 bits]
        data: [256 bits]
# off=228 ctb=b8 tag=14 hlen=2 plen=51
:public sub key packet:
        version 4, algo 22, created 1736114906, expires 0
        pkey[0]: [80 bits] ed25519 (1.3.6.1.4.1.11591.15.1)
        pkey[1]: [263 bits]
        keyid: B654F428066CB8A3
# off=281 ctb=88 tag=2 hlen=2 plen=245
:signature packet: algo 22, keyid 50ED5BF8E4042B5A
        version 4, created 1736114906, md5len 0, sigclass 0x18
        digest algo 10, begin of digest 50 5d
        hashed subpkt 33 len 21 (issuer fpr v4 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A)
        hashed subpkt 2 len 4 (sig created 2025-01-05)
        hashed subpkt 27 len 1 (key flags: 02)
        hashed subpkt 9 len 4 (key expires after 1y0d0h0m)
        subpkt 16 len 8 (issuer key ID 50ED5BF8E4042B5A)
        subpkt 32 len 117 (signature: v4, class 0x19, algo 22, digest algo 10)
        data: [256 bits]
        data: [255 bits]
# off=528 ctb=b8 tag=14 hlen=2 plen=56
:public sub key packet:
        version 4, algo 18, created 1736114910, expires 0
        pkey[0]: [88 bits] cv25519 (1.3.6.1.4.1.3029.1.5.1)
        pkey[1]: [263 bits]
        pkey[2]: [32 bits]
        keyid: 587AE7468688C837
# off=586 ctb=88 tag=2 hlen=2 plen=126
:signature packet: algo 22, keyid 50ED5BF8E4042B5A
        version 4, created 1736114910, md5len 0, sigclass 0x18
        digest algo 10, begin of digest 73 9c
        hashed subpkt 33 len 21 (issuer fpr v4 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A)
        hashed subpkt 2 len 4 (sig created 2025-01-05)
        hashed subpkt 27 len 1 (key flags: 0C)
        hashed subpkt 9 len 4 (key expires after 1y0d0h0m)
        subpkt 16 len 8 (issuer key ID 50ED5BF8E4042B5A)
        data: [256 bits]
        data: [256 bits]
```

- The first packet is the public or secret key (depending on what you looked
  at). You might spot `expires 0`. This means no expiration but will be
  overwritten later.
- The second packet is the user id.
- The third packet is probably the most interesting packet as it is the
  signature of the two earlier packets and contains the settings for the key,
  like preferred algorithms and the expiration. This is the reason why you can
  easily change the expiration on a key without invalidating the key or
  invalidating signatures of the key and uid: The key stays the same, only the
  signature changes. Every uid get its own signature, so it can be easily
  removed.
- Then the two subkeys follow, first the public/secret key itself,…
- …then the signature from the primary key that also tells others that these are
  subkeys of the primary key and other settings. This means, that to change
  expiration of the subkeys, you need the primary key.
  These keys are not special to the primary key. In fact, it is possible to
  reuse the same same subkeys and recertify them to another primary key (should
  you have lost the old one). There is also other black magic you can do.
  Signing keys also [cross-sign] the primary key to prevent someone pretending
  that the subkey is theirs.

## Offline Keys and a YubiKey

Generally offline key only means, that it is currently not on the device. It
could be on another computer, on a flash drive, or a smart card like a YubiKey.
It is generally recommended to use an offline primary key, but any key could be
offline.  
Having an offline primary key comes with some caveats. While you can use all
other keys normally, you cannot create, revoke, sign other keys, or change
subkeys or uids. This includes settings, e.g. expiration.

It is important to understand that a smartcard or a YubiKey with smartcard
functionality a key only goes one way. You can store a key on a smartcard/YK but
you can never retrieve it (in lieu of exploits). This is very much intentional,
as the idea of smartcards is to do the secure computing on the card and never
load any secrets into comptuer memory. This is by design how it is supposed to
work.  
This is why you need to watch out for the command `addtocard` which saves the
key to the smartcard and deletes the key from disk. Meaning you can never backup
that key again, if you do not have a backup already. The same applies if you
generate the key on the smartcard — your computer will never see the contents of
that key.

Another limitation of my YubiKey is, that it only has three key slots. One for
a signature key, one for an encryption key, one for an authentication key. So if
you use separate keys for certify and sign, you are SOL because you can only put
one into the signing slot. In my case I put the primary key into the signature
slot as an additional backup and to be able to manage subkeys more easily
without needing to go to the true offline primary key.

If you see `sec#`, the primary key is not on the device. If you see `sec>` the
primary key is on a smartcard (if you look at the keygrip file you will find a
stub instead of the actual key).

### Backup and Removal of Key

There is something more important than backing up your encrypted email — backing
up your secret keys. The easiest way is to just backup the entire `GNUPGHOME`,
usually `~/.gnupg/`. In my case I have LUKS encrypted flash drive I copy it to.
All usual backup practices apply — 3-2-1, in case your house gets struck by
lightning while the flash drive is inserted into your computer.

```sh
DATE=$(date +%F)
mkdir /run/media/syphdias/LUKS001/gpg/.gnupg-$DATE
rsync -aP ~/.gnupg/ /run/media/syphdias/LUKS001/gpg/.gnupg-$DATE
```

I use the date, in case I mess something up, during renewal. Now we remove the
primary key from our keyring on the device. We already found the keygrip when
looking at our keys.

```console
❯ rm ~/.gnupg/private-keys-v1.d/2BFDFFC57B771DCE7F009C3BEF37142EBCF4B8E5.key
❯ GNUPGHOME=gnupg_202501051540_gqK/ gpg -K
/home/syphdias/.gnupg/pubring.kbx
--------------------------------------------------------------------------------
sec#  ed25519 2025-01-05 [C] [expires: 2026-01-05]
      8CE453902E8E810DB46B8A5550ED5BF8E4042B5A
uid           [ultimate] your@email.example
ssb   ed25519 2025-01-05 [S] [expires: 2026-01-05]
ssb   cv25519 2025-01-05 [E] [expires: 2026-01-05]
```

Seeing `sec#` shows the successful removal of the primary key. Keep in mind the
caveats from above.

### Renewing with Offline Key

A year has passed and you are still into GPG for some nerdish reason and want to
keep working with out keys without regenerating completely new ones.

```console
❯ cd /run/media/syphdias/LUKS001/gpg/
❯ cp -r .gnupg-2024-01-10/ .gnupg-2025-01-14/
❯ # only work on a copy of your backup for safety
❯ gpg --quick-set-expire
usage: gpg [options] --quick-set-exipre FINGERPRINT EXPIRE [SUBKEY-FPRS]
❯ # renew primary key
❯ GNUPGHOME=.gnupg-2025-01-14 gpg --quick-set-expire 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A 2027-01-05
❯ # renew subkeys
❯ GNUPGHOME=.gnupg-2025-01-14 gpg --quick-set-expire 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A 2027-01-05 149A02287C20580A7CF01CAAB654F428066CB8A3 3782DF5CA1038377F172D881587AE7468688C837
```

You can use `GNUPGHOME=.gnupg-2025-01-14 gpg -K` this worked. If you wanted you
could also change other attributes at this time, like preferred algorithms, or
add another uid. If we are happy with the keyring, you can keep the directory as
backup to copy for next year.

There are a few ways how you could get the renewed offline keys to your
machine(s) now. You could copy the subkeys by keygrip — dont! Or copy over the
entire keyring and remove the primary key again — meh! The proper way is to
export the secret subkeys only and import them into your current keyring.

```console
❯ # check if you will import what you wanted
❯ GNUPGHOME=.gnupg-2025-01-14 gpg --export-secret-subkeys --export-options export-minimal 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A |gpg --import --import-options show-only
❯ GNUPGHOME=.gnupg-2025-01-14 gpg --export-secret-subkeys --export-options export-minimal 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A |gpg --import
❯ # if you published your GPG key, update the published key
❯ gpg --send-keys 8CE453902E8E810DB46B8A5550ED5BF8E4042B5A
```

You could only export certain subkeys, if you wanted. Or, if you have multiple
uids, you could also filter your export with `--export-filter
keep-ui=mbox=your@email.example`. To check your export you can check the import
without importing by using the command `gpg --import --import-opstions
show-only`. This can also be useful for others' public keys.

Every time you change settings of your GPG key, you create a new signature. With
`--export-optitons export-minimal` removes all signatures except the most recent
one.

### Moving Key to YubiKey

If you want to have your primary key not just in your backup, you can edit your
key (`gpg --edit-key KEY`) and select `keytocard`. Leave with `save`. From then
on you can use the key on the YubiKey. Keep in mind that there are only three
key slots. I mainly have this a additional backup of my primary key and to be
able to use the primary key to manage subkeys that I might not need in my backup
and to set settings on-the-fly without requiring my true offline primary key.

## Publishing to a Keyserver

Keyservers were very open in the past. You just uploaded your key to them and
there were forever findable and others could upload their signatures for other
keys, etc. This is kinda dead. For one you could DOS someone with overwhelming
their key with signature. This was basically the last nail in the coffin for the
Web of Trust.  Another reason the traditional keyservers are no longer around is
the GDPR which requires the option to remove personal identifiable data on
request. Today you can either publish your GPG in a place you think proper for
people to manually find it, use [Web Key Directory] (WKD) to store keys on your
website at `/.well-known/openpgpkey/hu/hash-of-uid`, or [upload your key to
https://keys.openpgp.org] where you will also need to verify your email
address.

## Misc

In the past GPG wanted key owners verifying each other's keys and establish a
"Web of Trust" in key signing parties. This did not take hold and is basically
nerd sports for enthusiasts that do it for the fun of it. The practical
application is basically dead. Trust on first use (TOFU) is easier in practice
with less overhead. You can combine the two modes by setting `trust-model
tofu+pgp` in your `${GNUPGHOME:-$HOME/.gnupg}/gpg.conf`.

YubiKeys has more uses than acting as a smartcard for GPG.

```console
❯ ykman --device 12345678 info
WARNING: PC/SC not available. Smart card (CCID) protocols will not function.
ERROR: Unable to list devices for connection
Device type: YubiKey 5C NFC
Serial number: 12345678
Firmware version: 5.4.3
Form factor: Keychain (USB-C)
Enabled USB interfaces: OTP, FIDO, CCID
NFC transport is enabled

Applications    USB     NFC
Yubico OTP      Enabled Enabled
FIDO U2F        Enabled Enabled
FIDO2           Enabled Enabled
OATH            Enabled Enabled
PIV             Enabled Enabled
OpenPGP         Enabled Enabled
YubiHSM Auth    Enabled Enabled
```

## Settings

There are a few settings I think you should take a look at in the
`~/.gnupg/gpg.conf` file.

```conf
# If you do not set this option the first secret key found will be used
default-key YOURKEYID
# If you encrypt someone, you will never again be able to decrypt it. This can
# be a feature to be anonymous, but can be a pain, if you use this to send
# emails and still want to be able to read your sent emails.
default-recipient-self
# disable comments in clear text signatures and ASCII armored messages
no-comments
# default to long format
keyid-format 0xlong
# try GPG agent before asking for passphrase
use-agent
# use TOFU and WOT/PGP
trust-model tofu+pgp
# used for --(recv|send|search)-keys
keyserver hkps://keys.openpgp.org:443/
```

I will not include any cipher preferences as they will go out of date.

## Best-ish Practices

…as far as I can tell.

- use secure passphrases
- reference key by long format (opposed to short id or email)
- again, _always_ set expiration for all keys
- create revoke file for your primary key `gpg --gen-revoke KEY`
- [do not use comment field](http://web.archive.org/web/20201020082313/https://debian-administration.org/users/dkg/weblog/97)
- maybe use name field — peoples opinions vary

## Further Reading

There is probably more that I want to write but at some point it just has to be
another day, another post.

I can recommend drduh's [YubiKey-Guide]. If you want to use a YubiKey at least
give it a skim. And if you got your YubiKey, read it whole.

[reverse asymmetric encryption of the contents hash]: https://blog.koehntopp.info/2018/03/04/hashes-in-structures.html
[Advanced Intro to GnuPG]: https://begriffs.com/posts/2016-11-05-advanced-intro-gnupg.html
[disable it for now]: https://security.stackexchange.com/questions/275883/should-one-really-disable-aead-for-recent-gnupg-created-pgp-keys
[never give up]: https://www.youtube.com/watch?v=dQw4w9WgXcQ
[cross-sign]: https://www.gnupg.org/faq/subkey-cross-certify.html
[Web Key Directory]: https://www.gnupg.org/blog/20160830-web-key-service.html
[upload your key to https://keys.openpgp.org]: https://keys.openpgp.org/about/usage
[YubiKey-Guide]: https://github.com/drduh/YubiKey-Guide
