---
title: OWASP Top 10 Lab (Part 1)
date: 2025-07-26 12:00:00 -0400
categories: [Projects]
tags: [AppSec]
image: /assets/img/d1168706.png
---

The [OWASP Top 10](https://owasp.org/www-project-top-ten/) is a list of 10
critical web application security risks and serves as a standard for web
developers and security professionals looking to mitigate any application
security vulnerabilities.

In this lab, I will demonstrate the top 5 security risks and how to mitigate
them. These examples will be pretty bare bones, seeking to show as little as
possible aside from the actual risk. This is to both make the concept on
display clearer and keep the code minimal.

The web app code for each risk can be found on
[GitHub](https://github.com/josephdepalo/owasp10lab).

## 1. Broken Access Control & 2. Cryptographic Failures

### Definitions

**Broken Access Control** refers to when a user can act outside of their
permissions. A normal user being able to access an admin's dashboard would be
an example of this, and this is what we will walk through below.

**Cryptographic Failures** refers to any instance where sensitive data is
missing encryption or weak encryption is used. This also pertains to data that
the end user shouldn't be able to modify being unsigned or using a weak
signature algorithm/key. One example of a cryptographic failure is storing
passwords using an insecure algorithm like SHA-256 and no
[salt](https://www.geeksforgeeks.org/techtips/what-is-password-salting/).

### Walkthrough

![Login Page](/assets/img/f744d67a.png)

First, we will register a user. I'll use the username `user1` and password
`password`. Once our user is created, let's login.

![user1 Dashboard](/assets/img/c823db81.png)

If we take a look at the URL, logging into the website brought us to our
user's dashboard at `/dashboard/2`. It the `/dashboard` part makes sense, but
what is the `/2` referring to? A reasonable guess is some kind of user ID. If
that's the case, then changing this should allow us to see the dashboards of
other users if there aren't controls in place to prevent that.

Let's try accessing `/dashboard/1`.

![Unauthorized Page](/assets/img/5fd2915d.png)

It seems that we aren't authorized to view this page, but now we know that it
exists. Now let's take a look at how this web app is actually tracking our
identity. If we inspect the page and take a look at our cookies, we see only
an `access_token` value.

![Site Cookies](/assets/img/2b22ab4c.png)

The contents of `access_token` look like a **JSON web token (JWT)**. JWTs are
a common token format typically used for stateless authentication to in modern
web apps and APIs. They consist of 3 blocks of base64 encoded text separated
by dots in the form `<Header>.<Payload>.<Signature>`.

We can take a closer look at our JWT using online tools. Copy the token and go
to [jwt.io](https://www.jwt.io/). Here, paste the token into the decoder.

![Decoded JWT](/assets/img/5c33f80e.png)

There are two important things to note about this JWT:

- In the header, `alg` is set to `none` and there is no signature. This means
  we can change the payload to whatever we want.
- The payload contains an `id` field that is set to `2`, the same value we saw
  in the URL for our dashboard. This confirms our suspicions that the value
  after `/dashboard` is an ID.

Since we have the freedom to do whatever we want with this JWT, let's try
changing the payload `id` to `1` (switch to the *JWT Encoder* tab of
[jwt.io](https://www.jwt.io/) and edit the `id` value). Return to the web app
and replace the old JWT with the new one.

Now when we access `/dashboard/1`, we're in!

![Admin Dashboard](/assets/img/55063156.png)

### Mitigation

Be sure to sign and encrypt whenever possible. In this case, it would be
trivial to generate a complex key and change the signature algorithm from
`none` to `HS256`. This would be sufficient to prevent the token forgery that
allowed us to impersonate the admin.

## 3. Injection (and more bad cryptography)

### Definitions

**Injection** refers to when an attacker can send untrusted data into some kind
of interpreter. For example, when making database queries an attacker can
potentially execute arbitrary SQL if proper precautions aren't taken. Command
injection is another thing to look out for and it is what we will show below.

### Walkthrough

Now that we've made it to the admin dashboard, let's do some more digging. The
only thing on the admin dashboard is a form to send commands to the server.
With a quick `ls` we see the files for the Flask app. We're able to execute
arbitrary commands on the server!

A little more poking and we find `/instance/test.db`, which is a sqlite3
database. We can dump it with the below command (the correct table can be found
by executing `.tables;` via sqlite3).

```bash
sqlite3 /instance/test.db "select * from user;"
```

![Dumped Credentials](/assets/img/4506c652.png)

Now we have what seems to be a SHA-256 hash for the admin account (we can check
what kind of hash something is with online tools such as
[hashes.com](https://hashes.com/en/decrypt/hash)). No salt accompanies this
hash, so we should be able to crack it quite easily using online tools.

![Admin Password](/assets/img/42d7cce3.png)

### Mitigation

Anywhere that would allow a user to execute arbitrary commands should be
heavily protected and often it is best to limit the capabilities of these
features. Though not shown in this example, even forms that are not intended to
execute arbitrary commands can be used to do so if fed into some kind of
interpreter (think SQL injections).

As for the passwords, they should not have been hashed with SHA-256. A secure
key derivation function (KDF) should be used instead, like argon2 or bcrypt.
These algorithms both take salts and are purpose built for storing passwords.

## 4. Insecure Design

### Definitions

**Insecure Design** refers to any security design flaw. This is quite broad and
can include many things. For example, not rate limiting login attempts or not
having authorization checks on sensitive functions would be considered insecure
design.

### Walkthrough

As we saw in the previous section, the admin password is pretty simple. We
could try brute forcing the login rather than manipulating JWTs.

Here we'll use the tool [hydra](https://www.kali.org/tools/hydra/) to do so.
Below is a breakdown of the command and its output:

- `-l admin`: Set the username to admin.
- `-P rockyou.txt`: Use the password list `rockyou.txt`.
- `127.0.0.1`: Our target host.
- `-s 5000`: Use port 5000.
- `/login`: The endpoint to target
- `Username=^USER^&Password=^PASS^`: What to send in the POST request body.
  - The fields `Username` and `Password` could be found by inspecting the HTML
      of the login form.
- `Invalid`: Text contained on failed login pages. Anything attempts that have a
  response with this text will not be shown.

```bash
$ hydra -l admin -P rockyou.txt 127.0.0.1 -s 5000 http-post-form "/login:Username=^USER^&Password=^PASS^:Invalid"
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-07-26 20:11:42
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-post-form://127.0.0.1:5000/login:Username=^USER^&Password=^PASS^:Invalid
[5000][http-post-form] host: 127.0.0.1   login: admin   password: password123
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-07-26 20:12:02
```

And just like that, we've found our password `password123` once again. With no
rate limiting of any kind in place and no password policy set, brute forcing
becomes quite easy when users don't go out of their way to set secure
passwords.

### Mitigation

In this case, this kind of brute force attack could be prevented by
implementing a lockout policy and enforcing complex passwords. A brute force
attack would not be nearly as feasible if I wasn't allowed to attempt a login
more than 3 times every 15 minutes per IP address or if the password was
complex enough and not easily guessable.

## 5. Security Misconfiguration

### Definitions

**Security Misconfiguration** typically refers to the improper deployment of
some tool. Default passwords or verbose errors are examples.

### Walkthrough

Since we're running this web app locally we already know that debug mode is on.
This certainly would be a security misconfiguration in a production environment
and can be by an attacker to understand the system better and even see parts of
the code in the case of Flask.

First thing we must do is break something. On `/login`, inspect the HTML and
rename the password input from `Password` to `uh oh` or anything else. Now when
we submit the form, we get quite a verbose error message. Scrolling down we can
click on the block containing `request.form["Password"]` and actually see a
decent chunk of the logic for logging in.

![Verbose Error](/assets/img/3aadb088.png)

While this alone would likely not be enough to bypass authentication or do
something malicious, a lot of information is disclosed from verbose error
messages.

### Mitigation

Simply removing `debug=True` from `app.run` is sufficient to not print verbose
error messages. That aside, better error checking and safer operations should
be used such that the web app rarely throws exceptions in the first place. For
example, instead of indexing `request.form` directly,
`request.form.get("Password", None)` could have been used instead to get
"Password" without ever throwing an exception. A check can then be made if the
value of this is `None` and the app can fail safely.

## Conclusion

This lab covered five critical OWASP Top 10 risks: Broken Access Control,
Cryptographic Failures, Injection, Insecure Design, and Security
Misconfiguration. Each example showed how simple mistakes in code or
configuration can lead to serious vulnerabilities.

By understanding these issues and applying proper security practices early,
developers can reduce risk and build more secure applications. The next part
of this lab will cover the remaining five risks.
