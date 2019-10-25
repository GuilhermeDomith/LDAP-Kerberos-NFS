# LDAP + Kerberos + NFS
Trabalho realizado para as disciplinas de Serviços e Segurança do curso de Tecnologia em Sistemas para Internet

--- 
## Cliente 01 (Servidor)

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

---
## Cliente 02 
> Fazer as mesmas estapas para outros possíveis clientes.

Instale os pacotes necessários para que o cliente se comunique com o servidor do _kerberos_.
```shell
# yum install pam_krb5 krb5-workstation
```

Copie e subistitua o arquivo de configurações:

```shell
# cp /tmp/krb5.conf /etc/
```

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

