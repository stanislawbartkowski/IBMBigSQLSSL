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

