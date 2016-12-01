# Git Crypt Instructions with GPG

Git-crypt is a mechanism for handling encrypted repositories using GPG keys. 

## Pre-requisites

    brew install git-crypt
    brew install gnupg2
    gpg --gen-key


## Generate and Save your GPG public key

Put in your GPG key or identity into the access/credentials document that your sysadmin uses. Either paste in your public key into the relevant column, OR put in your key identity and the keyserver in the format: `keyserver.ubuntu.com: username@domain.com`

## Getting Public Key from Keyserver

The identity is often an email address, and that's a good thing since that's a (presumably) uncorrupted and publicly validated database, rather than this spreadsheet which can easily be tampered with! So how to convert the GPG identity to a GPG public key with which we can sign?
* My identity is cybertoast@gmail.com. The way to get it into a usable form is:
* Go to the keyserver, such as http://keyserver.ubuntu.com:11371/
* Search for cybertoast@gmail.com
* Get my keys. I have two. For the sake of this email let's just use the first one:

    pub  2048R/7236DE85 2010-03-05            
    Fingerprint=ABF1 4424 B040 537C C228  492B BC07 477B 7236 DE85
    uid Sundar Raman <cybertoast@gmail.com>
    sig  sig3  7236DE85 2010-03-05 __________ __________ [selfsig]
    sub  2048R/79475632 2010-03-05            
    sig sbind  7236DE85 2010-03-05 __________ __________ []
    Fetch the relevant key into your keychain, using the ID of the key (the bolded text above):
    $ gpg --keyserver keyserver.ubuntu.com --recv-key 0x47C103C5
    gpg: requesting key 47C103C5 from hkp server keyserver.ubuntu.com
    gpg: key 47C103C5: public key "Caleb Flynn <calebflynn@gmail.com>" imported
    gpg: Total number processed: 1
    gpg:               imported: 1  (RSA: 1)
    
    Importing an explicit public key
    If the key is just plain text, then;
    Save the text to a local file from the relevant user record
    Import the key:
    $ gpg --import ~/Desktop/cj.gpg.pub 
    gpg: key 95C8F9C3: "Charles DiMaggio <charlesdimaggio@localprojects.com>" not changed
    gpg: Total number processed: 1
    gpg:              unchanged: 1
    
    The key can also be imported with:
    gpg --keyserver keyserver.ubuntu.com --recv [Identifier]

## Setting up an encrypted repo with select files

git-crypt uses the .gitattributes file to indicate which files to encrypt or not. But let's start with the basics.

### Starting with a brand new repo

In order to simplify your life, the `git@github.com:local-projects/git-crypt-starter.git` repo provides a quick start. This provides a sensible .gitignore and .gitattributes file from which you can start for your project:

    # Start with the starter repo, but name it whatever you want
    git clone git@github.com:local-projects/git-crypt-starter.git my-secret-repo
    
    cd my-secret-repo
    
    # Remove the remote, so that you can put in your own remote
    git remote remove origin
    
    # Add in your own origin
    git remote add origin git@github.com:my-secret-repo
    
    # If you decide NOT to use the starter-repo you'll have to init your repo
    # git init
    # .. and init git-crypt
    # git crypt init

You should now have a starting point for your encryption needs. Note what's in the .gitattributes file:

    # Encrypt everything else
    * filter=git-crypt diff=git-crypt

    # Things to ignore, which MUST be added AFTER what's encrypted,
    # since this is the excclusion list!
    .git* !filter !diff
    readme* !filter !diff
    README* !filter !diff

This tells the system not to encrypt readme* or git* files and folders, but encrypt everything else. Start with a file that you want to encrypt by just creating a file:

    echo "something secret to encrypt" > secrets.sample

Check what git-crypt sees:

    $ git crypt status
    not encrypted: .gitattributes
    not encrypted: .gitignore
    not encrypted: README.md
        encrypted: secrets.sample

Push this repo to the origin, but make sure you're NOT pushing it to the local-projects repo since that would just make all your secrets publicly visible!!!!

### Starting from an existing repo

For the most part an encrypted repo will probably already exist and be provided to you, in which case you just need to decrypt the relevant content. 

    $ cd /tmp/
    $ git clone git@lpgitserver:pepperdine.creds
    $ cd pepperdine.creds
    $ git crypt status
    $ git crypt unlock

    You need a passphrase to unlock the secret key for
    user: "Sundar Raman <cybertoast@gmail.com>"
    2048-bit RSA key, ID 79475632, created 2010-03-05 (main key ID 7236DE85)

    Enter passphrase: 
    $ more secret-file 

    ## Exposed secrets
    ...

### Encrypting and Adding users

Adding new files and encrypting is taken care of for you by virtue of the .gitattributes configuration. But you'll need to add users who have access. You do this by adding the relevant gpg identities:

    $ git crypt add-gpg-user 7236DE85
    [master 41e2c90] Add 1 git-crypt collaborator
     2 files changed, 6 insertions(+)
     create mode 100644 .git-crypt/.gitattributes
     create mode 100644 .git-crypt/keys/default/0/ABF14424B040537CC228492BBC07477B7236DE85.gpg

One potentially confusing thing thing is that when you add users, you won't notice anything having changed in the repo:

    $ git status
    On branch master
    nothing to commit, working directory clean

But you still need to push for the users to be in effect:

    $ git push origin master
    Counting objects: 8, done.
    Delta compression using up to 8 threads.
    Compressing objects: 100% (6/6), done.
    Writing objects: 100% (8/8), 1.12 KiB | 0 bytes/s, done.
    Total 8 (delta 1), reused 0 (delta 0)
    To git@github.com:local-projects/git-crypt-starter.git
       d24b37a..41e2c90  master -> master

# Validating that your encryption worked

How can you test that all the encryption steps you performed actually work? By cloning to a temp folder and looking at what you get:

    cd /tmp
    git clone git@git.server:my-test-secret-repo
    cd my-test-secret-repo
    $ more secrets.sample 
    "secrets.sample" may be a binary file.  See it anyway? 

This generally would mean that the file is encrypted if it was originally a text file. Obviously you may have encrypted a binary (image?) file, in which case your validation will have to account for this. So decrypt it and ensure you've got what you expect:

    $ git crypt unlock

    You need a passphrase to unlock the secret key for
    user: "Sundar Raman <cybertoast@gmail.com>"
    2048-bit RSA key, ID 79475632, created 2010-03-05 (main key ID 7236DE85)
    
    $ more secrets.sample 
    Some secret that should be encrypted, like for realz.

Pretty straight-foward :)

# Troubleshooting

## Files that got missed from encryption

You may be in a situation, especially when starting with a new repo, where you have files that should be encrypted but they are not, when when you check git-crypt's status:

    $ git crypt status
    not encrypted: .gitattributes
    not encrypted: .gitignore
    not encrypted: README.md
        encrypted: secrets.sample *** WARNING: staged/committed version is NOT ENCRYPTED! ***

This happens because the .gitattributes was not applied before the file was added to the repo. The fix is simple, just re-init for git-crypt:

    $ git crypt init
    Generating key...
    $ git crypt status --fix
    secrets.sample: staged encrypted version
    Staged 1 encrypted file.
    Warning: if these files were previously committed, unencrypted versions still exist in the repository's history.
    $ git crypt status
    not encrypted: .gitattributes
    not encrypted: .gitignore
    not encrypted: README.md
        encrypted: secrets.sample

Now push the repo with any changes, and the files you expected to be encrypted will be. 



# References

* https://www.gnupg.org/gph/en/manual.html
* https://github.com/AGWA/git-crypt

