---
layout: post
title:  "Signing Commits Using GPG on macOS"
subtitle: "Protect Your Git Repositories From Commit Forgery Using Signing."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/monitors.png" alt="Monitors">
</div>

If you are a developer using Git as part of your workflow on a regular basis then you’re probably familiar with the concept of using SSH keys to push commits to a remote repository (if you use a URL of the form ssh://) potentially hosted on GitHub, GitLab, Bitbucket etc. Alternatively, you may use a username and password to access your remote repository over HTTPS. Did you know though that whilst your credentials identify you to the remote server (and thus determine the actions you may perform within the remote repo), they do nothing to identify your commits as belonging to you? Try changing your Git name and e-mail address to those of someone completely different as follows:

```
git config --global user.name "John Appleseed"
git config --global user.email "john.appleseed@example.com"
```

Using the same SSH keys, you will have the same level access to the repository as you had before and will be able to push commits (providing you had this level of access before) under the the new identity you have switched to. This may not be a big deal if your remote Git repository is privately hosted with access strictly controlled however if you commit to a popular open source project with many contributors it could be feasible to masquerade as another contributor, say someone who is well-known and trustworthy in order to sneak miscreant code into the project.

The solution to this problem is to cryptographically sign commits submitted to the remote repository. By signing commits we can verify that the code committed was actually written by the author named on the commit. Two popular methods of signing commits include [GPG](https://en.wikipedia.org/wiki/GNU_Privacy_Guard) and [S/MIME](https://en.wikipedia.org/wiki/S/MIME). For the purposes of this article, we’ll be focussing on signing commits using GPG as is more common.

# Generating a key pair

The quickest way to get started signing your commits on macOS is to download a copy of [GPGSuite](https://gpgtools.org), a set of tools for signing files and messages on macOS. Crucially, GPGSuite allows us to store the passphrase for our GPG private key in the macOS Keychain meaning that we won’t have to type it out each time we commit to our repository.

With GPGSuite installed, the next step is generate a pair of cryptographic keys — one public which will be uploaded to our version control server and another private which will be held locally on your machine. GPGSuite provides a tool called GPG Keychain which can be used to achieve this. Open GPG Keychain and select _New_.

A dialogue box prompting you to enter your name, e-mail address and a passphrase to protect your public / private key pair will be shown as follows:

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/gpg-keychain.png" alt="GPG Keychain">
</div>

<br/>

<div align="center">
_GPG Keychain_
</div>

Make sure to enter the name and e-mail address you will be committing to Git under. These are the details you specified when you initially set up Git on your machine using the git config command e.g.

```
git config --global user.name "John Appleseed"
git config --global user.email "john.appleseed@example.com"
```

If you have not yet set up Git on your machine, you will need to run the above commands substituting in your own name and e-mail address. If you use a visual Git client rather than the command line e.g. Tower, you may have entered this information into the client which will have executed these commands on your behalf. Use of the `--global` parameter sets the specified name / e-mail address as the default for all Git repositories on your machine however you may omit the parameter in order to set your name and e-mail address on a per-repository basis.

# Configuring the Git signing key

With your key pair generated, the next step is to tell Git which private key to use to sign your commits. Use GPG Suite to list your secret keys as follows:

```
gpg --list-secret-keys --keyid-format LONG
```

You should see some output as follows:

```
— — — — — — — — — — — — — — — — — 
sec rsa4096/543C1DB7EF341B87 2019–02–12 [SC] [expires: 2023–02–12]
    6867CE356C43ED4F9C5B5C64367C3DB3CD214C87
uid [ultimate] John Appleseed<john.appleseed@example.com>
ssb rsa4096/34DC284DBBA54BDC 2019–02–12 [E] [expires: 2023–02–12]
```

If you have multiple secret keys, this output will repeated for each secret key. Look for the portion of the output in __bold__ above — this is the identifier of your secret key. To instruct Git to use the secret key with this identifier run the following command, substituting the identifier of your secret key:

```
git config --global user.signingkey 543C1DB7EF341B87
```

Again, you may omit the `--global` parameter in order to set a signing key to be used on a per repo basis.

# Exporting your public key

Now that Git knows which signing key is to be used to sign your commits, your Git remote (server) e.g. GitHub, GitLab, Bitbucket etc. needs a copy of your public key in order to verify your signed commits. Open GPG Keychain and select the public key to be exported from the list of key pairs shown then hit the Export button. You will be prompted to save your public key to a file on disk. Ensure that the _Include secret key_ in _exported file_ option is __not selected__.

Once exported you should have a text file beginning and ending with the following header and footer respectively. In between should be a long sequence of characters comprising your encoded public key.

```
-----BEGIN PGP PUBLIC KEY BLOCK-----
<PUBLIC KEY HERE>
-----END PGP PUBLIC KEY BLOCK-----
```

Copy & paste the entire block including the headers and footers into your remote Git server e.g. in GitHub, navigate to _Settings -> SSH and GPG Keys_ then hit the _New GPG Key_ button — a text field will be displayed for you to paste your key. In GitLab, this feature is available [from version 9.5 onwards](https://docs.gitlab.com/ee/user/project/repository/gpg_signed_commits/).

Full instructions exist for some popular Git hosting services including:

- [GitHub](https://help.github.com/en/articles/signing-commits)
- [GitLab](https://docs.gitlab.com/ee/user/project/repository/gpg_signed_commits/)
- [Bitbucket](https://confluence.atlassian.com/bitbucketserver/using-gpg-keys-913477014.html)

With your public key uploaded to the remote server it should be possible for the server to verify signed commits.

# Signing commits

All that is left to do is make a signed commit. To ensure that Git signs commits by default, you can enabled commit signing globally (_or per repo without the `--global` parameter_) as follows:

```
git config --global commit.gpgsign true
```

Alternatively when committing, supply the -S parameter to sign a commit as a one-off e.g.

```
git commit -S -m 'This commit is signed'
```

The first time you sign a commit, GPG Keychain will display a prompt asking you to enter passphrase you supplied when you initially generated your key pair so that the private key may be read from the encrypted file where it is stored on disk. An option can be checked on the prompt allowing you to save the passphrase in the macOS Keychain so that when signing future commits you needn’t be prompted for the passphrase each time.

And then you’re done! 🎉

# Further Information

Commits pushed to GitHub should show a VERIFIED badge to indicate that these commits have been cryptographically signed. It is worth noting that it is also possible to sign tags in Git by supplying the `-s` parameter e.g.

```
git tag -s 1.5.0
```

Many popular Git hosting services allow us to go a step further and require that contributions to a repository be signed, for more information see:

- [GitHub — About Required Commit Signing](https://help.github.com/en/articles/about-required-commit-signing)
- [GitLab — Rejecting Commits That Are Not Signed](https://docs.gitlab.com/ee/user/project/repository/gpg_signed_commits/index.html#rejecting-commits-that-are-not-signed-premium)
- [Bitbucket — Using GPG Keys / Requiring GPG Keys](https://confluence.atlassian.com/bitbucketserver/using-gpg-keys-913477014.html)