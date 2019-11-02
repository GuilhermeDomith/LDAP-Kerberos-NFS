# LDAP + Kerberos + NFS
Trabalho realizado para as disciplinas de Serviços e Segurança do curso de Tecnologia em Sistemas para Internet

## Introdução

Primeiro, será utilizado hostnames para referenciar às máquinas servidor e cliente, e não os seus ip's utilizados. Sendo assim, **kerberos_server** e **client_01** seram os nomes utilizados. 

O Arquivo **/etc/hostname** de cada máquina deve ter o nome localhost alterado para o seu nome correspondente e o mais importante, no aqruivo **/etc/hosts** deve ser inserido as linhas que associam o ip de cada máquina ao seu respectivo nome. Como no exemplo usado abaixo:

**Arquivo:** /etc/hosts
```conf
...
192.168.0.28    kerberos_server.jungle.kvm      kerberos_server
192.168.0.7     client_01.jungle.kvm            client_01
```


## Instalação e Configuração do Kerberos

### **Máquina**: Kerberos_Server
---

Para instalar o *Kerberos* execute o comando:
```shell
# yum install krb5-server
```
O nome de domínio utilizado para o Kerberos será **jungle.kvm**. Por isso, quando fizermos referência a uma máquina utilizaremos, por exemplo: kerberos_server.**jungle.kvm**, client_01.**jungle.kvm**, client_02.**jungle.kvm**, etc.

É preciso configurar o nome de domínio do Kerberos em três arquivos, são eles:

- krb5.conf
- kdc.conf
- kadm5.acl 

Após instalado o kerberos o nome padrão utilizado por ele é **example.com**, este nome deve ser alterado nos arquivos acima para **jungle.kvm**. Nos locais em que o nome de domínio for em caracteres maiúsculos deve-se manter desta forma. Portanto, onde se encontra **EXAMPLE.COM** altera-se para **JUNGLE.KVM**. 


No aquivo **/etc/krb5.conf**, além de precisar alterar o nome de domínio que possui muitas ocorrências, deve ser alterado também o **kdc** e **admin_server** para **kerberos_server.jungle.kvm**, que por padrão se encontram com o valor **kerberos.example.com**. Após as configurações o arquivo se deverá estar da seguinte forma:


**Arquivo:** /etc/krb5.conf
```shell
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = /etc/pki/tls/certs/ca-bundle.crt
 default_realm = JUNGLE.KVM
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 JUNGLE.KVM = {
  kdc = kerberos_server.jungle.kvm
  admin_server = kerberos_server.jungle.kvm
 }

[domain_realm]
 .jungle.kvm = JUNGLE.KVM
 jungle.kvm = JUNGLE.KVM
 ```


Nos aquivos **kdc.conf** e **kadm5.acl**, localizados em **/var/kerberos/krb5kdc/**, devem ser alterados apenas o nome de domínio, em ambos os arquivos há apenas uma ocorrência. Por fim, terão o seguinte conteúdo: 

**Arquivo:** /var/kerberos/krb5kdc/kdc.conf
```conf
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 JUNGLE.KVM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

**Arquivo:** /var/kerberos/krb5kdc/kadm5.acl
```conf
*/admin@JUNGLE.KVM      *
```

Após as configurações é preciso criar o banco de dados a ser usado pelo Kerberos. Após criá-lo executando o comando abaixo **fornceça uma senha master** para o banco.

```shell
# kdb5_util create -s -r JUNGLE.KVM
---------------------
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'JUNGLE.KVM',
master key name 'K/M@JUNGLE.KVM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key: 
Re-enter KDC database master key to verify: 
```

Habilite o Kerberos para iniciar quando o sistema for iniciado.
```shell
# systemctl enable kadmin.service
# systemctl enable krb5kdc.service
```

Reinicie a máquina ou inicie os serviços manualmente.
```shell
# systemctl start kadmin.service
# systemctl start krb5kdc.service
```

É preciso adicionar o kerberos ao firewall do sistema. Você pode ver se ele está disponível para ser adicionado com o comando abaixo. É um comando opcional, ele apenas listará todos os serviços disponíveis, mas o kerberos deve ser listado para que o próximo comando seja executado com sucesso.

```shell
# firewall-cmd --get-services | grep kerberos
```
**Não será preciso se _systemctl disable firewalld.service_
foi executado?**
```shell
# firewall-cmd --permanent --add-service kerberos
# firewall-cmd --reload
```

Vamos criar os usuários que serão usados para se autenticarem através do Kerberos. Após executar o comando **kadmin.local** será aberto um prompt, forneça os comandos como é mostrado abaixo para adicionar os usuários. Antes adicioná-los, abaixo é listado qual a necessidade de cada comando. 

> **Comandos do kadmin.local**
> - addprinc
>     - -randkey
> - ktadd
>     - -k
> - listprincs

Criando os usuários admin os hosts no Kerberos:
```shell
# kadmin.local 
---------------------
kadmin.local:  addprinc root/admin
 Enter password for...

kadmin.local:  addprinc -randkey host/client_01.jungle.kvm
 ... (output)

kadmin.local: ktadd -k /tmp/client_01.keytab host/client_01.jungle.kvm
 ... (output)

kadmin.local: listprincs
K/M@JUNGLE.KVM
host/client_01.jungle.kvm@JUNGLE.KVM
kadmin/admin@JUNGLE.KVM
kadmin/changepw@JUNGLE.KVM
kadmin/localhost@JUNGLE.KVM
kiprop/localhost@JUNGLE.KVM
krbtgt/JUNGLE.KVM@JUNGLE.KVM
root/admin@JUNGLE.KVM

kadmin.local: quit
```

Até aqui o Kerberos já está configurado. Agora é preciso configurar o Kerberos nas máquinas clientes. O arquivo **krb5.conf** terá as mesmas configurações nas máquinas clientes, por isso ele pode ser copiado para elas. E os arquivos com extenção ***.keytab**, gerados para cada máquina no comando acima, deverá ser usado pelas máquinas cliente também. 

Executando o comando abaixo os arquivos necessários serão enviados para a máquina **cliente_01**.

```shell
# scp /etc/krb5.conf /tmp/client_01.keytab root@client_01:/tmp/
```

### **Máquina**: client_01
---

Instale os pacotes necessários para que o cliente se comunique com o servidor do Kerberos.

```shell
# yum install pam_krb5 krb5-workstation
```

Copie o arquivo **krb5.conf** obtido da máquina **kerberos_server** e subistitua o arquivo já existente:

```shell
# cp /tmp/krb5.conf /etc/
```

Agora é preciso utilizar o console do **ktutil** para adicionar a chave também obtida do **kerberos_server**.

```shell
# ktutil
--------------
ktutil:  rkt /tmp/client_01.keytab 
ktutil:  wkt /etc/krb5.keytab
ktutil:  list
slot KVNO Principal
---- ---- ----------------------------------------
   1    2     host/client_01.jungle.kvm@JUNGLE.KVM
   2    2     host/client_01.jungle.kvm@JUNGLE.KVM
   ... (output)

ktutil:  quit
```

> **Para Outros Clientes**
>
> Faça as mesmas estapas acima para outros possíveis clientes que utilizarão o Kerberos. Apenas altere para o hostname que será utilizado para cada máquina.

## Instalação e configuração do OpenLDAP

### **Máquina**: kerberos_server
---

Instalar pacotes necessários.

```shell
# yum install openldap-servers openldap-clients migrationtools
```

Copiar arquivo de exemplo do ldap
```shell
# cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
```
Alterar dono do diretórios e arquivos 
```shell
# chown -R ldap: /var/lib/ldap/
```
Insira uma senha para o servidor ldap. Copie o hash da senha que será exibido no console após a senha ser fornecida.
```shell
# slappasswd
```

```shell
# cd /etc/openldap/slapd.d/cn\=config
# vim olcDatabase\=\{0\}config.ldif
```
Insira a variável **olcRootPW** no arquivo **olcDatabase={0}config.ldif** e como valor coloque o hash da sua senha que foi gerado acima. Como exemplo abaixo.

**Arquivo:** /etc/openldap/slapd.d/cn=config/olcDatabase={0}config.ldif
```conf
...
olcRootPW: {SSHA}cMDvUOc6aI0ArtrUokrHtP9IzrU7okr+TjuZ7

```

No arquivo **olcDatabase={2}hdb.ldif** altere para o nome de domínio configurado no servidor Kerberos. Altere todas as ocorrências de **dc=my-domain** e **dc=com** para **dc=jungle** e **dc=kvm**, respectivamente. Insira também o **olcRootPW** como feito anteriormente. E por último insira os dois atributos **olcAccess** como feito a

**Arquivo:** /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif
```conf
dn: olcDatabase={2}hdb
objectClass: olcDatabaseConfig
objectClass: olcHdbConfig
olcDatabase: {2}hdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=jungle,dc=kvm
olcRootDN: cn=Manager,dc=jungle,dc=kvm
olcRootPW: {SSHA}58Rggp4f6iyS6Cvn5kGrrBBUKVhjod1A
olcDbIndex: objectClass eq,pres
olcDbIndex: ou,cn,mail,surname,givenname eq,pres,sub
structuralObjectClass: olcHdbConfig
entryUUID: 50ea6406-920c-1039-95f0-1d53df8faa2e
creatorsName: cn=config
createTimestamp: 20191102223154Z
entryCSN: 20191102223154.230847Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20191102223154Z
olcAccess: {0}to attrs=userPassword by self write by dn.base="cn=Manager,dc=jungle,dc=kvm" write by anonymous auth by * none
olcAccess: {1}to * by dn.base="cn=Manager,dc=jungle,dc=kvm" write by self write by * read
```

No arquivo **olcDatabase={1}monitor.ldif** altere somente **dc=my-domain** e **dc=com** para **dc=jungle** e **dc=kvm**.

Após configurar os três arquivos acima habilite e inicie o OpenLDAP.
```shell
# systemctl enable slapd
# systemctl start slapd
```

___
**Ver se vai manter**

Da mesma forma que foi realizado com o *Kerberos*, verifique se o _OpenLDAP_ é listado como um serviço disponível no firewall **//Verificar//**
```shell
# firewall-cmd --get-services | grep ldap
```
Adicione o serviço ao firewall e reinicie-o **//Verificar//**. Ambos os comandos abaixo deverão retornar _success_ **//Verificar//**
```shell
# firewall-cmd --permanent --add-service ldap
# firewall-cmd --reload
```
___

Adicionar os arquivos ao LDAP.
```shell
# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif 

# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 

# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif 
```

Crie o arquivo que será usado pata a estrutura raiz do diretório ldap.

```shell
# vim base.ldif
```

Para configurações usadas até aqui será criado a seguinte estrutura.

```conf
dn: dc=jungle,dc=kvm
objectClass: dcObject
objectClass: organization
dc: jungle
o : jungle

dn: ou=People,dc=jungle,dc=kvm
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=jungle,dc=kvm
objectClass: organizationalUnit
ou: Group
```

Insira a estrutura raiz ao LDAP e faça uma busca para confirmar a inserção. Caso a busca retorne a estrutura criada o insersão foi realizada com sucesso. 

```shell
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f base.ldif 

# ldapsearch -x -D cn=Manager,dc=jungle,dc=kvm -W -b dc=jungle,dc=kvm
```

Crie os usuários que serão usados para se autenticar no NFS. Como o Kerberos se encarregará da autenticação, não serão criados senhas para estes usuários.

```shell
# useradd user1
# useradd user2
```

No arquivo **migrate_common.ph** altere o valor da variável **DEFAULT_MAIL_DOMAIN** para   **"jungle.kvm"**, **DEFAULT_BASE** para **"dc=jungle,dc=kvm"** e **EXTENDED_SCHEMA** para o valor **1**. Abaixo o resultado das configurações somente nas três variáveis modificadas:

**Arquivo:** /usr/share/migrationtools/migrate_common.ph
```conf
$DEFAULT_MAIL_DOMAIN = "jungle.kvm";
$DEFAULT_BASE = "dc=jungle,dc=kvm";
...
$EXTENDED_SCHEMA = 1;
```

Vamos utilizar o **migrate_passwd** e **migrate_group** para converter as informações dos usuários criados acima, e grupos criados automanticamente, na mesma estrutura do LDAP.

```shell
# egrep  user[0-9] /etc/passwd > /tmp/users
# egrep  user[0-9] /etc/group > /tmp/groups
#
# /usr/share/migrationtools/migrate_passwd.pl /tmp/users /tmp/users.ldif
# /usr/share/migrationtools/migrate_group.pl /tmp/groups /tmp/groups.ldif

```

Adicione as estruturas criadas ao LDAP.
```shell
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f /tmp/groups.ldif 
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f /tmp/users.ldif 
```
### **Máquina**: client_01
---

Instalar os pacotes necessários.

```shell
# yum install nss-pam-ldapd
```
Vamos configurar o PAM para utilizar os serviços de LDAP e Kerberos para autenticação nos clientes. Iremos configurar o endereço e o DN do Servidor LDAP. E do Kerberos iremos configurar o Reino, o KDC e o Servidor de Admin.

Após executar o comando abaixo será aberto uma interface para alterar as configurações.
```shell
# authconfig-tui
```

Marque ou altere as seguintes configurações na primeira tela que será aberta. 

```conf
[*] Utilizar LDAP
[*] Utilizar Kerberos
```

Selecione avançar.

```conf
ldap://kerberos_server.jungle.kvm/
DN base: dc=jungle,dc=kvm
```

Selecione avançar.

```conf
[*] Utilizar DNS para resolver máquinas para reinos
```

Clique em OK. E assim, o cliente foi configurado para usar o LDAP e Kerberos configurados anteriormente.

___
**Ver se isso pode influenciar**

[root@client_01 ~]# authconfig-tui

getsebool:  SELinux is disabled
___

Para ter certeza que as configurações foram aplicadas, verifique no arquivo **/etc/nsswitch.conf** que as configurações indicam o uso do LDAP para autenticação.

**Arquivo:** /etc/nsswitch.conf
```conf
...
passwd:     files sss ldap
shadow:     files sss ldap
group:      files sss ldap
...
```
Os usuários **user1** e **user2** para autenticação Kerberos foram criados apenas na máquina **kerberos_server**. Estes usuários também precisam ser criados nas máquinas cliente. Utilizando o comando **getent** os usuários salvos no LDAP podem ser facilmente criados na máquina atual.

```shel
# getent passwd user1
# id user1
-------------------
uid=1001(user1) gid=1001(user1) grupos=1001(user1)

# getent passwd user2
# id user2
-------------------
uid=1002(user2) gid=1002(user2) grupos=1002(user2)
```


> **Para Outros Clientes**
>
> Faça as mesmas estapas acima para outros possíveis clientes que utilizarão o Kerberos.