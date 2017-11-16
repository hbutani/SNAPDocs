**Goal**: To connect B.I clients like OBIEE through ODBC to a kerberized cluster.

**Pre-Requisites**: Need MIT Kerberos client on the desktop where you are connecting from. Ask your admin for the server krb5.conf ( /etc/krb5.conf typically). Copy this locally. 

Run kinit as the user that is given the permissions on the kerberized cluster. 

typically kinit user@HOST.DOMAIN.COM 

Then do a klist. It should list the credentials cache. 

ODBC INI file sample for connecting to Kerberized cluster. 

This uses the Simba ODBC Driver for Spark. 

 
[ODBC]
 
; Specify any global ODBC configuration here such as ODBC tracing.
[ODBC Data Sources]
Sparkline DSN = Simba Spark ODBC Driver
 
[Sparkline DSN]
Driver          = /Library/simba/spark/lib/libsparkodbc_sbu.dylib
Description     = Sparkline ODBC Driver DSN
HOST            = snap002.com
PORT            = 10001
Schema          = sample
SparkServerType = 3
AuthMech        = 1
KrbHostFQDN     = _HOST
KrbServiceName  = spark
KrbRealm        = domain.COM


Set this up as a system or user DSN on mac or windows. 

In Excel or any B.I client you can now use the DSN to connect.