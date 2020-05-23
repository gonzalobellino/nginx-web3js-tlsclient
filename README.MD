# Using NGINX with GETH service JSON-RPC + TLS client auth

## NGINX create config nginx.conf

~~~
events { }   

http {
    server {                          
        listen      443 ssl;
        server_name localhost;

        ssl_certificate        /certs/server.crt;
        ssl_certificate_key    /certs/server.key;
        ssl_client_certificate /certs/ca.crt;
        ssl_verify_client      on;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers   HIGH:!aNULL:!MD5;
        access_log /logs/access.log;
        error_log /logs/error.log debug; 

        location / {
            proxy_pass http://localhost:8545;
        }
    }
}
~~~

## TLS 

### Create the file openssl_ext.conf

~~~
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = AR
ST = BA
O = DUMMY_ORG
OU = ETH
CN = localhost
emailAddress = email@email.com

[server_extensions]
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth

[client_extensions]
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage=clientAuth
~~~

### Create CA dummy

~~~
openssl genrsa -des3 -out ./certs/ca.key 2048
openssl req -x509 -new -nodes -key ./certs/ca.key -sha256 -days 1825 -out ./certs/ca.crt -config ./certs/openssl_ext.conf
~~~

## Create Server key/crt

~~~
openssl genrsa  -out ./certs/server.key 
openssl req -new -key ./certs/server.key -out ./certs/server.csr -config ./certs/openssl_ext.conf

openssl x509 -req -extensions server_extensions -extfile ./certs/openssl_ext.conf -in ./certs/server.csr -CA ./certs/ca.crt -CAkey ./certs/ca.key -CAcreateserial -out ./certs/server.crt -days 3650
~~~

## Create Client key/cert

~~~
openssl genrsa  -out ./certs/client.key 
openssl req -new -key ./certs/client.key -out ./certs/client.csr -config ./certs/openssl_ext.conf

openssl x509 -req -extensions client_extensions -extfile ./certs/openssl_ext.conf -in ./certs/client.csr -CA ./certs/ca.crt -CAkey ./certs/ca.key -CAcreateserial -out ./certs/client.crt -days 3650
~~~


## NGINX docker start (dev: for easy test use network host when running geth on localhost)

~~~javascript
docker run -d --rm --name nginx-geth --network host -v $PWD/logs/:/logs  -v $PWD/certs:/certs -v $PWD/nginx.conf:/etc/nginx/nginx.conf nginx  
~~~


## Test usage with CURL (use an updated version, older versions dont work with TLS AUTH CLIENT)

~~~
curl --cert ./certs/client.crt  --key ./certs/client.key --cacert ./certs/ca.crt -s -X POST  -H "Content-type:application/json" --data '{"jsonrpc":"2.0","method":"eth_gasPrice","id":1}'  https://localhost | jq .
~~~


## Test using nodejs

### NPM
~~~
npm init
npm install web3
npm install web3-providers-http
~~~

### Create new file connect.js

~~~
var Web3HttpProvider = require('web3-providers-http');
var fs = require('fs');
var https = require('https');
Web3 = require("web3");

web3 = new Web3();

var options = {
    keepAlive: true,
    timeout: 20000, // milliseconds,
    headers: [{name: 'Access-Control-Allow-Origin', value: '*'}],
    withCredentials: false,
    agent: {https: https.Agent({  
        method: 'POST',
        rejectUnauthorized: true,
        // Necessary only if the server uses a self-signed certificate.
        ca: [fs.readFileSync("./certs/ca.crt")],
        // Necessary only if the server requires client certificate authentication.
        key: fs.readFileSync("./certs/client.key"),
        cert: fs.readFileSync("./certs/client.crt")
      })}
};

web3.setProvider(new Web3HttpProvider(process.env.GETH_RPC_URL, options))

web3.eth.getBalance(process.env.WALLET_ADDRESS).then(console.log)
web3.eth.getBlock( 'latest', true, function(error, result) {
    console.log(result) ;   
});
~~~

### Run test 

*Replace <HOSTNAME>, <WALLET_ADDRESS>

~~~
WALLET_ADDRESS=<WALLET_ADDRESS> GETH_RPC_URL=https://<HOSTNAME> node connect.js
~~~

### More tunning info

[https://nodejs.org/api/tls.html#tls_tls_connect_options_callback](https://nodejs.org/api/tls.html#tls_tls_connect_options_callback)