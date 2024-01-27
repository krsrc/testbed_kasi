# Certificates

## References

- SSL and CA introduction: <https://m.blog.naver.com/alice_k106/221468341565>

## Remove password from `key.pem`

Convert RSA private `key.pem` to `cert.key`.

```bash
openssh rsa -in key.pem -out cert.key
```

## Convert `.pem` to `.crt`

> [!CAUTION]
> When the certificate is structured in a chain, 
> the above command does not include the contents of the root and intermediate CAs in the output. 
> While browsers usually have most well-known intermediate CAs, including the root CA, 
> so do not encounter issues, 
> but problems may arise when a full chain CA is required, as in the case like curl. 
> Therefore, to establish a chain of trust, 
> it is necessary to create a crt file that includes all CAs from each layer.
>
> ```bash
> cat start.pem intermediate.pem root.pem > cert.crt
> ```

```bash
openssh x509 -inform PEM in cert.pem -out cert.crt
```

> [!NOTE]
> In case of CA for *.kasi.re.kr, .pem and .crt is exactly same.
> So, we can just rename .pem to .crt.
>
> ```bash
> ln -s cert.pem cert.crt
> ```
>
> In this case, there is no fullchain issue.
