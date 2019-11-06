# LDAP + Kerberos + NFS

Trabalho realizado para as disciplinas de Serviços e Segurança do curso de Tecnologia em Sistemas para Internet

1. [Configurações Iniciais](./#1)
2. [Instalação e Configuração do Kerberos](./#2)
   * [kerberos\_server](./#2-1)
   * [client\_01](./#2-2)
3. [Instalação e Configuração do OpenLDAP](./#3)
   * [kerberos\_server](./#3-1)
   * [client\_01](./#3-2)
4. [Instalação e Configuração do NFS](./#4)
   * [kerberos\_server](./#4-1)
   * [client\_01](./#4-2)

## **1. Configurações Iniciais** <a id="1"></a>

Será utilizado hostnames para referenciar às máquinas servidor e cliente, e não os seus ip's utilizados. Sendo assim, **kerberos\_server** e **client\_01** seram os nomes utilizados.

O Arquivo **/etc/hostname** de cada máquina deve ter o nome localhost alterado para o seu nome correspondente e, o mais importante, no aqruivo **/etc/hosts** deve ser inserido as linhas que associam o ip de cada máquina ao seu respectivo nome. Como no exemplo usado abaixo:

**Arquivo:** /etc/hosts

```text
...
192.168.0.28    kerberos_server.jungle.kvm      kerberos_server
192.168.0.7     client_01.jungle.kvm            client_01
```

## **2. Instalação e Configuração do Kerberos** <a id="2"></a>

## **`Máquina Kerberos_Server`**

 Para instalar o \*Kerberos\* execute o comando:

```text
# yum install krb5-server
```

O nome de domínio utilizado para o Kerberos será **jungle.kvm**. Por isso, quando fizermos referência a uma máquina utilizaremos, por exemplo: kerberos\_server.\*\*jungle.kvm\*\*, client\_01.\*\*jungle.kvm\*\*, client\_02.\*\*jungle.kvm\*\*, etc.

 É preciso configurar o nome de domínio do Kerberos em três arquivos, são eles: 

* krb5.conf
* kdc.conf
* kadm5.acl
* 
 Após instalado o kerberos o nome padrão utilizado por ele é \*\*example.com\*\*, este nome deve ser alterado nos arquivos acima para \*\*jungle.kvm\*\*. Nos locais em que o nome de domínio for em caracteres maiúsculos deve-se manter desta forma. Portanto, onde se encontra \*\*EXAMPLE.COM\*\* altera-se para \*\*JUNGLE.KVM\*\*. No aquivo \*\*/etc/krb5.conf\*\*, além de precisar alterar o nome de domínio que possui muitas ocorrências, deve ser alterado também o \*\*kdc\*\* e \*\*admin\_server\*\* para \*\*kerberos\_server.jungle.kvm\*\*, que por padrão se encontram com o valor \*\*kerberos.example.com\*\*. Após as configurações o arquivo se deverá estar da seguinte forma: \*\*Arquivo:\*\* /etc/krb5.conf \`\`\`shell includedir /etc/krb5.conf.d/ \[logging\] default = FILE:/var/log/krb5libs.log kdc = FILE:/var/log/krb5kdc.log admin\_server = FILE:/var/log/kadmind.log \[libdefaults\] dns\_lookup\_realm = false ticket\_lifetime = 24h renew\_lifetime = 7d forwardable = true rdns = false pkinit\_anchors = /etc/pki/tls/certs/ca-bundle.crt default\_realm = JUNGLE.KVM default\_ccache\_name = KEYRING:persistent:%{uid} \[realms\] JUNGLE.KVM = { kdc = kerberos\_server.jungle.kvm admin\_server = kerberos\_server.jungle.kvm } \[domain\_realm\] .jungle.kvm = JUNGLE.KVM jungle.kvm = JUNGLE.KVM \`\`\` Nos aquivos \*\*kdc.conf\*\* e \*\*kadm5.acl\*\*, localizados em \*\*/var/kerberos/krb5kdc/\*\*, devem ser alterados apenas o nome de domínio, em ambos os arquivos há apenas uma ocorrência. Por fim, terão o seguinte conteúdo:

 \*\*Arquivo:\*\* /var/kerberos/krb5kdc/kdc.conf 

{% code-tabs %}
{% code-tabs-item title="kdc.conf" %}
```bash
[kdcdefaults] kdc_ports = 88 
kdc_tcp_ports = 88 
[realms] JUNGLE.KVM = { 
#master_key_type = aes256-cts
 acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal }

```
{% endcode-tabs-item %}
{% endcode-tabs %}

 \*\*Arquivo:\*\* /var/kerberos/krb5kdc/kadm5.acl \`\`\`conf \*/admin@JUNGLE.KVM \* \`\`\` Após as configurações é preciso criar o banco de dados a ser usado pelo Kerberos. Após criá-lo executando o comando abaixo \*\*fornceça uma senha master\*\* para o banco. \`\`\`shell \# kdb5\_util create -s -r JUNGLE.KVM --------------------- Loading random data Initializing database '/var/kerberos/krb5kdc/principal' for realm 'JUNGLE.KVM', master key name 'K/M@JUNGLE.KVM' You will be prompted for the database Master Password. It is important that you NOT FORGET this password. Enter KDC database master key: Re-enter KDC database master key to verify: \`\`\` Habilite o Kerberos para iniciar quando o sistema for iniciado. \`\`\`shell \# systemctl enable kadmin.service \# systemctl enable krb5kdc.service \`\`\` Reinicie a máquina ou inicie os serviços manualmente. \`\`\`shell \# systemctl start kadmin.service \# systemctl start krb5kdc.service \`\`\`  
 \*\*\` Firewall Habilitado \`\*\* Se o serviço firewalld estiver habilitado, adicione o Kerberos e reinicie o serviço. \`\`\`shell \# firewall-cmd --permanent --add-service kerberos -------------------- success \# firewall-cmd --reload -------------------- success \`\`\` ---  


Vamos criar os usuários e hosts que serão usados para se autenticarem através do Kerberos. Após executar o comando **kadmin.local** será aberto um prompt, forneça os comandos como é mostrado abaixo para adicionar os usuários. Quando adicionado usuários será solicitado a criação de uma senha.

```text
# kadmin.local 
---------------------
kadmin.local: addprinc root/admin
kadmin.local: addprinc user1
kadmin.local: addprinc user2
kadmin.local: addprinc -randkey host/client_01.jungle.kvm
kadmin.local: ktadd -k /tmp/client_01.keytab host/client_01.jungle.kvm
kadmin.local: listprincs
   K/M@JUNGLE.KVM
   host/client_01.jungle.kvm@JUNGLE.KVM
   kadmin/admin@JUNGLE.KVM
   kadmin/changepw@JUNGLE.KVM
   kadmin/localhost@JUNGLE.KVM
   kiprop/localhost@JUNGLE.KVM
   krbtgt/JUNGLE.KVM@JUNGLE.KVM
   root/admin@JUNGLE.KVM
   user1@JUNGLE.KVM
   user2@JUNGLE.KVM
kadmin.local: quit
```

**`Comandos do kadmin.local`**

* addprinc
  * -randkey
* ktadd - Gera o arquivo .keytab do host cadastrado.
  * -k
* listprincs

Agora é preciso configurar o Kerberos nos hosts cliente. O arquivo **krb5.conf** terá as mesmas configurações nas máquinas cliente, por isso ele será copiado a partir delas.

Os arquivos com extenção **.keytab**, que será gerado para cada host adicionado no **kadmin.local**, será usado pelas máquinas cliente também. No exemplo acima ele foi gerado no diretório **/tmp**.

## **`Máquina client_01`**

 Instale os pacotes necessários para que o cliente se comunique com o servidor do Kerberos. \`\`\`shell \# yum install pam\_krb5 krb5-workstation \`\`\` Copie o arquivo \*\*krb5.conf\*\* da máquina \*\*kerberos\_server\*\* e subistitua o arquivo já existente. O arquivo \*\*client\_01.keytab\*\* deve ser copiado para a qualquer local da máquina atual, ele será usado posteriormente. \`\`\`shell \# scp root@kerberos\_server:/etc/krb5.conf /etc/ \# scp root@kerberos\_server:/tmp/client\_01.keytab /tmp/ \`\`\` Agora é preciso utilizar o console do \*\*ktutil\*\* para adicionar a chave também obtida do \*\*kerberos\_server\*\*. \`\`\`shell \# ktutil -------------- ktutil: rkt /tmp/client\_01.keytab ktutil: wkt /etc/krb5.keytab ktutil: list slot KVNO Principal ---- ---- ---------------------------------------- 1 2 host/client\_01.jungle.kvm@JUNGLE.KVM 2 2 host/client\_01.jungle.kvm@JUNGLE.KVM ... \(output\) ktutil: quit \`\`\` Faça as mesmas estapas acima para outros possíveis clientes que utilizarão o Kerberos. Apenas altere para o hostname que será utilizado para cada novo host cliente.  


## **3. Instalação e Configuração do OpenLDAP** <a id="3"></a>

 \#\# \*\*\` Máquina Kerberos\_Server \`\*\* Instalar pacotes necessários. \`\`\`shell \# yum install openldap-servers openldap-clients migrationtools \`\`\` Copiar arquivo de exemplo do ldap \`\`\`shell \# cp /usr/share/openldap-servers/DB\_CONFIG.example /var/lib/ldap/DB\_CONFIG \`\`\` Alterar dono do diretórios e arquivos \`\`\`shell \# chown -R ldap: /var/lib/ldap/ \`\`\` Insira uma senha para o servidor ldap. Copie o hash da senha que será exibido no console após a senha ser fornecida. \`\`\`shell \# slappasswd \`\`\` \`\`\`shell \# cd /etc/openldap/slapd.d/cn\=config \# vim olcDatabase\=\{0\}config.ldif \`\`\` Insira a variável \*\*olcRootPW\*\* no arquivo \*\*olcDatabase={0}config.ldif\*\* e como valor coloque o hash da sua senha que foi gerado acima. Como exemplo abaixo. \*\*Arquivo:\*\* /etc/openldap/slapd.d/cn=config/olcDatabase={0}config.ldif \`\`\`conf ... olcRootPW: {SSHA}cMDvUOc6aI0ArtrUokrHtP9IzrU7okr+TjuZ7 \`\`\` No arquivo \*\*olcDatabase={2}hdb.ldif\*\* altere para o nome de domínio configurado no servidor Kerberos. Altere todas as ocorrências de \*\*dc=my-domain\*\* e \*\*dc=com\*\* para \*\*dc=jungle\*\* e \*\*dc=kvm\*\*, respectivamente. Insira também o \*\*olcRootPW\*\* como feito anteriormente. E por último insira os dois atributos \*\*olcAccess\*\* como feito a \*\*Arquivo:\*\* /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif \`\`\`conf dn: olcDatabase={2}hdb objectClass: olcDatabaseConfig objectClass: olcHdbConfig olcDatabase: {2}hdb olcDbDirectory: /var/lib/ldap olcSuffix: dc=jungle,dc=kvm olcRootDN: cn=Manager,dc=jungle,dc=kvm olcRootPW: {SSHA}58Rggp4f6iyS6Cvn5kGrrBBUKVhjod1A olcDbIndex: objectClass eq,pres olcDbIndex: ou,cn,mail,surname,givenname eq,pres,sub structuralObjectClass: olcHdbConfig entryUUID: 50ea6406-920c-1039-95f0-1d53df8faa2e creatorsName: cn=config createTimestamp: 20191102223154Z entryCSN: 20191102223154.230847Z\#000000\#000\#000000 modifiersName: cn=config modifyTimestamp: 20191102223154Z olcAccess: {0}to attrs=userPassword by self write by dn.base="cn=Manager,dc=jungle,dc=kvm" write by anonymous auth by \* none olcAccess: {1}to \* by dn.base="cn=Manager,dc=jungle,dc=kvm" write by self write by \* read \`\`\` No arquivo \*\*olcDatabase={1}monitor.ldif\*\* altere somente \*\*dc=my-domain\*\* e \*\*dc=com\*\* para \*\*dc=jungle\*\* e \*\*dc=kvm\*\*. Após configurar os três arquivos acima habilite e inicie o OpenLDAP. \`\`\`shell \# systemctl enable slapd \# systemctl start slapd \`\`\`  
 \*\*\` Firewall Habilitado \`\*\* Se o serviço firewalld estiver habilitado, adicione o LDAP e reinicie o serviço. \`\`\`shell \# firewall-cmd --permanent --add-service ldap -------------------- success \# firewall-cmd --reload -------------------- success \`\`\` ---  
 Adicionar os arquivos ao LDAP. \`\`\`shell \# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif \# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif \# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif \`\`\` Crie o arquivo que será usado pata a estrutura raiz do diretório ldap. \`\`\`shell \# vim base.ldif \`\`\` Para configurações usadas até aqui será criado a seguinte estrutura. \`\`\`conf dn: dc=jungle,dc=kvm objectClass: dcObject objectClass: organization dc: jungle o : jungle dn: ou=People,dc=jungle,dc=kvm objectClass: organizationalUnit ou: People dn: ou=Group,dc=jungle,dc=kvm objectClass: organizationalUnit ou: Group \`\`\` Insira a estrutura raiz ao LDAP e faça uma busca para confirmar a inserção. Caso a busca retorne a estrutura criada o insersão foi realizada com sucesso. \`\`\`shell \# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f base.ldif \# ldapsearch -x -D cn=Manager,dc=jungle,dc=kvm -W -b dc=jungle,dc=kvm \`\`\` No arquivo \*\*migrate\_common.ph\*\* altere o valor da variável \*\*DEFAULT\_MAIL\_DOMAIN\*\* para \*\*"jungle.kvm"\*\*, \*\*DEFAULT\_BASE\*\* para \*\*"dc=jungle,dc=kvm"\*\* e \*\*EXTENDED\_SCHEMA\*\* para o valor \*\*1\*\*. Abaixo o resultado das configurações somente nas três variáveis modificadas: \*\*Arquivo:\*\* /usr/share/migrationtools/migrate\_common.ph \`\`\`conf $DEFAULT\_MAIL\_DOMAIN = "jungle.kvm"; $DEFAULT\_BASE = "dc=jungle,dc=kvm"; ... $EXTENDED\_SCHEMA = 1; \`\`\`

Crie os usuários que serão usados para se usar o NFS. Como o Kerberos se encarregará da autenticação, não serão criados senhas para estes usuários.

```text
# useradd user1
# useradd user2
```

Vamos utilizar o **migrate\_passwd** e **migrate\_group** para converter as informações dos usuários e grupos criados acima na mesma estrutura do LDAP.

```text
# egrep  user[0-9] /etc/passwd > /tmp/users
# egrep  user[0-9] /etc/group > /tmp/groups
#
# /usr/share/migrationtools/migrate_passwd.pl /tmp/users /tmp/users.ldif
# /usr/share/migrationtools/migrate_group.pl /tmp/groups /tmp/groups.ldif
```

Adicione as estruturas criadas ao LDAP.

```text
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f /tmp/groups.ldif 
# ldapadd -x -D cn=Manager,dc=jungle,dc=kvm -W -f /tmp/users.ldif
```

## **`Máquina client_01`**

Instalar os pacotes necessários.

```text
# yum install nss-pam-ldapd
```

Vamos configurar o PAM para utilizar os serviços de LDAP e Kerberos para autenticação nos clientes. Iremos configurar o endereço e o DN do Servidor LDAP. E do Kerberos iremos configurar o Reino, o KDC e o Servidor de Admin.

Após executar o comando abaixo será aberto uma interface para alterar as configurações.

```text
# authconfig-tui
```

Marque ou altere as seguintes configurações na primeira tela que será aberta.

```text
[*] Utilizar LDAP
[*] Utilizar Kerberos
```

Selecione avançar.

```text
ldap://kerberos_server.jungle.kvm/
DN base: dc=jungle,dc=kvm
```

Selecione avançar.

```text
[*] Utilizar DNS para resolver máquinas para reinos
```

Clique em OK. E assim, o cliente foi configurado para usar o LDAP e Kerberos configurados anteriormente.

Para ter certeza que as configurações foram aplicadas, verifique no arquivo **/etc/nsswitch.conf** que as configurações indicam o uso do LDAP para autenticação.

**Arquivo:** /etc/nsswitch.conf

```text
...
passwd:     files sss ldap
shadow:     files sss ldap
group:      files sss ldap
...
```

Os usuários **user1** e **user2**, para se autenticar no Kerberos, foram criados apenas na máquina **kerberos\_server**. Ao configurar os serviços acima no PAM os usuários já podem ser listados localmente, utilizando os comando **getent** ou **id**. Mas repare, abrindo o arquivo **/etc/passwd** os usuários não serão listados, apenas pelos comandos acima citados.

```text
# getent passwd user1
-------------------
user1:x:1001:1001:user1:/home/user1:/bin/bash
# id user1
-------------------
uid=1001(user1) gid=1001(user1) grupos=1001(user1)
```

Faça as mesmas estapas acima para outros possíveis clientes que utilizarão o Kerberos.

## **4. Instalação e Configuração do NFS** <a id="4"></a>

## **`Máquina Kerberos_Server`**

Instalar pacotes necessários.

```text
# yum install nfs-utils
```

É preciso criar o arquivo **exports** para que seja inserido os diretórios que serão compartilhados pelo NFS. Abaixo será compartilhado o diretório **/home** com permissão de leitura e escrita.

**Arquivo:** /etc/exports

```text
/home    *(rw,sync)
```

Habilite o NFS na inicialização do sistema e inicie o serviço manualmente.

```text
# systemctl enable rpcbind
# systemctl enable nfs-server

# systemctl start rpcbind
# systemctl start nfs-server
```

Para listar os diretórios compartilhados pelo NFS execute:

```text
# showmount -e
-----------------
Export list for kerberos_server.localdomain:
/home *
```

**`Firewall Habilitado`**

Se o serviço firewalld estiver habilitado, adicione o NFS e reinicie o serviço.

```text
# firewall-cmd --permanent --add-service nfs
--------------------
success

# firewall-cmd --reload
--------------------
success
```

## **`Máquina client_01`**

Instalar pacotes necessários.

```text
# yum install nfs-utils autofs
```

Adicione a seguinte linha no final do arquivo **auto.master**.

**Arquivo:** /etc/auto.master

```text
...
/home /etc/auto.autofs --timeout=600
```

Agora vamos colocar o caminho local para onde o diretório compartilhado pelo **kerberos\_server** será montado.

**Arquivo:** /etc/auto.autofs

```text
*     kerberos_server:/home/&
```

Habilite o NFS na inicialização do sistema e inicie o serviço manualmente.

```text
# systemctl enable autofs
# systemctl start autofs
```

Pronto, o login com os usuários e hosts cadastrados no Kerberos poderão ser realizados a partir de agora. Faça logout e se logue com um dos usuários cadastrados no LDAP e Kerberos.

O login deve ser feito diretamente na máquina cliente configurada, para fazer o acesso via SSH nesta máquina deve ser realizado as configurações abaixo.

**`Login em máquina cliente via SSH`**

Para o login diretamente na máquina foi utilizado as configurações do PAM e assim permitir a autenticação via Kerberos. Mas para login em máquina cliente via SSH é preciso habilitar as opções nos arquivos mostrados abaixo.

**Arquivo:** /etc/ssh/ssh\_config

```text
...
GSSAPIAuthentication yes
GSSAPIDelegateCredentials yes
...
```

**Arquivo:** /etc/ssh/sshd\_config

```text
...
GSSAPIAuthentication yes
...
```

Reinicie o serviço de SSH.

```text
# systemctl reload sshd
```

Após logar em **client\_01** , com o user1 ou user2, observe que o diretório **home do usuário** está localizado na máquina **kerberos\_server**. Verifique executando o comando:

```text
# mount | grep /home/user
----------------
kerberos_server:/home/user1 on /home/user1 type nfs4 (rw,
relatime,vers=4.1,rsize=262144,wsize=262144,namlen=255,hard,
proto=tcp,port=0,timeo=600,retrans=2,sec=sys,
clientaddr=192.168.0.7,local_lock=none,addr=192.168.0.28)
```

Você pode listar os **tickets** fornecidos pelo kerberos ao logar nesta máquina com alguns dos usuários cadastrados.

```text
# klist
```

O **ticket** permite a quem possuí-lo logar em outras máquinas cliente configuradas sem que a senha seja solicitada, até a data de expiração do mesmo.

Para testar o login sem autenticação, recurso oferecido pelo Kerberos, configure uma segunda máquina cliente da mesma forma que foi realizado em **client\_01**. E na máquina **kerberos\_server** use o console do **kdmin.local** para adicionar este novo **host** clinente que será configurado.

## **5. Adicionando Novos Hosts e Usuários** <a id="5"></a>

Para adicionar novos hosts é preciso fazer as [configurações inicias](./#1) na nova máquina cliente, adicionar o host ao [kadmin.local](./#kadmin-local) e gerar o arquivo **.keytab** e depois realizar os mesmos passos realizados no **client\_01** nas etapas [2](./#2-2), [3](./#3-2) e [4](./#4-2).

Para adicionar novos usuários é preciso adiciona-los ao [kadmin.local](./#kadmin-local) e depois [cadastra-los ao LDAP](./#criar-usuario). As máquinas clientes estando configuradas para usar o LDAP e Kerberos os novos usuários já poderão ser usados para se autenticarem e acessarem os seus diretórios compartilhados pelo NFS.

