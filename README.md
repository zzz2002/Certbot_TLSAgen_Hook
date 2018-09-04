## CerbotTlsagen --- Certbot-renew-hook
**Copyright (c) 2017, 2018 John L. Allen**  
This program is free software: you can redistribute it and/or modify it under the terms of the 
GNU General Public License as published by the Free Software Foundation, either version 3 of 
the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; 
without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  
See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <http://www.gnu.org/licenses/>.

# Certbot TLSA Generator

This bash script was created to generate and automatically add and delete TLSA records.

It started out to be a “simple” wrapper to Viktor Dukhovni's tlsagen script.

It can be run two way
1) as a Letsencrypt (certbot) renew-hook.
2) as a Standalone "utility".

## Operating as a Renew-Hook

After a certificate has been renewed and new certificates generated, certbot calls any "renew-hooks" that have been specified to be run, once for each successfully renewed certificate. The "hook" is passed two arguments,
1. Called **RENEWED_LINEAGE** is the location of the new certs and keys. Currently this is usually "/etc/letsencrypt/live/*certificate-name*"
2. Called **RENEWED_DOMAINS** which contains a space delimited list of the certificate’s domains and subdomains.
## Standalone operation
This program can also be run standalone, in which case it makes use of two positional parameters:
1. **$1** acts in the same way as RENEWED_LINEAGE above.
2. **$2** acts in the same way as RENEWED_DOMAINS above.
## How it works

There are two elements which are used to determine if a TLSA record should be generated and installed.
The first is the list of services for which we want TLSA records.  Common services are:

|Service|Port|Notes|
|:---:|---:|:---|
|smtp|25||
|smtps|465|I believe that this service is deprecated.|
|submission|587 ||
|imaps|993 ||
|sieve|4190|I am not sure if this should be included. |
|<dl><dd>caldav</dd><dd>carddav</dd><dd>https</dd></dl>|443|

For each service that we require a TLSA record for, we check to see if the service host matches one the domains or sub-domains in the certificate. If it does then a TLSA record using the port, service host and certificate is generated. We use TCP as the default protocol.
## Installation
Copy the all files to a location of your choice. Ensure that CertbotTLSAgen and any CertbotTLSAgen.DNSupdate.\* files are executable.
Copy the file *CertbotTLSAgen.cf* to */etc/defaults.* and to *…/letsencrypt/live/certificate-name*.
Follow the instruction about activating a renew-hook in the Letsencrypt documentation.

## Configuration
As we cannot use command line parameters when called as a Certbot renew-hook all other parameters have to set using configuration files.\
I use the */etc/default/CertbotTLSAgen.cf* for general/global parameters, and similar file placed in the *…/letsencrypt/live/cert-name/CetbotTLSAgen.cf* for certificate specific parameters.

The parameters in the certificate specific file ***override*** the general/global parameters.

### installPath="/usr/local/bin"
The “installPath” variable indicates where CertbotTLSAgen and its associated files are stored.
In the latest version this is retrieved/set by using the *dirname $(readlink -f "$0")*\
If **THIS MUST BE SET** then I would strongly suggest setting it in global configuration file (*/etc/default/CertbotTLSAgen.cf*) and that the variable be made read only in order to ensure that this is not accidentally modified.\
e.g. declare -r installPath="/usr/local/bin" use declare -r to make it read only


### services=( smtp smtps submission imap sieve dav davical https )
This is an array of service names, to specify which services you want to TLSA records for.\
e.g. *services=( dav https )* would limit the system to only generating certificates for those services.

### updateAction=”manual”
After the TLSA records have been created they need to be added to your DNS system. The methods currently available are:
 - **manual** The system takes no action to add the TLSA records to your DNS, you must do the update manually.\
 - **nsupdate** The system will attempt to install the TLSA records in your DNS, using the ISC bind utility **"nsupdate"**.\
 In order for this to work zones for which you want to use this facility must be capable of dynamic update.
### TLSA_TTL=1800
The TTL for TLSA records should probably be kept low, probably not more the 1 hour. I use 30 minutes (1800 seconds) as the TTL for TLSA records.
If a certificate is compromised you want the TLSA records associated with the compromised cert to expire ASAP so that they show up as invalid ASAP.
Ideally things like TLSA records etc, should not be cached, unfortunately we don't live in an ideal world
### TLSA Generator Parameters
The following parameters will produce 2 1 1 Certificate Authority certificate
```
CA_Required=y
CA_Usage=dane-ca	TLSA usage	also seen as 3 in TLSA records
CA_Selector=pkey	TLSA selector	also seen as 1 in TLSA records
CA_Type=sha-256		TLSA type	also seen as 1 in TLSA records
```
The following parameters will produce 3 1 1 TLSA certificate
```
EE_Required=y
EE_Usage=dane-ee	TLSA usage	also seen as 3 in TLSA records
EE_Selector=pkey	TLSA selector	also seen as 1 in TLSA records
EE_Type=sha-256		TLSA type	also seen as 1 in TLSA records
```

### TLSA_AutoRemove=no
Do you want the system to remove old/replaced TLSA records automatically, yes or no. 

### TLSA_RemoveDelay=5400
The approximate length of time in seconds between the installation of new/renewed certificates and the removal of old certificates. 

If TLSA_Autoremove is set to yes then a command will be passed to AT utility to start a TLSA delete operation after TLSA\_RemoveDelay seconds.\
**Note** as AT's smallest unit of time is minutes the time delay will be converted to minutes rounded down.

### outputFilepath=
This location defaults to RENEWED_LINEAGE.
The TLSA update records will be put in either the "TLSA\_additions" or "TLSA\_deletions" file, these files are currently saved 
in the Letencrypt live directory. This can be overridden by specifying a path in outputFilepath.\
NB.If you have more than one certificate for which you create TLSA records you will need one path per certificate. 
This can be done by setting this in the certificate version of the configuration file.

### logOnly=”no”
When running stand alone output messages to the terminal as well as the log, yes or no.

### logFile=”/var/log/letsencrypt/tlsagen.log”
The name and location of the CertbotTLSAgen log file.

### pseudoSRVrecords=( ... )
```
pseudoSRVrecords=( [smtp.example.com]='25 smtp.example.com.' 
		   [submission.example.com]='587 smtp.example.com.' 
		   [imap.example.com]='993 imap.example.com.' [sieve.example.com]='4190 sieve.example.com.' 
		   [dav.example.com]='443 dav.example.com.' [davical.example.com]='443 davical.example.com.' 
		   [https.example.com]='443 www.example.com.' ) 
```
Rather than using DIG to retrieve ports and targets, we can use a pseudo dig operation to retrieve this data.
pseudoSRVrecords is an associative array, the index to the array is the service host each entry in the array consists of the port associated with the service and the target URL for the service.
