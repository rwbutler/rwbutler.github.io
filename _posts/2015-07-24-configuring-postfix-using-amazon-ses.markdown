---
layout: post
title:  "Configuring Postfix using Amazon SES on Mac OS X Mavericks (10.9.5)"
date:   2015-07-24 16:23:35
categories: postfix osx mavericks amazon ses aws
---

Our Continuous Integration (CI) process is largely comprised of Bash scripts and we needed to be able to send e-mail notifications in the event of build failures. On a Mac OS X server, the de facto method of achieving this is via the Postfix [sendmail](http://www.postfix.org/sendmail.1.html) command. In order to make use of the command, the [Postfix](http://www.postfix.org/) service needs to be configured to relay mail to another SMTP server. In this blog post, we'll use [Amazon's Simple E-mail Service (SES)](http://aws.amazon.com/ses/) as the relay.

*Note: This blog post applies primarily to  Mac OS X Mavericks (10.9.5).* 

Firstly, navigate to the `etc/postfix` directory and open `main.cf` for editing. Much of the file content is commented out. Scroll down in the file to where you'll find a number of lines reading `relayhost=`.

Just below these lines, paste the following configuration settings for Amazon SES's SMTP server in the region relevant to you *(as documented in the AWS Developer Guide - see [Using the Amazon SES SMTP Interface to Send Email](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/postfix.html))*:

{% highlight bash %}

relayhost = [email-smtp.eu-west-1.amazonaws.com]:25
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_use_tls = yes
smtp_tls_security_level = encrypt
smtp_tls_note_starttls_offer = yes

{% endhighlight %}

*Note: The SMTP settings you need to use may vary slightly according to the region in which your AWS service is based. To determine the SMTP settings you need to use - from the AWS dashboard under Application Services, select SES. Then, from the SES Management Console choose SMTP Settings. This page will display the server settings and port number you'll need to use in place of the above.*

**Mac OS X Yosemite Users will need to add the line:**

{% highlight bash %}

smtp_sasl_mechanism_filter = plain

{% endhighlight %}

Next, we need to configure the credentials to be used to authenticate to Amazon SES. The simplest way to obtain your credentials is to navigate to SMTP Settings in the SES Management Console and click `Create My SMTP Credentials`. This will generate credentials for the user account you are signed into AWS with. 

If you need credentials for a separate account, you'll need to make sure that you have an access key for API access. You can create one of these through [AWS Identity and Access Management (IAM)](http://aws.amazon.com/iam/). From the IAM dashboard, either create or select an existing user then click the `Create Access Key` button. Once generated, the access key identifier will become your SMTP username. To obtain your password, download the [SES SMTP Credential Generator Java code](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/smtp-credentials.html) and save it in a file called `SesSmtpCredentialGenerator.java`. Assuming that you have the [Java Development Kit (JDK)](http://www.oracle.com/technetwork/java/javase/downloads/index.html) installed, from Terminal change directory to the directory containing the Java file you recently created and type:

{% highlight bash %}

javac SesSmtpCredentialGenerator.java

{% endhighlight %}

This will generate a file called `SesSmtpCredentialGenerator.class` in the same directory. Before running the compiled code, you'll need to obtain the secret access key (the other half of the access key that was generated for you by AWS alongside the access key identifier) and set this as an environment variable in Terminal by typing:

{% highlight bash %}

export AWS_SECRET_ACCESS_KEY="<insert your AWS secret access key here>"

{% endhighlight %}

Next, run the credential generator:

{% highlight bash %}

java SesSmtpCredentialGenerator

{% endhighlight %}

The credential generator will read the AWS_SECRET_ACCESS_KEY environment variable and generate your SMTP password by applying an algorithm to your AWS secret key. Now that you have your SMTP password, you'll probably want to unset the AWS_SECRET_ACCESS_KEY variable just be to safe. Typing the following should do the trick:

{% highlight bash %}

AWS_SECRET_ACCESS_KEY=""

{% endhighlight %}

Now that we have the username and password required to authenticate to Amazon SES, we're now ready to configure the Postfix service to use them. To do this, we need to generate a hashmap database which can be used by Postfix to lookup credentials. To achieve this first of all open `/etc/postfix/sasl_passwd` for editing. If using an editor in Terminal for this, you'll need to make use of the `sudo` command. Next, add the line:

{% highlight bash %}

[email-smtp.eu-west-1.amazonaws.com]:25 USERNAME:PASSWORD

{% endhighlight %}

Where the username is your AWS access key identifier we obtained above and the password is either that generated via the `Create My SMTP Credentials` button or generated by the Java code.

**Note: Ensure that the SMTP server and port number specified as part of this line match the ones you specified as part of the `relayhost` entry in Postfix's `main.cf` earlier.**

To generate the database next type:

{% highlight bash %}

sudo postmap /etc/postfix/sasl_passwd

{% endhighlight %}

This will generate the file `/etc/postfix/sasl_passwd.db`. Once this file has been generated it is recommended that you remove the original `/etc/postfix/sasl_passwd` file.

It is important to note that `sasl_passwd.db` now contains your **unencrypted SMTP credentials** so we need to restrict access to this file. The following commands will attribute ownership of the database to the root user and restrict access to all other users:

{% highlight bash %}

sudo chown root:root /etc/postfix/sasl_passwd.db

sudo chmod 0600 /etc/postfix/sasl_passwd.db

{% endhighlight %}

*If for whatever reason, you elected not to delete the original `sasl_password` file you should apply the same commands to that file.*

You should now be ready to restart the Postfix service so that it picks up on the changes we've have added to `main.cf`. The following commands stop and start the service:

{% highlight bash %}

sudo launchctl stop org.postfix.master

sudo launchctl start org.postfix.master

{% endhighlight %}

We are now almost ready to send a test e-mail to check that our changes have been supplied successfully. If the account used to authenticate to Amazon SES is in the SES sandbox (which it likely will be unless you have used it to send e-mail before) then we will need to verify the e-mail addresses we wish to send e-mail from and to.

To do so, navigate to the *E-mail Addresses* section of the SES Management Console and add each e-mail address to the verification list. AWS to send an e-mail to each of the addresses you specify here containing a link which must be clicked on to verify the address.

Alternatively, you can follow the Amazon Developer Guide to find out about [Moving Out of the Amazon SES Sandbox](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/request-production-access.html).

Now that we have verified the e-mail addresses we are going to use, we are finally ready to test that our changes have been applied successfully by sending a test e-mail using the Postfix `sendmail` command. The first e-mail address supplied is the **from** address and the second is the **to** address. After typing the first line below, an interactive prompt will open to allow you to type in the remaining three lines and will close after typing a full stop **.** on a new line ending the interactive session.

{% highlight bash %}

sendmail -f from@example.com to@example.com

From: from@example.com

Subject: Test

This e-mail was sent using sendmail!

.

{% endhighlight %}

With any luck after a short period of time, the e-mail should arrived in your inbox!

## Troubleshooting

If for some reason, you don't receive the test e-mail, try inspecting the mail queue using the `mailq` command to see whether the message has been sent or whether it is languishing in the queue for some reason. If the message is stuck in the queue or hasn't entered the queue, try inspecting the mail log at `/var/log/mail.log` to see if any error logged messages exist to indicate the source of the fault.

Once you have resolved the problem, flushing the queue as follows will cause Postfix to retry sending all messages in the queue:

{% highlight bash %}

sudo postfix flush

{% endhighlight %}

# Useful Commands

The following commands were useful whilst configuring Postfix and diagnosing errors:

#### Start Postfix

{% highlight bash %}

sudo launchctl start org.postfix.master

{% endhighlight %}

#### Stop Postfix

{% highlight bash %}

sudo launchctl stop org.postfix.master

{% endhighlight %}

#### View the mail queue

{% highlight bash %}

mailq

{% endhighlight %}

#### Flush the mail queue
*Useful to re-attempt sending all mail in the queue*

{% highlight bash %}

sudo postfix flush

{% endhighlight %}

#### Delete all jobs from the mail queue

{% highlight bash %}

sudo postsuper -d ALL

{% endhighlight %}

#### Create a Postfix lookup table (hashmap)
*For storing SMTP credentials*

{% highlight bash %}
sudo postmap /etc/postfix/sasl_passwd
{% endhighlight %}

#### View the mail log
*Useful to diagnose errors when sending mail*

{% highlight bash %}
vi /var/log/mail.log
{% endhighlight %}

# References

Further information about each of the steps covered here can be found as part of the Amazon Developer Guide.


- [Using the Amazon SES SMTP Interface to Send Email](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-email-smtp.html)
- [Integrating Amazon SES with Postfix](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/postfix.html)
- [Obtaining Your Amazon SES SMTP Credentials](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/smtp-credentials.html)
- [Verifying Email Addresses in Amazon SES](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses.html)
