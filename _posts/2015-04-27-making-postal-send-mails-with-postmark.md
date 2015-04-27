---
layout: post
title: "Making Postal send mails with Postmark"
date: 2015-04-27 20:21
comments: true
categories: [ ".net", "oss", "open source", "email" ]
---

In my [previous post][1] I praised [Postal][4] for being such a simple and easy library
to use for templating and sending emails from your .net app. I also briefly mentioned 
how it just uses the standard SMTP settings from your `Web.config` or `App.config` file 
for sending out the mails.

This is fine in most scenarios, but on one of our last projects we were using [Postmark][2]
and gave preference to using the HTTP API instead of relying on the SMTP protocol. The main
advantage of this is that HTTP tends to be far less chatty than SMTP and you can increase
the throughput of your mails.

## Step 1: Install Postmark .net library

From the Package Manager Console inside Visual Studio do:

    Install-Package Postmark

This installs the official Postmark .net library from NuGet. 

## Step 2: Implement an `IEmailService`

An `IEmailService` is Postal's way to allow you to alter the behaviour of sending the email.
The [default implementation][3] uses the `System.Net.Mail` namespace to compose a `MailMessage` instance
and send it using the configuration that sits in your `.config` file.

This is what we want to change in order to start using the Postmark API, we can send emails using the
following code snippet. 

``` csharp    
    MailMessage message = ...;

    var client = new PostmarkClient("<your api token goes here>");
    client.SendMessageAsync(message);
```

This is absolutely perfect, because the default `EmailService` implementation contains a method to create
a MailMessage based on an `Email` object, so we can simply use that and pass the `MailMessage` to the
`PostmarkClient`. The result is the following service:

``` csharp
    public class PostmarkEmailService : IEmailService
    {
        // the default implementation used to generated
        // the MailMessage objects
        private Postal.EmailService _inner;
        private PostmarkClient _client;

        public PostmarkEmailService(string token)
        {
            _inner = new EmailService();
            _client = new PostmarkClient(token);
        }

        public void Send(Email email)
        {
            var msg = CreateMailMessage(email);
            _client.SendMessageAsync(msg)
                   .Wait();
        }

        public Task SendAsync(Email email)
        {
            var msg = CreateMailMessage(email);
            return _client.SendMessageAsync(msg);
        }

        public MailMessage CreateMailMessage(Email email)
        {
            return _inner.CreateMailMessage(email);
        }
    }
```

## Step 3: Set an instance of our service as the default one

Now that we have a service that can send Emails using the Postmark API, the only thing that
remains to be done is to configure Postal the use it as `IEmailService` used for sending
out the mails during the startup of our app.

``` csharp
    Email.CreateEmailService = () => new PostmarkEmailService("<your token goes here>");
```

## Conclusion

That's it! From now on all `Send` calls on an `Email` object will use Postmark to send out
the emails. 
Changing Postal's default send implementation is super easy, thanks to the
nice NuGet package that Postmark provides and to the extensibility of Postal.

[1]: /blog/2015/03/17/open-source-net-libraries-that-make-your-life-easier/
[2]: http://www.postmarkapp.com
[3]: https://github.com/andrewdavey/postal/blob/master/src/Postal/EmailService.cs 
[4]: http://aboutcode.net/postal/

