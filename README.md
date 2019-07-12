# docker-access-remote
remote access to docker deamon

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
# delete all commnet ExecStart line from previous config loop
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
