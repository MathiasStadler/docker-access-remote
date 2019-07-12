# docker-access-remote remote access to docker daemon

## create script from readme.md

```bash
SCRIPT_NAME="readme2script.sh"
rm -rf ${SCRIPT_NAME}
cat << EOF |tee ${SCRIPT_NAME}
#!/bin/bash
set -o posix -o errexit
if [ -z \${1+x} ]; then printf "set script name that you would extract \n "; exit 1;fi
if [ -z \${2+x} ]; then printf "set source markdown file\n" ;exit 1; fi
if grep -q "\$1" "\$2"; then printf "" ; else printf "NOT exists\n"; exit 1 ; fi ;
unset SCRIPT
unset SCRIPT_PATH
SCRIPT=\$1;
SCRIPT_PATH="work_folder/\${SCRIPT}"
# printf "SCRIPT => %s \n" "\${SCRIPT}"
expr="/^\\\`\\\`\\\`bash \${SCRIPT}/,/^\\\`\\\`\\\`/{ /^\\\`\\\`\\\`bash.*$/d; /^\\\`\\\`\\\`$/d; p; }"
# printf "Expression %s \n" "\${expr}"
if [ -f "\${1}" ]; then printf "File exists please delete first \n";exit 1;fi;
sed -n "\${expr}" "\${2}" >"\${1}"
chmod +x "\${1}"
EOF
chmod +x ${SCRIPT_NAME}
```

## enable remote access

- from https://success.docker.com/article/how-do-i-enable-the-remote-api-for-dockerd

```bash enable-remote-access.sh
#!/bin/bash
#set flag
set -o errexit -o posix
# service
SERVICE="docker.service";
# determine service file
SERVICE_FILE="$(systemctl show -p FragmentPath ${SERVICE}|sed 's/.*=//')";
# create backup
sudo cp "${SERVICE_FILE}" "${SERVICE_FILE}.backup"
# delete all comment ExecStart line from previous config loop
sudo sed -i '/^#ExecStart.*$/ d' "${SERVICE_FILE}"
# comment available ExecStart (all match)
sudo sed -i '/^ExecStart/ s/^#*/#/g' "${SERVICE_FILE}"
# add remote network setting
sudo sed -i '/^#ExecStart/a ExecStart=/usr/bin/dockerd -H fd:// -H 0.0.0.0:2376' "${SERVICE_FILE}"
# Reload the unit files
sudo systemctl daemon-reload
# restart
sudo systemctl restart docker.service
```

## test-remote-access.sh

```bash test-remote-access.sh
#!/bin/bash
# flag
set -o posix -o errexit
# determine local ip
ip="$(ip route get 1 | sed 's/^.*src \([^ ]*\).*$/\1/;q')";
alias dockerx="docker -H="\${ip}":2376"
dockerx info
if [ $? -ne 0 ]
   then
      echo "Error can't connect remote docker"
    fi
```

## tls remote access to docker daemon

- from https://docs.docker.com/engine/security/https/

```bash tls-enable-access.sh
#!/bin/bash
set -o errexit -o posix
# create passphrase text file
echo "topSecret" >passphrase.txt
# create generate CA private
openssl genrsa -aes256 -passout  -out docker-ca-key.pem 4096
# remove key
# from https://www.jamescoyle.net/how-to/1073-bash-script-to-create-an-ssl-certificate-key-and-request-csr
openssl rsa -in docker-ca-key.pem -passin file:passphrase.txt -out docker-ca-key.pem
# create ca
openssl req -new -x509 -days 30 -key docker-ca-key.pem -sha256 -out docker-ca.pem -passout file:passphrase.txt\
  -subj "/C=US/ST=Test/L=for/O=Test/CN=localhost"
# create server key
openssl genrsa -out docker-server-key.pem 4096
# sign the server key
openssl req -subj "/CN=$HOST" -sha256 -new -key docker-server-key.pem -out docker-server.csr
# prepare file
ip="$(ip route get 1 | sed 's/^.*src \([^ ]*\).*$/\1/;q')";
echo subjectAltName = DNS:$HOST,IP:${ip},IP:127.0.0.1 >> docker-extfile.cnf
echo extendedKeyUsage = serverAuth >> docker-extfile.cnf
# create cert
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey docker-ca-key.pem \
-CAcreateserial -out docker-server-cert.pem -extfile docker-extfile.cnf
# create client key
openssl genrsa -out docker-key.pem 4096
# create XXX
openssl req -subj '/CN=client' -new -key docker-key.pem -out docker-client.csr
# write extfile-client.cnf
echo extendedKeyUsage = clientAuth > docker-extfile-client.cnf
# generate CA Private Key
openssl x509 -req -days 365 -sha256 -in docker-client.csr -CA docker-ca.pem -CAkey docker-ca-key.pem \
-CAcreateserial -out docker-cert.pem -extfile docker-extfile-client.cnf
# for security purpose delete files
rm -v docker-client.csr docker-server.csr docker-extfile.cnf docker-extfile-client.cnf
# set permissions
chmod -v 0400 docker-ca-key.pem docker-key.pem docker-server-key.pem
chmod -v 0444 docker-ca.pem docker-server-cert.pem docker-cert.pem
# copy to /etc/ssl
cp -v docker-ca-key.pem docker-key.pem docker-server-key.pem docker-ca.pem docker-server-cert.pem docker-cert.pem /etc/ssl
# server side
dockerd --tlsverify --tlscacert=/etc/ssl/docker-ca.pem --tlscert=/etc/ssl/docker-server-cert.pem --tlskey=/etc/ssl/docker-server-key.pem -H=0.0.0.0:2376
# modify config
# service
SERVICE="docker.service";
# determine service file
SERVICE_FILE="$(systemctl show -p FragmentPath ${SERVICE}|sed 's/.*=//')";
# create backup
sudo cp "${SERVICE_FILE}" "${SERVICE_FILE}.backup"
# delete all comment ExecStart line from previous config loop
sudo sed -i '/^#ExecStart.*$/ d' "${SERVICE_FILE}"
# comment available ExecStart (all match)
sudo sed -i '/^ExecStart/ s/^#*/#/g' "${SERVICE_FILE}"
# add remote network setting
sudo sed -i '/^#ExecStart/a --tlsverify --tlscacert=/etc/ssl/docker-ca.pem --tlscert=/etc/ssl/docker-server-cert.pem --tlskey=/etc/ssl/docker-server-key.pem -H=0.0.0.0:2376' "${SERVICE_FILE}"
# TODO old sudo sed -i '/^#ExecStart/a ExecStart=/usr/bin/dockerd -H fd:// -H 0.0.0.0:2376' "${SERVICE_FILE}"
# Reload the unit files
sudo systemctl daemon-reload
# restart
sudo systemctl restart docker.service


# test from client side
docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem \
  -H=$HOST:2376 version



# DON'T FORGET THE FIREWALL  

```