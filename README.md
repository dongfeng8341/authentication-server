# authentication-server [![Build Status](https://travis-ci.org/Neufund/authentication-server.svg)](https://travis-ci.org/Neufund/authentication-server)

JWT Authentication server using email, password, captcha and Google Authenticator

Main Features:

* Password strengthening using [scrypt][7] [RFC7914][5]
* Password authentication using [SRP-6a][1] [RFC2945][6] [RFC5054][2]
* One time tokens using [TOTP][11] [RFC4226][8] [RFC6238][3]
* Human detection using [reCAPTCHA][4]
* Authorized using [JWT][9] [RFC7519][10]

[1]: http://srp.stanford.edu/
[2]: https://tools.ietf.org/html/rfc5054
[3]: https://www.ietf.org/rfc/rfc6238.txt
[4]: https://developers.google.com/recaptcha/intro
[5]: https://tools.ietf.org/html/rfc7914
[6]: https://tools.ietf.org/html/rfc2945
[7]: https://www.tarsnap.com/scrypt.html
[8]: https://tools.ietf.org/html/rfc4226
[9]: https://jwt.io/
[10]: https://tools.ietf.org/html/rfc7519
[11]: https://en.wikipedia.org/wiki/Time-based_One-time_Password_Algorithm

## Build, test and run

```
virtualenv -p $(which python3) env
source env/bin/activate
pip install -r requirements.txt
```

Generate a keypair:
```
openssl ecparam -genkey -name secp521r1 -noout -out ec512.prv.pem
openssl ec -in ec512.prv.pem -pubout > ec512.pub.pem
```

Run tests:
```
pytest -v src tests
```

Start server:
```
FLASK_APP=src/server.py flask run
```

## How to use

TODO

*Recommended client side libraries*:

* https://www.npmjs.com/package/srp
* https://www.npmjs.com/package/scrypt-async

TODO: authentication-client project


## Specification

### Database Schema

|        Column        |   Type   |                      Description                      |
|----------------------|----------|-------------------------------------------------------|
| `created`            | datetime | Timestamp of account signup, does not change          |
| `updated`            | datetime | Timestamp of last change                              |
| `used`               | datetime | Timestamp of last login                               |
| `enabled`            | bool     | Admin toggle for enabling/disabling accounts          |
| `email`              | string   | Account email address                                 |
| `email_new`          | string   | Newly submitted, but unconfirmed email address        |
| `email_token`        | string   | Email address confirmation token (empty if confirmed) |
| `email_token_expiry` | datetime | Validity for email token                              |
| `salt`               | bytes    | Salt used for SCrypt and SRP                          |
| `verifier`           | bytes    | SRP Verifier                                          |
| `totp`               | bytes    | Time-based one-time-password secret                   |

### Parameters

#### scrypt

|         |       |
|---------|-------|
| `N`     | 16384 |
| `r`     |     8 |
| `p`     |     1 |
| `dkLen` |    32 |

The result taken in raw `binary` form. 256 bit random salt.

#### SRP-6a

4096-bit Group parameters (RFC5054 [§A.5][rfc5054-18]). 256 bit random salts and nonces.

[rfc5054-18]: https://tools.ietf.org/html/rfc5054#page-18

### Sign-up

Note: Captcha prevents this from automated detecting user existence

1. User: generates random 32 byte `salt`
5. User: `key = scrypt(password, salt)`
6. User: `verifier = SRP6A_verifier(email, key, salt)`
3. User submits
    * `email`
    * `salt`
    * `verifier`
    * recaptcha token
6. Server validate recaptcha
7. Server creates new user with submitted data

### Log-in

1. User calls `/challenge` with `email`
2. Server looks up `salt` and `verifier`
2. Server: `challenge = SRP6A_challenge(salt, verifier)`
3. Server responds:
    * `salt`
    * `mac(challenge)`
5. User: `key = scrypt(password, salt)`
6. User: `response = SRP6A_response(challenge, salt, key)`
7. User submits `/response`:
    * `email`
    * `mac(challenge)`
    *


## FAQ

### Why are passwords so difficult?

* Because they often have low entropy (like the codes on a suitcase, you can try all options in a few days).

* Because people use the same password in many places, so even if your site is not important—the same password may access the users bank account.

### What's wrong with sending plain password over https?

CA's are unreliable, TLS is too complex/buggy.

* http://heartbleed.com/
* https://en.wikipedia.org/wiki/Cloudbleed
* https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/eUAKwjihhBs

### What's wrong with sending `hash(salt, pwd)`?

Does not protect against password brute forcing. An attacker who has the salt and hash can rapidly try combinations.

### Why not just Scrypt?

It's a lot better, but an attacker that has the salt and hash can still try passwords without being detected. Just much more slowly.

### Why not just SRP-6a?

SRP protects against eavesdroppers, nothing send over the wire will help an attacker.

But now the server stores something that is roughly similar to a salted hash: if an attacker obtains the user database, (s)he can quickly do an brute force.

By using both we make it impossible to attack the password when the TLS leaks and very hard when the user database is leaked.

### Why not SMS Verification?

[SMS is not secure][kraken-sms]

[kraken-sms]: http://blog.kraken.com/post/153209105847/security-advisory-mobile-phones

### Why generate the salt client side

It is in the clients interest to protect itself.
