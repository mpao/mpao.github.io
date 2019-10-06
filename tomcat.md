---
layout: default
title: Tomcat 
parent: Docker
nav_order: 10
---
# Tomcat per deployments vari

##### scarica l'immagine
```bash
docker pull tomcat
```
#####  crea container ed eseguilo sulla porta `80`
```bash
docker run -d -p 80:8080 --restart always --name webserver tomcat
```
##### connettiti alla `bash` del container
```bash
docker exec -it webserver /bin/bash
```
##### installa `vim` e edita `context.xml` come segue per permettere l'esecuzione della `GUI` per il management da qualunque indirizzo IP
```bash
apt-get update
apt-get install vim
vim webapps/manager/META-INF/context.xml
```
```html
<Context antiResourceLocking="false" privileged="true" >
  <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow=".*" />
</Context>
```
##### crea e configura un utente per la `GUI`

```bash
vim conf/tomcat-users.xml
```
```html
<role rolename="manager-gui"/>
<user username="tomcat" password="s3cret" roles="manager-gui"/>
```
