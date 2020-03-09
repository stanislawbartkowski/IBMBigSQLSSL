# IBM BigSQL and SSL connection
How to enable secure SSL connection between IBM BigSQL server and the client? Several usefull links:<br>
https://www.ibm.com/support/knowledgecenter/SSEPGG_11.1.0/com.ibm.db2.luw.admin.sec.doc/doc/t0025241.html<br>
https://www.ibm.com/support/knowledgecenter/SSCRJT_6.0.0/com.ibm.swg.im.bigsql.doc/doc/bi_admin_biga_ssl.html<br>
More practical: https://developer.ibm.com/hadoop/2016/01/08/configure-big-sql-support-ssl/<br>

Below I'm presenting a procedure which worked for me in several environments.
## BigSQL server
# Review the environment.
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

# Create a directory where all SSL related files will reside
Assuming /etc/bigsql/security directory.<br>
> mkdir /etc/bigsql/security<br>

There is no need for any other then *bigsql* to deal with this directory.<br>
> chmod 700 /etc/bigsql/security<br>
# Create a server key database
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
# Create self-signed certificate
> gsk8capicmd_64 -cert -create -label bigsql -db bigsql.kdb  -dn "CN=aa1.fyre.ibm.com"<br>

*-dn* parameter is the certficate subject and can include more features. It is a good practice the have *CN* as the hostname where BigSQL Head node is installed.<br>
Verify the current content of key database.<br>
> gsk8capicmd_64 -cert  -list  -db bigsql.kdb -stashed<br>
```
Certificates found
* default, - personal, ! trusted, # secret key
-	bigsql
```
# Create a certificate signed by CA authority.
If more trusted connectivity is required then instead of self-signed certificate the CA signed certificate is necessary.<br>
Create Certificate Signing Reqeust (csr). The same key database is used for certficate and CSR requests.<br>
> gsk8capicmd_64 -certreq -create -dn "CN=aa1.fyre.ibm.com,O=myBIGSQL,OU=FYRE,L=H,ST=MZ,C=WAW" -db bigsql.kdb -label bigsql -file bigsql.csr -stashed<br>
The *bogsql.csr* file should be created.<br>
Verify the content key database regarding CSR entries.<br>
>   gsk8capicmd_64 -certreq  -list  -db bigsql.kdb -stashed<br>
```
Certificates requests found
	bigsql

```
Send the *bigsql.csr* to the CA centre to be signed.<br>



