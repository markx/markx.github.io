---
title: Make CHICKEN Work With HTTPS On Mac
date: 2018-09-03 14:20:06
tags: 
- chicken
- scheme
---


I tried to write a web scraper in CHICKEN Scheme, but it didn't go well not far from the beginning. I had OpenSSL installed via Homebrew, and installed the `http-client` and `openssl` eggs. But when I tried to get the content of an HTTPS website, it showed an error:
```
Error: (ssl-do-handshake) ssl: library=SSL routines, function=ssl3_get_server_certificate, reason=certificate verify failed

	Call history:

	intarweb.scm:798: g2624
	intarweb.scm:775: write-request-line
	intarweb.scm:749: http-method->string
	intarweb.scm:750: uri-common#uri->string
	intarweb.scm:751: number->string
	intarweb.scm:752: number->string
	intarweb.scm:748: string-append
	intarweb.scm:748: display
	http-client.scm:571: k776
	http-client.scm:571: g780
	http-client.scm:700: close-connection!
	http-client.scm:225: close-input-port
	http-client.scm:226: close-output-port
	http-client.scm:701: max-retry-attempts
	http-client.scm:702: max-retry-attempts
	http-client.scm:705: raise	  	<--```


The problem is from the setup between the openssl egg and the OpenSSL library. Because Apple has deprecated use of OpenSSL in favor of its own TLS and crypto libraries, the OpenSSL from Homebrew is not symlinked into the standard location, and its CA file is put at `/usr/local/etc/openssl/cert.pem`. And the openssl egg doesn't know to look at this location.


So to make it work, we need to make sure that the openssl egg uses this CA file. I didn't find a compiler option or some arguments to config this, so I had to change the egg's code to make it work.

Get the openssl egg's source code, and edit `openssl.scm`:

``` bash
$ chicken-install -r openssl
$ cd openssl
$ open openssl.scm
```

In the function `ssl-make-client-context*`, there is the code:
``` scheme
(define (ssl-make-client-context* #!key (protocol 'tlsv12) (cipher-list "DEFAULT") certificate private-key (private-key-type 'rsa) private-key-asn1? certificate-authorities certificate-authority-directory (verify? #t))
  (unless (or certificate-authorities certificate-authority-directory)
    (set! certificate-authority-directory (ssl-default-certificate-authority-directory)))
    
    ...
```

This means if neither the CA file, or the CA directory is available, it uses the default CA directory. And we can do something similar: in addition to using the default CA directory, we just specify the CA file:

``` scheme
(define ssl-default-certificate-authorities
  (make-parameter "/usr/local/etc/openssl/cert.pem"))

(define (ssl-make-client-context* #!key (protocol 'tlsv12) (cipher-list "DEFAULT") certificate private-key (private-key-type 'rsa) private-key-asn1? certificate-authorities certificate-authority-directory (verify? #t))
  (unless (or certificate-authorities certificate-authority-directory)
    (set! certificate-authority-directory (ssl-default-certificate-authority-directory)))
  (unless certificate-authorities
    (set! certificate-authorities (ssl-default-certificate-authorities)))

     ... 
```

Then in the source code directory, install the modified local version:

``` bash
$ chicken-install -t local -l .
```

And that's it. Now the http-client egg should work with HTTPS sites.
