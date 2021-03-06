# Configuring ScyllaDB for SSL/TLS
This guide explains how to [generate self-signed
certificates](http://docs.scylladb.com/operating-scylla/generate_certificate/) 
to be used in ScyllaDB, cqlsh, and ValuStor.

It uses a separate certificate for every client and server.
If this is not desired, replace "server1", "server2", etc. with "server".
Do the same for "client1", etc.
Both the certificate authority and the client certificates are password protected.
This can be disabled by removing "-aes256" from the key generation commands.
This guide uses domain names, but you can replace them with IP addresses.

### Certificate Authority
The first step is to create a certificate authority (CA).
```
cat << EOF > ca-scylla.cfg
RANDFILE = NV::HOME/.rnd
[ req ]
default_bits = 4096
default_keyfile = <hostname>.key
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[ req_distinguished_name ]
C = <country code>
ST = <state>
L = <locality/city>
O = <domain>
OU = <organization, usually domain>
CN= <anything, your name or your company>
emailAddress = <email>
[v3_ca]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer:always
basicConstraints = CA:true
[v3_req]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
EOF

# Edit the certificate configuration.
vi ca-scylla.cfg

# Create the certificate authority (CA)
openssl genrsa -aes256 -out ca-scylla.key 4096
openssl req -x509 -new -nodes -key ca-scylla.key -days 36500 -config ca-scylla.cfg -out ca-scylla.pem
```

### Server Certificates
The next step is to generate server certificates. Repeat this process once for every ScyllaDB server node
(e.g. server1, server2, ...).
```
cat << EOF > scylla-server1.cfg
RANDFILE = NV::HOME/.rnd
[ req ]
default_bits = 4096
default_keyfile = server1.key
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[ req_distinguished_name ]
C = <country code>
ST = <state>
L = <locality/city>
O = <domain>
OU = <organization, usually domain>
CN= <host>.<domain> or <ip address>
emailAddress = <email>
[v3_ca]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer:always
basicConstraints = CA:true
[v3_req]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
EOF

# Edit the certificate configuration. Set CN to the host.domain or IP of the ScyllaDB server.
vi scylla-server1.cfg

# Create the server certificate.
openssl genrsa -out scylla-server1.key 4096
openssl req -new -key scylla-server1.key -out scylla-server1.csr -config scylla-server1.cfg
openssl x509 -req -in scylla-server1.csr -CA ca-scylla.pem -CAkey ca-scylla.key -CAcreateserial -out scylla-server1.crt -days 3650
```

### Client Certificates
The next step is to generate client certificates. Repeat this process once for every client
(e.g. client1, client2, ...).
```
cat << EOF > scylla-client1.cfg
RANDFILE = NV::HOME/.rnd
[ req ]
default_bits = 4096
default_keyfile = client1.key
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[ req_distinguished_name ]
C = <country code>
ST = <state>
L = <locality/city>
O = <domain>
OU = <organization, usually domain>
CN= <host>.<domain> or <ip address>
emailAddress = <email>
[v3_ca]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer:always
basicConstraints = CA:true
[v3_req]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
EOF

# Edit the certificate configuration. Set CN to the host.domain or IP of the client.
vi scylla-client1.cfg

# Create the client certificate. Protect the key with a password.
openssl genrsa -aes256 -out scylla-client1.key 4096
openssl req -new -key scylla-client1.key -out scylla-client1.csr -config scylla-client1.cfg
openssl x509 -req -in scylla-client1.csr -CA ca-scylla.pem -CAkey ca-scylla.key -CAcreateserial -out scylla-client1.crt -days 3650
```

Once all the client certificates have been created, create a master file.
This file will be used to manage client certificates.
New ones can be added to the file at any time and old ones removed as needed.
```
cat scylla-client*.crt > scylla-master-clients.crt
```

### Configuring ScyllaDB
Now that the certificates have been generated, ScyllaDB must be configured to use them.
Edit the `/etc/scylla/scylla.yaml` file on each server and make the following changes:
```
server_encryption_options:
    internode_encryption: all
    certificate: /etc/scylla/scylla-server1.crt
    keyfile: /etc/scylla/scylla-server1.key
    require_client_auth: true
    truststore: /etc/scylla/keys/ca-scylla.pem

client_encryption_options:
    enabled: true
    certificate: /etc/scylla/scylla-server1.crt
    keyfile: /etc/scylla/scylla-server1.key
    require_client_auth: true
    truststore: /etc/scylla/scylla-master-clients.crt
```
Restart the scylla server.

### Configuring cqlsh
To configure the `cqlsh` client, edit `~/.cassandra/cqlshrc` on each client.
```
[authentication]
username = myusername
password = mypassword
[cql]
version = 3.3.1
[connection]
hostname = <host>.<domain>
port = 9042
factory = cqlshlib.ssl.ssl_transport_factory
[ssl]
certfile=/etc/scylla/ca-scylla.pem
validate = true
userkey = /etc/scylla/scylla-client1.key
usercert = /etc/scylla/scylla-client1.crt
```
If you followed the instructions, you can set certificate validation to *true*.
*cqlsh* will automatically prompt you for the certificate passwords if needed.

### Configuring ValuStor
ValuStor can be configured as follows:
```
hosts = <host1>.<domain>, <host2>.<domain>
server_trusted_cert = /etc/scylla/ca-scylla.pem
server_verify_mode = 3
client_ssl_cert = /etc/scylla/keys/scylla-client1.crt
client_ssl_key = /etc/scylla/keys/scylla-client1.key
client_key_password = password
```

If you used IP addresses instead of domain names, set *server_verify_mode* to 2.
This performs IP verification instead of DNS verification.
If you disabled client key passwords, leave out the *client_key_password*.