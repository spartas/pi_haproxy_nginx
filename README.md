nginx running in a docker container behind haproxy. I run this on a 
Raspberry Pi. haproxy is used to check client requests (including 
client certificates) and route and blackhole traffic.

This is not a comprehensive guide. I have replaced my production hostname 
with `core.example.net` in this public repository.


## Prerequisites: Docker, docker-compose, haproxy
```
# apt install docker-ce haproxy libffi-dev libssl-dev python3{,-dev,-pip}
# apt install docker-ce haproxy
# pip3 install docker-compose
```

## CAs, Keys, and Certificaes
### Generating the self-signed cert
This is a dummy cert that's presented to clients that make requests on
invalid hostnames. `sswc` = self-signed wildcard certificate

```
openssl genrsa 2048 > sswc.key
openssl req -new -x509 -nodes -days 36500\
    -key sswc.key \
    -subj "/CN=*.example.com" \
    -config <(cat /etc/ssl/openssl.cnf) \
    -out sswc.cert
openssl x509 -noout -fingerprint -text < sswc.cert > sswc.info
cat sswc.cert sswc.info > sswc.pem
```
Source: <https://superuser.com/questions/1374959/self-signed-wildcard-certificate>

## Generating client certificates
Valid client certificates are required to access our internal data

### Make sure our CA has a valid key and certificate
```
openssl genrsa -des3 -out core_ca.key 4096
openssl req -new -x509 -days 365 -key core_ca.key -out core_ca.crt
```

### Generate the client certificate
```
openssl genrsa -des3 -out user.key 4096
openssl req -new -key user.key -out user.csr
openssl x509 -req -days 365 -in user.csr -CA core_ca.crt -CAkey core_ca.key -set_serial 01 -out user.crt
openssl pkcs12 -export -out user.pfx -inkey user.key -in user.crt -certfile core_ca.crt
```

I use letsencrypt to generate the webserver SSL certificate.


### Start and test
```
systemctl enable --now haproxy
docker-compose up -d -f /root/core/docker-compose.yml core
```
