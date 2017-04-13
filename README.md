# Certbot-renew-hook

To be worked on asap
Certbot TLSA Generator
This bash script was created to generate and automatically install TLSA records when called by the Letsencrypt (certbot) renew-hook.  It started out to be a “simple” wrapper to Viktor Dukhovni's tlsagen. 
In addition this script maybe run standalone. 
How it works
There are two elements which are used to determine if a TLSA record should be generated and installed.  
The first is the list of services for which we want TLSA records.  Common services are:
Service
Port
smtp
25
smtps
465 (I believe that this service is depracted, and as such should not be used or included).
submission
587
imap
143
imaps
993
sieve
4190 (I am not sure if this should be included)
caldav/
carddav/
htttps
443

The second, which is described below, is a list of domains and subdomains that are included in a security certificate.
For each service that we require a tlsa record for, we check to see if the service host matches one the domains or subdomains in the certificate. If there is a match we then generate a TLSA record using the port, service host and certificate. We use TCP as the default protocol.
Operating as a Renew-Hook
After a certicate has been renewed certbot calls any renew-hooks that have been set to run, once for each successfully renewed certificate. These commands are passed two shell variables,
1. RENEWED_LINEAGE which points to the location of the new certs and keys. Currently this is /etc/letsencrypt/live/certificate-name
2.  RENEWED_DOMAINS which contains a space delimited list of the certificate’s domains and subdomains.
Standalone operation
This program can also be run standalone, in which case it makes use of two positional parameters:
1. The first ($1) acts in the same way as RENEWED_LINEAGE above, and must define the path to the certificate and keys for which we wish to generate TLSA records. 
2. the second ($2) takes the place of RENEWED_DOMAINS and consists of a quoted space-delimited list of the domains and/or subdomains eg “ example.com smtp.example.com imap.example.com …”
As we cannot use command line parameters when called as a Certbot renew-hook all other parameters have to set using included files. I use the “/etc/default/CertbotTLSAgen.cf” for general/global parameters, and similar file placed in the …/letsencrypt/live for certificate specific paramters.  The certificate specific parameters override the general/global parameters.
Installation
Copy the all files to a location of your choice. Ensure that the file CertbotTLSAgen has execute permission.
Copy the file CertbotTLSAgen.cf to /etc/defaults.
Copy CertbotTLSAgen.cf to …/letsencrypt/live/certificate name.
Follow the instruction about activating a renew-hook in the Letsencrypt documentation.
CertbotTLSAgen.cf
There are two instances of this file used when this app is run, the “global” version which is stored in /etc/default, and the certificate specific instance which is kept in …/live/cert-name. The certificate specific version OVERRIDES the “global” version. The parameters and there purpose are listed below.
installPath="/usr/local/bin"
The “installPath” variable indicates where CertbotTLSAgen and its associated files are stored. 
THIS MUST BE SET in /etc/default/CertbotTLSAgen.cf
In order to ensure that this is not accidentally modified I would suggest that this line only be included in the global version of this file (/etc/default/CertbotTLSAgen.cf).
services=( smtp smtps submission imap sieve dav davical https )
The “services” variable is an array of service names, to specify which services you want to records for.
The common service names and their ports are shown above.
updateAction=”manual”
After the TLSA records have been created they need tobe added to your DNS system. The methodss currently available are:
	manual	     The system takes no action to add the TLSA records to your DNS, you must do the update                                                		      manually.
	nsupdate    The system will attempt to install the TLSA records in your DNS, using nsupdate. 
		     In order for this to work zones for which you want to use this facility must be 
                                capable of dynamic update.
TLSA_TTL=1800
The TTL for TLSA records should probably be kept low, probably not more than hour. I use 30 minutes (1800 seconds) as the TTL for TLSA records.
TLSA_AutoRemove=no
Do you want the system to automatically remove old/replaced TLSA records automatically, yes or no.

TLSA_RemoveDelay=5400
The approximate length of time in seconds between the installation of new/renewed certificates and the removal of old certificates. 
TLSA Generator parameters
	TLSA_Usage=dane-ee
	TLSA_Selector=pkey
	TLSA_Type=sha-256
outputFilepath="/etc/bind/tlsa/example.com"
The location where files containing the generated TLSA records will be placed.  If the path do0.00 cmes not exists, we will attempt to create it.

outputFilenameBase="tlsa"
The filename base is used to generate actual filenames in the form of base.sequence.
logFile=”/var/log/certbot_tlsagen,log”
The name and location of the CertbotTLSAgen log file.
logOnly=”no”
When running stand alone output messages to the terminal as well as the log, yes or no. 
