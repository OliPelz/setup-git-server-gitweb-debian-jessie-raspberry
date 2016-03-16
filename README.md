# How to set up a private Git server on Debian 8.0 (jessie) - raspberry pi (including gitweb)
Just a few notes with the exact commands to set up your own private git server on Debian 8.0 jessie (for me on a raspberry pi 1). I did this instructions because I felt the need to do as no 100% ACCURATE and correct information for this can be found online.
* It will use SSH keys for authentication.
* this is a virgin system so I will show you my exact steps:

my hostname is ```raspberry```, my FQDN is ```raspberry.skynet```
```bash
apt-get update
apt-get upgrade
apt-get install apache2 openssh-server git gitweb vim -y
```
activate cgi in order gitweb will work (cgi is disabled by default in apache2 on debian)
```bash
a2enmod cgi
```
enable apache2 for using gitweb
```bash
systemctl enable apache2
```
create local git user (remember the git password as we will need it later):
```bash
adduser git
passwd git
```
login as git
```bash
su - git
``` 
create dir / files for the authorized_keys client authentication we will use later
```bash
mkdir ~/.ssh && chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys && chmod 600 .ssh/authorized_keys
```
create location to store all our future repositories
```bash
mkdir ~/repos/
```
create our first repository called ```my_first_repo```(change accordingly) and initalize as a git repos
```bash
mkdir ~/repos/my_first_repo
cd $_
git --bare init
git update-server-info
logout
```
configure gitweb to use our new base repositories path, as root open the file 
```/etc/gitweb.conf``` , find the line that starts with ```$projectroot = ```
edit it to point it to our git user's repos path so it looks like:
```$projectroot = "/home/git/repos";```
save and close the file, next start apache2:
```bash
systemctl start apache2
```
Now on any client in the same network browse to the following url to browse gitweb
which should give you an overview with the ```my_first_repo``` listing (or whatever you called your first repos).
use any modern web browser ```http://raspberry.skynet/gitweb``` 
Now for any client in the same network on the commandline to access our git server we will use SSH keys for authentication. You can do this without setting a SSH key password which will make your git server access very convenient, but using a password is recommended as it adds another layer of security. Use the following command replacing ```youremail@mailprovider.com``` with your email:
```bash
ssh-keygen -C "youremail@mailprovider.com"
```
Finally add our newly generated SSH key for authentication to the git user on the git server. This will be used for authentication:
```bash
cat ~/.ssh/id_rsa.pub | ssh git@raspberry.skynet "cat >> ~/.ssh/authorized_keys"
```
Now on any client in the same network to clone the new remote repos do (enter the git password you defined in a former step):
```bash
git clone ssh://git@raspberry.skynet/home/git/repos/my_first_repo
cd my_first_repo
touch HELLO_WORLD.TXT
git add HELLO_WORLD.TXT
git commit -a -m "adding first file to the project"
git push origin 
```
Finally refresh http://raspberry.skynet/gitweb to test if the new file has been uploaded correctly!


In summary now if you want to create a new repository on your git server, log in on the git server as user git, then if your new repo is called 
```tha_new_repoz_proj``` do:
```bash
mkdir ~/repos/tha_new_repoz_proj
cd $_
git --bare init
git update-server-info
logout
```
On any client in this network you then can clone this new repository using:
```bash
git clone ssh://git@raspberry.skynet/home/git/repos/tha_new_repoz_proj
```
To add new clients to SSH key authenticate to your new Git server use:
```bash
ssh-keygen -C "youremail@mailprovider.com"
cat ~/.ssh/id_rsa.pub | ssh git@raspberry.skynet "cat >> ~/.ssh/authorized_keys"
```

# Update: protect your local gitweb web site with basic user authentication 
open the gitweb apache2 config after we made a backup of it first:
```bash
cp /etc/apache2/conf-available/gitweb.conf /etc/apache2/conf-available/gitweb.conf.BAK
vi /etc/apache2/conf-available/gitweb.conf
```

edit the file so in the end it will look like:
```bash
<IfModule mod_alias.c>
  <IfModule mod_cgi.c>
    Define ENABLE_GITWEB
  </IfModule>
  <IfModule mod_cgid.c>
    Define ENABLE_GITWEB
  </IfModule>
</IfModule>

<IfDefine ENABLE_GITWEB>
  Alias /gitweb /usr/share/gitweb

  <Directory /usr/share/gitweb>
    AuthType Basic
    AuthName "Gitweb protected web site"
    AuthBasicProvider file
    AuthUserFile /usr/share/gitweb/.htpasswd
    Require valid-user
    Options +FollowSymLinks +ExecCGI
    AddHandler cgi-script .cgi
  </Directory>
</IfDefine>

```
Now add a new authorization user, we will use git for our purpose (you can use any username you would like):
```bash
htpasswd -c /usr/share/gitweb/.htpasswd git
```
Finally restart apache2, then browse to gitweb using your browser, you will then be greeted by our new password prompt
```bash
service apache2 restart
```
