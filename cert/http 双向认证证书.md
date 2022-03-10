#### 1. 术语

###### 1.1 最终实例证书

> 即用户证书

* 最终实体可以是各种类型的实体，如自然人、组织机构、设备、Web服务器等
* 验证最终实体证书（用户证书）需要一个与其颁发者和颁发机构密钥标识符（Authority Key Identifier）匹配的中间证书

###### 1.2 中间证书

> 用作根证书的替代，根密钥可以保持脱机状态，并尽可能不频繁地使用。如果中间密钥被泄露，根CA可以撤销中间证书并创建新的中间密钥对

* 由于根证书签署了中间证书，因此中间证书可用于签署客户安装和维护的SSL“信任链”
* 中间证书的subject字段与它所签署的最终实体证书的issuer字段相同
* 中间证书的subject key identifier（主题密钥标识符）字段与最终实体证书的的authority key identifier（颁发者的密钥标识符）字段相同

###### 1.3 根证书

* Issuer（颁发者字段）和Subject（主题，使用者字段）是相同的
* 如果验证程序在其信任存储中有此根证书，就可以认为在TLS连接中使用的最终实体证书是可信的

###### 1.4 其他

* certs ：专门放证书的
* crl：放证书吊销列表 
* database：数据库索引文件记录了证书的状态啊编号啊以及都给谁颁发了证书默认不存在。需要手工创建，不创建会提示 缺少这个文件
* new_certs_dir：存放新证书目录
* certificate：CA的证书文件
* serial：序列号下一个要颁发的证书编号  16进制数
* crlnumber：吊销证书的编号 需要手工创建
* private_key：私钥文件的路径和名
* policy：定义客户端和CA申请证书的时候它的信息是否和CA是必须匹配的

#### 2. 双向认证生成

##### 2.1 流程图

![双向认证流程](https://www.nginx.org.cn/uploads/article/20200927/dd1384b6ab027e9715a86f583603b881.jpg)

![使用中间证书的双向证书之间关系](https://blog.behrang.org/assets/images/final-ca-env.svg)

##### 2.2 openssl.cnf 配置文件

###### 2.2.1 根CA配置文件

```properties
# OpenSSL root CA configuration file.
# Copy to `/root/ca/openssl.cnf`.

[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = /home/mobaxterm/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = CN
stateOrProvinceName_default     = China
localityName_default            =
0.organizationName_default      = DouLeHuYu Inc
organizationalUnitName_default  = doulehuyu.com
emailAddress_default            =

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

###### 2.2.2 中间CA配置文件

```properties
# OpenSSL intermediate CA configuration file.
# Copy to `/root/ca/intermediate/openssl.cnf`.

[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = /home/mobaxterm/ca/intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = CN
stateOrProvinceName_default     = China
localityName_default            =
0.organizationName_default      = DouLeHuYu Inc
organizationalUnitName_default  = doulehuyu.com
emailAddress_default            =

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
# pathlen:0保证在中间CA下面不能有其他证书颁发机构
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

##### 2.3 双向认证生成

###### 2.3.1 新建目录

> index.txt和serial文件充当平面文件数据库，以跟踪已签名的证书

```bash
mkdir -p /home/mobaxterm/ca/{certs,crl,newcerts,private}
cd /home/mobaxterm/ca
chmod 700 private && touch index.txt && echo 1000 > serial
```

###### 2.3.2 创建根私钥

```bash
[/home/mobaxterm/ca] openssl genrsa -out private/ca.key.pem 4096
```

###### 2.3.3 创建根证书

```bash
[/home/mobaxterm/ca] openssl req -config openssl.cnf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem
Enter pass phrase for private/ca.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [CN]:
State or Province Name [China]:
Locality Name []:
Organization Name [DouLeHuYu Inc]:
Organizational Unit Name [doulehuyu.com]:
Common Name []:DouLeHuYu Inc Root CA
Email Address []:
```

###### 2.3.4 创建中间证书目录

> 将crlnumber文件添加到中间CA目录树，crlnumber用于跟踪证书吊销列表

```bash
mkdir -p /home/mobaxterm/ca/intermediate/{certs,crl,csr,newcerts,private}
cd /home/mobaxterm/ca/intermediate
chmod 700 private && touch index.txt && echo 1000 > serial && echo 1000 > crlnumber
```

###### 2.3.5 创建中间证书密钥

```bash
[/home/mobaxterm/ca] openssl genrsa -out intermediate/private/intermediate.key.pem 4096
```

###### 2.3.6 使用中间证书创建证书签名请求

> 详细信息通常应与根CA相同，Common Name（证书持有者通用名/FQDN）必须不同

```bash
[/home/mobaxterm/ca] openssl req -config intermediate/openssl.cnf -new -sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
Enter pass phrase for intermediate/private/intermediate.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [CN]:
State or Province Name [China]:
Locality Name []:
Organization Name [DouLeHuYu Inc]:
Organizational Unit Name [doulehuyu.com]:
Common Name []:DouLeHuYu Inc CA
Email Address []:
```

###### 2.3.7 创建中间证书

> 使用带有v3_intermediate_CA扩展项的根CA对中间CSR进行签名，中间证书的有效期应短于根证书

```bash
[/home/mobaxterm/ca] openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
Using configuration from openssl.cnf
Enter pass phrase for /home/mobaxterm/ca/private/ca.key.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4098 (0x1002)
        Validity
            Not Before: Jan 28 07:38:00 2022 GMT
            Not After : Jan 26 07:38:00 2032 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = China
            organizationName          = DouLeHuYu Inc
            organizationalUnitName    = doulehuyu.com
            commonName                = DouLeHuYu Inc CA
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                0E:77:98:80:4D:11:70:3E:F0:BD:76:E6:C9:BA:2D:A0:ED:B2:23:06
            X509v3 Authority Key Identifier:
                keyid:94:47:02:25:88:0C:FF:EE:C1:FF:E4:32:BE:C3:4D:75:C6:4E:06:64

            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
Certificate is to be certified until Jan 26 07:38:00 2032 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

###### 2.3.8 创建证书链文件

> 当应用程序（如web浏览器）验证由中间CA签名的证书时，它必须向上溯到根证书，完成信任链，并使用此文件来验证由中间CA签名的证书

```bash
[/home/mobaxterm/ca] cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem

[/home/mobaxterm/ca] chmod 444 intermediate/certs/ca-chain.cert.pem
```

###### 2.3.9 创建服务器私钥

```bash
[/home/mobaxterm/ca] openssl genrsa -out intermediate/private/test.actumtech.com.key.pem 2048
```

###### 2.3.10 创建服务器证书签名请求

> 使用私钥创建证书签名请求（CSR），并且CSR的详细信息无需与中间CA相匹配，Common Name（公用名）必须是FQDN（完全限定的域名，例如，www.example.com）

```bash
[/home/mobaxterm/ca] openssl req -config intermediate/openssl.cnf -key intermediate/private/test.actumtech.com.key.pem -new -sha256 -out intermediate/csr/test.actumtech.com.csr.pem
Enter pass phrase for intermediate/private/test.actumtech.com.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [CN]:
State or Province Name [China]:
Locality Name []:
Organization Name [DouLeHuYu Inc]:
Organizational Unit Name [doulehuyu.com]:
Common Name []:*.test.actumtech.com
Email Address []:
```

###### 2.3.11 创建服务器证书

```bash
[/home/mobaxterm/ca] openssl ca -config intermediate/openssl.cnf -extensions server_cert -days 375 -notext -md sha256 -in intermediate/csr/test.actumtech.com.csr.pem -out intermediate/certs/test.actumtech.com.cert.pem
Using configuration from intermediate/openssl.cnf
Enter pass phrase for /home/mobaxterm/ca/intermediate/private/intermediate.key.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4099 (0x1003)
        Validity
            Not Before: Jan 28 08:01:22 2022 GMT
            Not After : Feb  7 08:01:22 2023 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = China
            organizationName          = DouLeHuYu Inc
            organizationalUnitName    = doulehuyu.com
            commonName                = *.test.actumtech.com
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            Netscape Comment:
                OpenSSL Generated Server Certificate
            X509v3 Subject Key Identifier:
                AE:93:C1:2D:AA:7E:B0:E1:8F:14:26:4A:F6:40:B9:4C:46:3D:76:0D
            X509v3 Authority Key Identifier:
                keyid:0E:77:98:80:4D:11:70:3E:F0:BD:76:E6:C9:BA:2D:A0:ED:B2:23:06
                DirName:/C=CN/ST=China/O=DouLeHuYu Inc/OU=www.doulehuyu.com/CN=DouLeHuYu Inc Root CA
                serial:10:02

            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
Certificate is to be certified until Feb  7 08:01:22 2023 GMT (375 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

###### 2.3.12 创建客户端私钥

```bash
[/home/mobaxterm/ca] openssl genrsa -out intermediate/private/doule.key.pem 2048
```

###### 2.3.13 创建客户端证书签名请求

> 使用私钥创建证书签名请求（CSR），并且CSR的详细信息无需与中间CA相匹配，Common Name可以是任何唯一标识符，客户端证书的Common Name与根证书或中间证书的Common Name不同

```bash
[/home/mobaxterm/ca] openssl req -config intermediate/openssl.cnf -key intermediate/private/doule.key.pem -new -sha256 -out intermediate/csr/doule.csr.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [CN]:
State or Province Name [China]:
Locality Name []:
Organization Name [DouLeHuYu Inc]:
Organizational Unit Name [doulehuyu.com]:
Common Name []:doule
Email Address []:
```

###### 2.3.14 创建客户端证书

```bash
[/home/mobaxterm/ca] openssl ca -config intermediate/openssl.cnf -extensions usr_cert -days 375 -notext -md sha256 -in intermediate/csr/doule.csr.pem -out intermediate/certs/doule.cert.pem
Using configuration from intermediate/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4101 (0x1005)
        Validity
            Not Before: Jan 28 08:15:26 2022 GMT
            Not After : Feb  7 08:15:26 2023 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = China
            organizationName          = DouLeHuYu Inc
            organizationalUnitName    = doulehuyu.com
            commonName                = doule
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Client, S/MIME
            Netscape Comment:
                OpenSSL Generated Client Certificate
            X509v3 Subject Key Identifier:
                52:7A:65:1F:17:26:2A:4F:FC:51:9D:90:C2:B0:D3:68:5A:00:8E:98
            X509v3 Authority Key Identifier:
                keyid:B4:31:1D:41:67:B8:69:FC:A9:3F:F4:21:04:30:F8:3C:C4:6C:A5:DA

            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Client Authentication, E-mail Protection
Certificate is to be certified until Feb  7 08:15:26 2023 GMT (375 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

###### 2.3.15 双向认证验证

* openssl

```bash
[/home/mobaxterm/ca] openssl s_client -connect index.test.actumtech.com:4321 -tls1 -key intermediate/private/doule.key.pem -cert intermediate/certs/doule.cert.pem -CAfile intermediate/certs/ca-chain.cert.pem -state -debug
```

* curl

```bash
[/home/mobaxterm/ca] curl -H 'Content-type':'application/json' --cacert intermediate/certs/ca-chain.cert.pem --cert intermediate/certs/doule.cert.pem --key intermediate/private/doule.key.pem -v "https://index.test.actumtech.com:4321/meisuindex/applet/getList?p=0&r=10"
```

###### 2.3.16 客户端证书p12生成

* 客户端P12文件

```bash
[/home/mobaxterm/ca] openssl pkcs12 -export -cacerts -CAfile intermediate/certs/ca-chain.cert.pem -inkey intermediate/private/doule.key.pem -in intermediate/certs/doule.cert.pem -out doule.p12
```

* CA P12文件

```bash
[/home/mobaxterm/ca] openssl pkcs12 -export -cacerts -inkey private/ca.key.pem -in certs/ca.cert.pem -out ca.p12
```

###### 2.3.17 nginx配置

```nginx
server {
    listen 443 ssl;
    server_name aaab.com;
    ssl_certificate  /etc/nginx/conf.d/cert/test.actumtech.com.crt.pem;
    ssl_certificate_key /etc/nginx/conf.d/cert/test.actumtech.com.key.pem;
    ssl_client_certificate /etc/nginx/conf.d/cert/ca-chain.cert.pem;
    ssl_verify_client on;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        proxy_set_header Host aaab.com;
        proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_pass http://test;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

<h6>参考资料</h6>

1. [X509证书详解（中文翻译）](https://www.cnblogs.com/nirvanan/articles/13815185.html)
2. [Root CA configuration file](https://jamielinux.com/docs/openssl-certificate-authority/appendix/root-configuration-file.html)
3. [证书用法](https://www.cnblogs.com/i2u9/p/x509certtypes.html)
4. [Creating a rudimentary private certificate authority using OpenSSL](https://blog.behrang.org/articles/creating-a-ca-with-openssl.html)
5. [中间证书和根证书的认证](https://blog.51cto.com/u_9843231/2466504)
6. [OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/introduction.html)
7. [Becoming a X.509 Certificate Authority](https://www.davidpashley.com/articles/becoming-a-x-509-certificate-authority/)
8. [openssl制作V3版证书实现基于https的 webservice双向认证](https://blog.csdn.net/weixin_33749131/article/details/92344596)
9. [HTTPS双向认证（Mutual TLS authentication)](https://help.aliyun.com/document_detail/160093.html?spm=5176.21213303.J_6028563670.7.2be23edaf7ftgX&scm=20140722.S_help%40%40%E6%96%87%E6%A1%A3%40%40160093.S_0.ID_160093-RL_%E5%8F%8C%E5%90%91%E8%AE%A4%E8%AF%81-OR_s%2Bhelpproduct-V_1-P0_0)
10. [使用SLB部署HTTPS业务（双向认证）](https://help.aliyun.com/document_detail/85954.html?spm=5176.21213303.J_6028563670.11.2be23edaf7ftgX&scm=20140722.S_help%40%40%E6%96%87%E6%A1%A3%40%4085954.S_0.ID_85954-RL_%E5%8F%8C%E5%90%91%E8%AE%A4%E8%AF%81-OR_s%2Bhelpproduct-V_1-P0_1)
11. [Nginx https 双向认证](https://www.cnblogs.com/yelao/p/9486882.html)
12. [NGINX 配置本地HTTPS(双向认证)](https://www.nginx.org.cn/article/detail/300)
13. [使用 OpenSSL 制作一个包含 SAN（Subject Alternative Name）的证书](https://blog.csdn.net/weixin_34387468/article/details/91855502)
14. [openssl 生成X509 V3的根证书及签名证书](https://blog.csdn.net/xiangguiwang/article/details/80333728)
15. [x509v3_config](https://www.openssl.org/docs/man1.0.2/man5/x509v3_config.html)
16. [Fix depth lookup:unable to get issuer certificate](https://wiki.zimbra.com/wiki/Fix_depth_lookup:unable_to_get_issuer_certificate)
17. [Openssl.conf Walkthru](https://www.phildev.net/ssl/opensslconf.html)
18. [Creating a CA](https://www.phildev.net/ssl/creating_ca.html)
19. [HTTPS双向认证指南 - 简书](https://www.jianshu.com/p/2b2d1f511959)
