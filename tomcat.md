---
layout: default
title: Tomcat 
parent: Docker
nav_order: 10
---
# Tomcat per deployments vari

Indice
{: .no_toc .text-delta }

1. TOC
{:toc}

#### 1. scarica l'immagine
```bash
docker pull tomcat
```
####  2. crea container ed eseguilo sulla porta `80`
```bash
docker run -d -p 80:8080 --restart always --name webserver tomcat
```
#### 3. connettiti alla `bash` del container
```bash
docker exec -it webserver /bin/bash
```
#### 4. installa `vim` e edita `context.xml` come segue per permettere l'esecuzione della `GUI` per il management da qualunque indirizzo IP
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
#### 5. crea e configura un utente per la `GUI`

```bash
vim conf/tomcat-users.xml
```
```html
<role rolename="manager-gui"/>
<user username="tomcat" password="s3cret" roles="manager-gui"/>
```
