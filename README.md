# LDAP + Kerberos + NFS
Trabalho realizado para as disciplinas de Serviços e Segurança do curso de Tecnologia em Sistemas para Internet

## Instalar e configurar Kerberos

### Cliente 01 (Servidor)
---

Para instalar o *Kerberos* digite o seguinte comando:
```shell
# yum install krb5-server
```

O arquivo de configurações padrão do *Kerberos* pode ser encontrado em:

```shell
# vim /etc/krb5.conf
```

Utilizando o arquivo de configuração inicial do *Kerberos*, substitua todas as ocorrências de example.com e EXAMPLE.COM por jungle.kvm e JUNGLE.KVM, respectivamente. O nosso &&&&& será chamado de **jungle.kvm** e o &&&&&2 de **JUNGLE.KVM**. Para o &&&&& será dado o nome **cenvm01**, ele deve ser configurado nas informações do &&&&& *[realms]*.

> Os nomes fornecidos para as configurações acima não necessariamente precisam ser os mesmos escolhidos, mas precisam ser iguais em todas as referências ocorridas no arquivo *krb5.conf* e nos arquivos que serão alterados abaixo.

Com as configurações acima citadas o arquivo *krb5.conf* deverá ter o seguinte resultado:

```conf
FILE: /etc/krb5.conf

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
  kdc = cenvm01.jungle.kvm
  admin_server = cenvm01.jungle.kvm
 }

[domain_realm]
 .jungle.kvm = JUNGLE.KVM
 jungle.kvm = JUNGLE.KVM
```

Para fazer com que &&&& é preciso configurar o arquivo *kdc.conf* e o *kadm5.acl*:

```shell
# vim /var/kerberos/krb5kdc/kdc.conf
# vim /var/kerberos/krb5kdc/kadm5.acl 
```

Neles precisamos apenas referênciar o &&&&&2 configurado acima. Caso tenha escolhido outro nome você deve fornecê-lo. Substitua o nome `EXAMPLE.COM`, configurado em ambos os arquivos, para **JUNGLE.KVM**.

Então, é preciso criar o banco de dados a ser usado pelo *Kerberos*. Após criá-lo, com o comando abaixo, será solicitado uma senha para o banco.

```shell
# kdb5_util create -s -r JUNGLE.KVM
```

Habilite os serviços do *Kerberos* para iniciarem quando o sistema for iniciado:
```shell
# systemctl enable kadmin.service
# systemctl enable krb5kdc.service
```

Reinicie a máquina ou inicie os serviços manualmente:
```shell
# systemctl start kadmin.service
# systemctl start krb5kdc.service
```

Verifique se o *Kerberos* é listado como um serviço disponível no firewall **//Verificar//**
```shell
# firewall-cmd --get-services | grep kerberos
```
Adicione o serviço ao firewall e reinicie-o **//Verificar//**. Ambos os comandos abaixo deverão retornar _success_ **//Verificar//**
```shell
# firewall-cmd --permanent --add-service kerberos
# firewall-cmd --reload
```

Vamos criar os usuários para acessarem o kerberos. Após digitar o comando abaixo será aberto um prompt. Forneça o comando `addprinc root/admin` para criar o usuário _admin_


```shell
# kadmin.local 
---
kadmin.local:  addprinc root/admin
 Enter password for...

kadmin.local:  addprinc -randkey host/cenvm02.jungle.kvm
 ... (output)

kadmin.local:  addprinc -randkey host/cenvm03.jungle.kvm
 ... (output)

kadmin.local: ktadd -k /tmp/cenvm02.keytab host/cenvm02.jungle.kvm
 ... (output)

kadmin.local: ktadd -k /tmp/cenvm03.keytab host/cenvm03.jungle.kvm
 ... (output)

kadmin.local: listprincs
 K/M@JUNGLE.KVM
 host/cenvm02.jungle.kvm@JUNGLE.KVM
 host/cenvm03.jungle.kvm@JUNGLE.KVM
 kadmin/admin@JUNGLE.KVM
 kadmin/changepw@JUNGLE.KVM
 kadmin/localhost@JUNGLE.KVM
 kiprop/localhost@JUNGLE.KVM
 krbtgt/JUNGLE.KVM@JUNGLE.KVM
 root/admin@JUNGLE.KVM

kadmin.local: quit
```
> Caso a versão do _Kerberos_ utilizada seja a versão 5 e os usuários e hosts adicionados sejam os mesmos acima, a saída ao digitar o comando `listprincs` deverá ser a mesma da que foi exibida. Caso contrário, a saída pode ser diferente.

> ## Comandos do Kadmin.local
> addprinc - 
>
> -randkey - gerar chave aleatória
> 
> ktadd -k
> 
> listprincs

Enviar os arquivos necessários para &&&&&&

```shell
# scp /etc/krb5.conf /tmp/cenvm02.keytab cenvm02:/tmp/
```

### Cliente 02 
---

Instale os pacotes necessários para que o cliente se comunique com o servidor do _kerberos_.
```shell
# yum install pam_krb5 krb5-workstation
```

Copie o arquivo obtido do servidor kerberos e subistitua o arquivo de configurações da maquina local:

```shell
# cp /tmp/krb5.conf /etc/
```

Abra o console do kutil para &&&&&
```shell
# ktutil
---
ktutil:  rkt /tmp/cenvm02.keytab 
ktutil:  wkt /etc/krb5.keytab
ktutil:  list
slot KVNO Principal
   1    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   2    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   3    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   4    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   5    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   6    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   7    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
   8    2       host/cenvm02.jungle.kvm@JUNGLE.KVM
ktutil:  quit
```

> ## Comandos do Kutil
> rkt
>
> wkt
>
> list

> Faça as mesmas estapas para outros possíveis clientes que utilizaram os serviços configurados.

## Instalar e configurar OpenLDAP

### Cliente 01 (Servidor)
---

Instalar pacotes necessários

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
Insira uma senha para o servidor ldap. Copie o hash da senha que será exibido no console.
```shell
# slappasswd
```

```shell
# cd /etc/openldap/slapd.d/cn\=config
# vim olcDatabase\=\{0\}config.ldif
```
Insira a variável **olcRootPW** do final do arquivo e como valor coloque o hash da sua senha copiada acima. 
```conf
FILE: olcDatabase\=\{0\}config.ldif

...
olcRootPW: {SSHA}cMDvUOc6aI0ArtrUokrHtP9IzrU7okr+TjuZ7

```

Altere as referencias de domínio para o seu nome configurado na instalação do _Kerberos_. 

```shell
# vim olcDatabase={2}hdb.ldif
```

Todas as referências abaixo devem ser trocadas

**my-domain**: jungle  
**dc=com**: dc=kvm

Utilize o recurso de replace do _vim_ para evitar erros nas alterações. Insira tambpem o **olcRootPW**, como feito anteriormente.

```conf
FILE: vim olcDatabase={2}hdb.ldif

...
olcRootPW: {SSHA}cMDvUOc6aI0ArtrUokrHtP9IzrU7okr+TjuZ7

olcAccess: {0}to attrs=userPassword by self write by dn.base="cn=Manager,dc=jungle,dc=kvm" write by anonymous auth by * none

olcAccess: {1}to * by dn.base="cn=Manager,dc=jungle,dc=kvm" write by self write 
by * read

```

Em **monitor.ldif** altere somente as referências **my-domain** e **com**  para os mesmos valores colocados anteriormente.

```shell
# vim olcDatabase\=\{1\}monitor.ldif
```

Após isso, habilite e inicie o serviço do _OpenLDAP_.
```shell
# systemctl enable slapd
# systemctl start slapd
```

Da mesma forma que foi realizado com o *Kerberos*, verifique se o _OpenLDAP_ é listado como um serviço disponível no firewall **//Verificar//**
```shell
# firewall-cmd --get-services | grep ldap
```
Adicione o serviço ao firewall e reinicie-o **//Verificar//**. Ambos os comandos abaixo deverão retornar _success_ **//Verificar//**
```shell
# firewall-cmd --permanent --add-service ldap
# firewall-cmd --reload
```
Adicionar os arquivos ao _OpenLDAP_
```shell
# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif 

# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 

# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif 
```

Cria a estrutura raiz do diretório ldap.

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

Insira a estrutura raiz ao ldap e faça uma busca para confirmar a inserção. Caso a busca retorne a estrutura criada o insersão foi realizada com sucesso. 

```shell
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f base.ldif 

# ldapsearch -x -D cn=Manager,dc=jungle,dc=kvm -W -b dc=jungle,dc=kvm
```

Crie os usuários que serão usados para se autenticar no _NFS_. Como o _Kerberos_ se encarregará da autenticação não serão criados senhas para estes usuários.

```shell
# useradd demouser1
# useradd demouser2
```

```shell
# cd /usr/share/migrationtools/
# vim migrate_common.ph
```

Altere as seguintes configurações

```conf
# Default DNS domain
$DEFAULT_MAIL_DOMAIN = "padl.com";

# Default base 
$DEFAULT_BASE = "dc=padl,dc=com";

...

# such as person.
$EXTENDED_SCHEMA = 0;
```
VVV
```conf
# Default DNS domain
$DEFAULT_MAIL_DOMAIN = "jungle.kvm";

# Default base 
$DEFAULT_BASE = "dc=jungle,dc=kvm";

...

# such as person.
$EXTENDED_SCHEMA = 1;
```

Vamos utilizar o **migrate_passwd** e **migrate_group** para converter as informações dos usuários, criados acima, na estrutura do _LDAP_. Após criadas as estruturas de **users** e **groups** adicione ao _LDAP_.

```shell
# grep demo /etc/passwd > /tmp/users
# grep demo /etc/group > /tmp/groups
#
#  ./migrate_passwd.pl /tmp/users /tmp/users.ldif
#  ./migrate_group.pl /tmp/groups /tmp/groups.ldif
#
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f /tmp/groups.ldif 
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f /tmp/users.ldif 
```

### Cliente 02 
---

```shell
# yum install nss-pam-ldapd
```
Vamos configurar o _PAM_ para utilizar os serviços de _LDAP_ e _Kerberos_ para autenticação nos clientes. Iremos configurar o endereço do Servidor _LDAP_ e o DN base do mesmo. E do Kerberos iremos configurar o Reino, o KDC e o Servidor de Admin.

```shell
# authconfig-tui
```

Marque ou altere as seguintes configurações na tela que será aberta. 

```conf
[*] Utilizar LDAP
[*] Utilizar Kerberos
```

Clique em avançar

```conf
ldap://cenvm01.jungle.kvm/
DN base: dc=jungle,dc=kvm
```

Clique em avançar

```conf
[*] Utilizar DNS para resolver máquinas para reinos
```

Clique em OK. E assim, o cliente foi configurado para usar o _LDAP_ e _Kerberos_ configurados anteriormente.

Verifique no arquivo **/etc/nsswitch.conf** que as configurações indicam o uso do _LDAP_ para autenticação.

```conf
FILE: /etc/nsswitch.conf

...
passwd:     files sss ldap
shadow:     files sss ldap
group:      files sss ldap
...
```

NAO FUNCIONOU
```shel
# getent passwd demouser1
# id demouser1

# getent passwd demouser2
# id demouser2
```


> Faça as mesmas estapas para outros possíveis clientes que utilizaram os serviços configurados.