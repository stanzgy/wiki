## Create self-signed certificates with OpenSSL

### Root CA

#### Create Root CA key (enable password with '-des3' option)

    $ openssl genrsa -des3 -out root.key 4096


#### Create Root CA

    $ openssl req  -new -x509 -sha256 -days 3650 -key root.key -reqexts v3_req -extensions v3_ca -out root.crt


### Intermediate CA

#### Create Intermediate CA key

    $ openssl genrsa -out ca.key 4096


#### Create Intermediate CA request

    $ openssl req -new -key ca.key -reqexts v3_req -extensions v3_ca -out ca.csr


#### Sign Intermediate CA request

    $ openssl x509 -req -in ca.csr -CA root.crt -CAkey root.key -CAcreateserial -CAserial ca.srl -extfile v3_ca.ext -days 3650 -sha256 -out ca.crt


#### Create CA chain file

    $ cat ca.crt root.crt > ca_chain.crt


### Server Certificates

#### Create Server certificate key

    $ openssl genrsa -out server.key 4096


#### Create Server certificate request

    $ openssl req -new -key server.key -out server.csr


#### Sign Server certificate

    $ openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -CAserial server.srl -days 730 -sha256 -out server.crt


#### Create Server Certificates with 'Subject Alternative Name' option

> NOTE: `openssl x509` doesn't support creating certificates with SAN option well,
>       use `openssl ca` instead.

    $ openssl ca -in server.csr -config ../openssl.cnf -days 1460 -out server.crt


### Client certificates

Similar to server certificates


#### Create PKCS12 format certificates

    $ openssl pkcs12 -export -certfile ca_chain.crt -in client.crt -inkey client.key -out client.p12


## Config file example

### v3_ca.ext

    basicConstraints        = CA:TRUE
    subjectKeyIdentifier    = hash
    authorityKeyIdentifier  = keyid:always,issuer:always
    keyUsage                = cRLSign, dataEncipherment, digitalSignature, keyCertSign, keyEncipherment, nonRepudiation


### openssl.cnf

> NOTE: The 'subjectAltName' option is used to specify certificate's SAN option

    [ca]
    default_ca          = CA_default            # The default ca section

    [CA_default]
    dir                 = .                     # top dir
    database            = $dir/index.txt        # index file.
    new_certs_dir       = $dir                  # new certs dir

    certificate         = $dir/ca.crt           # The CA cert
    serial              = $dir/ca.srl           # serial no file
    private_key         = $dir/ca.key           # CA private key
    RANDFILE            = $dir/.rand            # random number file

    default_days        = 1460                  # how long to certify for
    default_crl_days    = 30                    # how long before next CRL
    default_md          = sha256                # md to use

    policy              = policy_any            # default policy
    email_in_dn         = no                    # Don't add the email into cert DN
    x509_extensions     = v3_req

    name_opt            = ca_default            # Subject name display option
    cert_opt            = ca_default            # Certificate display option
    copy_extensions     = copy                  # Copy extensions from request

    [policy_any]
    countryName            = supplied
    stateOrProvinceName    = optional
    organizationName       = optional
    organizationalUnitName = optional
    commonName             = supplied
    emailAddress           = optional


    [req]
    default_bits        = 4096
    #default_keyfile    = req.key
    #attributes         = req_attributes

    distinguished_name  = req_distinguished_name
    x509_extensions     = v3_ca
    #req_extensions     = v3_req
    default_md          = sha256

    utf8                = yes
    dirstring_type      = nobmp

    [req_distinguished_name]
    emailAddress = test@email.address
    countryName                 = Country Name (2 letter code)
    countryName_default         = CN
    countryName_min             = 2
    countryName_max             = 2

    stateOrProvinceName         = State or Province Name (full name)
    #stateOrProvinceName_default = Some-State

    localityName                = Locality Name (eg, city)

    0.organizationName          = Organization Name (eg, company)
    #0.organizationName_default  = Internet Widgits Pty Ltd

    # we can do this but it is not needed normally :-)
    #1.organizationName         = Second Organization Name (eg, company)
    #1.organizationName_default = World Wide Web Pty Ltd

    organizationalUnitName      = Organizational Unit Name (eg, section)
    #organizationalUnitName_default =

    commonName                  = Common Name (eg, YOUR name)
    commonName_max              = 64

    emailAddress                = Email Address
    emailAddress_max            = 40

    [req_attributes]
    challengePassword       = A challenge password
    challengePassword_min   = 4
    challengePassword_max   = 20


    [v3_ca]
    basicConstraints        = CA:TRUE
    subjectKeyIdentifier    = hash
    authorityKeyIdentifier  = keyid:always,issuer:always
    keyUsage                = cRLSign, dataEncipherment, digitalSignature, keyCertSign, keyEncipherment, nonRepudiation

    [v3_req]
    basicConstraints        = CA:FALSE
    subjectKeyIdentifier    = hash
    keyUsage                = digitalSignature, keyEncipherment, nonRepudiation
    subjectAltName         = @altNames


    [altNames]
    DNS.1 = example.com
    DNS.2 = *.example.com
