# IBM BigSQL and SSL connection
How to enable secure SSL connection between IBM BigSQL server and the client? Several useful links:<br>

https://www.ibm.com/support/knowledgecenter/SSEPGG_11.1.0/com.ibm.db2.luw.admin.sec.doc/doc/t0025241.html<br>

https://www.ibm.com/support/knowledgecenter/SSCRJT_6.0.0/com.ibm.swg.im.bigsql.doc/doc/bi_admin_biga_ssl.html<br>

More practical: https://developer.ibm.com/hadoop/2016/01/08/configure-big-sql-support-ssl/<br>

Below I'm presenting a procedure which worked for me in several environments.
# BigSQL server
## Review the environment.
The BigSQL secure connection can be configured using *bigsql* credentials. No need for *root* authority.<br>
BigSQL SSL connection is implemented by means of IBM Global Security Kit (GSKit). Make sure that GSKit dependency is included in the *LD_LIBRARY_PATH*.<br>
(as *bigsql* user)<br>
>  echo $LD_LIBRARY_PATH
```
/home/bigsql/sqllib/lib64:/home/bigsql/sqllib/lib64/gskit:/home/bigsql/sqllib/lib32
```
Find the *gsk8capicmd_64* utility and add to *PATH* variable.
> locate gsk8capicmd_64
```
/usr/ibmpacks/bigsql/6.0.0.0/db2/gskit/bin/gsk8capicmd_64
```
>export PATH=$PATH:/usr/ibmpacks/bigsql/6.0.0.0/db2/gskit/bin<br>

## Create a directory where all SSL related files will reside
Assuming /etc/bigsql/security directory.<br>
> mkdir /etc/bigsql/security<br>

There is no need for any other then *bigsql* to deal with this directory.<br>
> chmod 700 /etc/bigsql/security<br>
## Create a server key database
> cd /etc/bigsql/security/<br>
> gsk8capicmd_64 -keydb -create -db bigsql.kdb -pw "secret" -stash<br>

Verify that all necessary files are in place.<br>
> ls -l
```
rw------- 1 bigsql hadoop  88 Mar  9 12:00 bigsql.crl
-rw------- 1 bigsql hadoop  88 Mar  9 12:00 bigsql.kdb
-rw------- 1 bigsql hadoop  88 Mar  9 12:00 bigsql.rdb
-rw------- 1 bigsql hadoop 193 Mar  9 12:00 bigsql.sth
```
## Create self-signed certificate
> gsk8capicmd_64 -cert -create -label bigsql -db bigsql.kdb  -dn "CN=aa1.fyre.ibm.com"<br>

*-dn* parameter is the certificate subject and can include more features. It is a good practice the have *CN* as the hostname where BigSQL Head node is installed.<br>
Verify the current content of key database.<br>
> gsk8capicmd_64 -cert  -list  -db bigsql.kdb -stashed<br>
```
Certificates found
* default, - personal, ! trusted, # secret key
-	bigsql
```
## Create a certificate signed by CA authority.
For more trusted environment, user certificates signed by CA authority.<br>
Create Certificate Signing Reqeust (csr). The same key database is used for certficates and CSR requests.<br>
> gsk8capicmd_64 -certreq -create -dn "CN=aa1.fyre.ibm.com,O=myBIGSQL,OU=FYRE,L=H,ST=MZ,C=WAW" -db bigsql.kdb -label bigsql -file bigsql.csr -stashed<br>

The *bigsql.csr* file should be created.<br>
Verify the content key database regarding CSR entries.<br>

>   gsk8capicmd_64 -certreq  -list  -db bigsql.kdb -stashed<br>
```
Certificates requests found
	bigsql

```
Send the *bigsql.csr* to the CA centre to be signed.<br>
Two files should be received from CA centre.
* CA root chain certificate (here ca-chain.cert.pem)
* BigSQL server certificate signed by CA (here aa1.fyre.ibm.com.cert.pem)

Add root CA and BigSQL signed certificates to the key dataabase. Pay attention to *receive* command to add BigSQL certficate, it matches the certificate with the proper SSL key.
> gsk8capicmd_64 -cert -add -db bigsql.kdb -file /tmp/ca-chain.cert.pem -stashed<br>
> gsk8capicmd_64 -cert -receive -db bigsql.kdb -file /tmp/aa1.fyre.ibm.com.cert.pem -stashed<br>
<br>
Verify the current content of key database.<br>

> gsk8capicmd_64 -cert  -list  -db bigsql.kdb -stashed<br>

```
Certificates found
* default, - personal, ! trusted, # secret key
!	CN=thinkde.sb.com,OU=IntermediateRoom,O=MyHome,ST=Mazovia,C=PL
!	CN=thinkde.sb.com,OU=MyRoom,O=MyHome,L=Warsaw,ST=Mazovia,C=PL
-	bigsql

```
### BigSQL workers
Distribute *bigsql.kdb* and *bigsql.sth* across BigSQL Worker nodes<br>
> scp /etc/bigsql/security/bigsql.kdb <worker hostname>:/etc/bigsql/security/<br>
> scp /etc/bigsql/security/bigsql.sth <worker hostname>:/etc/bigsql/security/<br>

## Enable BigSQL for SSL connection
>db2 update dbm cfg using SSL_SVR_KEYDB /etc/bigsql/security/bigsql.kdb<br>
>db2 update dbm cfg using SSL_SVR_STASH /etc/bigsql/security/bigsql.sth<br>
>db2 update dbm cfg using SSL_SVR_LABEL bigsql<br>
>db2 update dbm cfg using SSL_SVCENAME 32052<br>

Several remarks
* Use full path names for key database and stash files.
* Decide on secure port connection, here 32502. Avoid using standard SSL ports like 8443, 443 etc.

Enable SSL<br>

Keep both, secure and non-secure connections active.
> db2set DB2COMM=SSL,TCPIP<br>

Only SSL connection available, disable non-secure.
>db2set DB2COMM=SSL<br>
## Restart BigSQL 
> bigsql stop<br>
> bigsql start<br>

Verify that BigSQL secure port is enabled<br>
```
> openssl s_client -connect aa1.fyre.ibm.com:32052
```
CONNECTED(00000003)
depth=2 C = PL, ST = Mazovia, L = Warsaw, O = MyHome, OU = MyRoom, CN = thinkde.sb.com
verify error:num=19:self signed certificate in certificate chain
---
Certificate chain
 0 s:/C=WAW/ST=MZ/L=H/O=myBIGSQL/OU=FYRE/CN=aa1.fyre.ibm.com
   i:/C=PL/ST=Mazovia/O=MyHome/OU=IntermediateRoom/CN=thinkde.sb.com
 1 s:/C=PL/ST=Mazovia/O=MyHome/OU=IntermediateRoom/CN=thinkde.sb.com
   i:/C=PL/ST=Mazovia/L=Warsaw/O=MyHome/OU=MyRoom/CN=thinkde.sb.com
 2 s:/C=PL/ST=Mazovia/L=Warsaw/O=MyHome/OU=MyRoom/CN=thinkde.sb.com
   i:/C=PL/ST=Mazovia/L=Warsaw/O=MyHome/OU=MyRoom/CN=thinkde.sb.com
---
Server certificate
-----BEGIN CERTIFICATE-----
............
```
## Extract the certificate to be picked-up by the client software
> gsk8capicmd_64 -cert -extract -db bigsql.kdb  -label bigsql -target /tmp/bigsql.arm -format ascii -fips -stashed<br>

# SSL client enablement
## jsqsh client
Use Java *keytool* to create keystore to be used by *jsqsh* utility or add to already existing keystore. The password here is used to protect Java keystore, it is not the password to get access to BigSQL key database.<br>
> keytool -import -file /etc/bigsql/security/bigsql.arm -keystore server.jks<br>

Launch *jsqsh*<br>
> jsqsh<br>
> 1> \connect -Ubigsql -Pbigsql -S aa1.fyre.ibm.com -p 32052 -ddb2 -Dbigsql -O sslConnection=true -O sslTrustStoreLocation=/home/bigsql/server.jks -O sslTrustStorePassword=secret<br>
<br>
Directly using jsqsh command line parameters<br>

>jsqsh -ddb2  -O sslConnection=true -O sslTrustStoreLocation=/home/bigsql/server.jks -O sslTrustStorePassword=secret -S aa1 -p 32052 -Ubigsql -Dbigsql<br>

* -Ubigsql The BigSQL user to connect
* -Pbigsql The BigSQL user password
* -S aa1.fyre.ibm.com BigSQL Head node hostname
* -p 32052 The SSL secure port
* -O sslTrustStoreLocation= The qualified pathname of the Java keystore file 
* -O sslTrustStorePassword= The password protecting Java keystore
<br>
Quick test that the SSL connection is working.<br>

> select * from syscat.tables;<br>
> create hadoop table x (x int);<br>
> insert into x values(1);<br>
> select * from x;<br>
> drop table x;<br>


