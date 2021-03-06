![SAP HANA Academy](https://yt3.ggpht.com/-BHsLGUIJDb0/AAAAAAAAAAI/AAAAAAAAAVo/6_d1oarRr8g/s100-mo-c-c0xffffffff-rj-k-no/photo.jpg)
# SAP HANA Security #
## Secure Client Connections for the SAP HANA Service ##
The SAP HANA Service on the SAP Cloud Platform only accepts secure (encrypted) connections from client tools.

To make this happen, you have two options:
-  Use the default (built-in) TLS/SSL security provider of your platform</li>
-  Use the SAP CommonCrypto Library (SCL)</li>

This blog is about the first option. It is the easiest to use and requires almost no configuration on the Microsoft Windows or Java platforms and minimal setup on macOS and Linux.

In the tutorial video, we are using the SAP HANA Service from the Cloud Foundry environment. However, as this concerns client-side configuration, it works exactly the same in the Neo environment (SAP datacenter).

For those interested in how to configure secure SAP HANA client connections for on-premise SAP HANA, just ignore the "Service" word. On the client-side, it works exactly the same.

Using the built-in security providers does have some restrictions as they cannot be used for SAP HANA Client Side Encryption, for example. CSE requires SCL.

The SAP CommonCrypto Library was created by SAP to guarantee a secure compute environment regardless of the underlying platform. For on-premise SAP HANA, openSSL has been deprecated.

### Tutorial Video ### 
[![Secure Client Connections](https://img.youtube.com/vi/loi28PvDZVI/0.jpg)](https://www.youtube.com/watch?v=loi28PvDZVI "Secure Client Connections")


### Tutorial Video Playlist ### 
[SAP HANA Security](https://www.youtube.com/playlist?list=PLkzo92owKnVz2TJuTO9B71U7gTsG6beVJ)


## Convert Root CA Certificate from .crt to .pem ##
For platforms using openSSL (macOS and Linux), you will need to point the sslTrustStore parameter to the Certificate Authority (CA) root certificate. 
You can download the certificate from the [DigiCert](https://www.digicert.com/digicert-root-certificates.htm) website. The DigiCert Global Root CA Thumbprint = A8985D3A65E5E5C4B2D7D66D40C6DD2FB19C5436
* [DigiCertGlobalRootCA.crt](https://dl.cacerts.digicert.com/DigiCertGlobalRootCA.crt)

The CRT file is a binary file. You need to convert this into a PEM (text) format. The command below assumes you downloaded the file to the Downloads directory. Storing the PEM file in a .ssl directory under the user account is a convention, not a requirement. THere are no specific file permissions for the directory or file required (unlike for SSH). 
```
cd Downloads
mkdir ~/.ssl
openssl x509 -inform der -in DigiCertGlobalRootCA.crt -out ~/.ssl/DigiCertGlobalRootCA.pem
```
## HDBSQL ##
The SAP HANA Interactive Terminal is included with every SAP HANA client. 
You can use it quickly test the connection to the SAP HANA Service. Notwithstanding its name, interactive input is a bit cumbersome but for running scripts the tool can be handy. 
### For Microsoft Windows
On Microsoft Windows the built-in TLS/SSL provider will be used. No need to specify provider or trust store. 
```
hdbsql -n zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321 -u system -p Initial1 -e  "SELECT VERSION FROM M_DATABASE"
```
### For Linux and macOS
Make sure to include the sslprovider and ssltruststore parameters. 
```
hdbsql -n zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321  -u system -p Initial1 -e -sslprovider openssl -ssltruststore ~/.ssl/DigiCertGlobalRootCA.pem  "SELECT VERSION FROM M_DATABASE"
```
In case your client is behind a firewall, you can use the web service proxy on port 80. 
```
hdbsql -n wsproxy.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:80 -wsurl /service/95f8319f-bacf-4c79-ab28-76c72a4c8e71 -u system -p Initial1 -e -sslprovider openssl -ssltruststore ~/.ssl/DigiCertGlobalRootCA.pem  -proxyhost  proxy.org.corp -proxyport 8080  "SELECT VERSION FROM M_DATABASE"
```

For the documentation, see
* [SAP HANA HDBSQL Options - SAP HANA Administration Guide for SAP HANA Service](https://help.sap.com/viewer/6a504812672d48ba865f4f4b268a881e/Cloud/en-US/c24d054bbb571014b253ac5d6943b5bd.html)
* [WebSockets Communication Protocol - SAP Cloud Platform, SAP HANA Service Getting Started Guide](https://help.sap.com/viewer/cc53ad464a57404b8d453bbadbc81ceb/Cloud/en-US/eeb55c2016cf4048987e890190e7f5f9.html)

## ODBC ##
To connect with a ODBC client, you need to install the SAP HANA client for your platform.

On Microsoft Windows, use the ODBC Data Source Administrator to create a System or User Data Source. Make sure to select the right 32 or 64-bit architecture and select Connect Using SSL under Configuration.

On Linux and macOS, create a .odbc.ini file in your home directory for User Data Source or /etc (convention, no requirement) for System Data Sources. The order of the parameters does not matter nor does the parameter case. File name case does matter! Tip: verify your entries with the list (ls) command.

Below two example, for Linux and macOS. The extension of the ODBC driver file and the location is different but apart from that, the entries are identical. 
Storing the PEM file in a .ssl directory under the user account is a convention, not a requirement. THere are no specific file permissions for the directory or file required (unlike for SSH). 

You can give the Data Source any name you want. Case is not important, spaces are possible between quotes but not recommended. 

### For Linux
```
[HaaS]
driver=/usr/sap/hdbclient/libodbcHDB.so
serverNode=zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321
encrypt=Yes
sslCryptoProvider=openssl
sslTrustStore=/usr/sap/hdblcient/.ssl/DigiCertGlobalRootCA.pem
```
### For macOS
```
[HaaS]
driver=/Applications/sap/hdbclient/libodbcHDB.dylib
serverNode=zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321
encrypt=Yes
sslCryptoProvider=openssl
sslTrustStore=/Users/JohnDoe/.ssl/DigiCertGlobalRootCA.pem
```
### isql
Test your connection (and enter SQL) with isql. This tool is included with the unixODBC Driver Manager package. 
You can download unixODBC for Linux and Mac from [unixODBC.org](http://www.unixodbc.org) and admire the beautiful retro early 90s web design. 
Syntax is isql DataSourceName username password. There is no interactive prompt. Not entering a password, returns an error. 
```
isql HaaS user password
SELECT VERSION FROM M_DATABASE
```

For the documentation, see
* [Connect to SAP HANA via ODBC - SAP HANA Client Interface Programming Reference for SAP HANA Service](https://help.sap.com/viewer/1efad1691c1f496b8b580064a6536c2d/Cloud/en-US/66a4169b84b2466892e1af9781049836.html)

## JDBC ##
To connect with a JDBC client, you need to install the SAP HANA client for your platform.

You can test the JDBC connection on the command line. The order of the parameters matters as does the case. 

For the built-in TLS/SSL encryption using Java you do not have to specify the provider or location of a certificate as the CA root certificate of the Java RTE/SDK is used.
You might need to add the full path to the java executable if it is not in your %PATH% or $PATH. 

### For macOS
```
java -jar "/Applications/sap/hdbclient/ngdbc.jar" -u user,Password1 
-n zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321 
-o encrypt=true -o validateCertificate=true 
-c "SELECT VERSION FROM M_DATABASE"
```
### For Windows
```
java -jar "C:\Program Files\sap\hdbclient\ngdbc.jar" -u user,Password1 
-n zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321 
-o encrypt=true -o validateCertificate=true 
-c "SELECT VERSION FROM M_DATABASE"
```

### For Windows
Sample code for the TestJDBCDriver class. Add the next-generation database client <strong>ngdbc.jar</strong> file as an external archive to your package build path.

We only specify to the DriverManager to encrypt the connection <?encrypt=true>. The JVM takes care of provider and trust store. 

```
import java.sql.*;

public class TestJDBCDriver {
	public static String connectionString =
		"jdbc:sap://zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com:54321/?encrypt=true";
	public static String user = "Donald";
	public static String password = "Individual1";
	public static void main(String[] argv) {
		Connection connection = null;
		try {
			connection = DriverManager.getConnection(connectionString, user, password);
		} catch (SQLException e) {
			System.err.println("Connection Failed. User/Passwd Error? Message: " + e.getMessage());
			return;
		}
		if (connection != null) {
			try {
				System.out.println("Connection to SAP HANA Service successful.");			
				Statement stmt = connection.createStatement();
				ResultSet resultSet = stmt.executeQuery("SELECT VERSION FROM M_DATABASE");
				resultSet.next();
				String version = resultSet.getString(1);
				System.out.println("Version = "+version);
			} catch (SQLException e) {
				System.err.println("Query failed!");
			}
		}
	}
}
```
For the documentation, see
* [Connect to SAP HANA via JDBC - SAP HANA Client Interface Programming Reference for SAP HANA Service](https://help.sap.com/viewer/1efad1691c1f496b8b580064a6536c2d/Cloud/en-US/ff15928cf5594d78b841fbbe649f04b4.html)

## Python ##
To connect with a Python client to the SAP HANA Service, you first need to install the SAP HANA client for your platform and then install hdbcli for your Python environment.
```
# N.N.N = 2.3.130 (January, 2019)
pip install /usr/sap/hdbclient/hdbcli-N.N.N.tar.gz # Linux
pip install /Applications/sap/hdbclient/hdbcli-N.N.N.tar.gz # macOS
pip install "C:\Program Files\SAP\hdbclient\hdbcli-N.N.N.zip" # Windows
```
When running your Python code on a Microsoft Windows platform, you only need to set encrypt=true. 
On macOS and Linux, you also need to specify sslCryptoProvider and sslTrustStore.
```
from hdbcli import dbapi

conn = dbapi.connect(
    address='zeus.hana.prod.eu-central-1.whitney.dbaas.ondemand.com', 
    port=54321, 
    user='cobra', 
    password='MySecret', 
    encrypt='true', 
    sslCryptoProvider='openssl', 
    sslTrustStore='/Users/cobra/.ssl/DigiCertGlobalRootCA.pem'
)

with conn.cursor() as cursor:
	sql = "SELECT SYSTEM_ID, DATABASE_NAME, VERSION FROM M_DATABASE"
	cursor.execute(sql)
	result = cursor.fetchall()
print ("Connection to SAP HANA Service successful.")
print ("SID =", result[0][0])
print ("Database Name =", result[0][1])
print ("Version =", result[0][2])
conn.close()
```

For the documentation, see
* [Connect to SAP HANA from Python - SAP HANA Client Interface Programming Reference for SAP HANA Service](https://help.sap.com/viewer/1efad1691c1f496b8b580064a6536c2d/Cloud/en-US/d12c86af7cb442d1b9f8520e2aba7758.html)

## Documentation ## 
* [Connecting to an SAP HANA Service Instance Directly from SAP HANA Clients - SAP HANA Client Interface Programming Reference for SAP HANA Service](https://help.sap.com/viewer/1efad1691c1f496b8b580064a6536c2d/Cloud/en-US/5bd9bcec690346a8b36df9161b1343c2.html)
* [WebSockets Communication Protocol – SAP Cloud Platform, SAP HANA Service Getting Started Guide](https://help.sap.com/viewer/cc53ad464a57404b8d453bbadbc81ceb/Cloud/en-US/eeb55c2016cf4048987e890190e7f5f9.html)
