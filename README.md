# PKI
PKI for certificate management on z/O
## Summary
The files here are to help people customise the PKI Server product from IBM.
##  RACF setup
I found the IKYSETUP Rexx provided with the product did not meet my needs, and I spent a lot of time trying to get it working.  In the end, I found it was easier, and more under my control to create JCL with the definitions.   I could then see the commands, and could run the delete commands, before rerunning the define commands.

When you run the IKYSETUP exec you can have it execute the commands, or display them.  I took the displayed commands and create PDS members with jobs to to execute the RACF commands.

Within my jobs I have two steps, DEL and DEF.  If you want to delete or redefin the definitions, you use the DEL section then run the DEF section.

Some of the resources are system wide, some are just for the instance (or user).  If you are going to delete a resource, you need to make sure it is not being used else where.  For example it defines RDEFINE FACILITY BPX.SERVER.    This is used by many servers, and is a system wide resource.  Adding a userid to run the PKI server is an instance resource, and if you delete the userid it should not have a big impact.

Part of customising PKI is to define some VSAM files. You should either use an existing High Level Qualifier (HLQ) or create a alias so datasets for the HLQ and cataloged in a usercat, and not in the master catalog.
You need to do this before you define the VSAM files.

The members should be run in the following order

### DEFCAT - define a usercatalog
If you are going to use the default HLQ (PKISERVE) for the VSAM datasets, you should set up an alias pointing to a user catalog.   This member gives an example job to define a user catalog

### ALIAS - define an alias for the HLQ pointing to a user catalog
This member is an example IDCAMS define alias command.

### SYSTEM - define system wide resources 
This defines

- SETROPTS EGN GENERIC(DATASET) 
- RDEFINE FACILITY BPX.SERVER 
- RDEFINE FACILITY IRR.DIGTCERT.GENCERT 
- RDEFINE FACILITY IRR.DIGTCERT.LISTRING
- SETROPTS CLASSACT(SURROGAT) 
- RDEFINE SURROGAT BPX.SRV.&PKISERV this says the userid &PKISERV can act as a surrogate
- RDEFINE CSFKEYS IRR.DIGTCERT.CERTIFAUTH.* UACC(NONE)
- RDEFINE CSFSERV CSFOWH UACC(NONE)
- RDEFINE FACILITY IRR.RPKISERV.** 
- SETROPTS CLASSACT(CSFKEYS) RACLIST(CSFKEYS)    
- SETROPTS CLASSACT(CSFSERV)  RACLIST(CSFSERV) 
- SETROPTS GENERIC(FACILITY)
- SETROPTS CLASSACT(STARTED) RACLIST(STARTED)   

plus the setropts to refresh RACFLIST(...) for the resources changed.

### USER - userid related resources

- Adds a userid profile for the PKI started task (PKISERV)
- Adds a userid profile for acting as a surrogate user ( daemnon) PKISRVD
- Adds a group PKIGRP
- Adds a dataset profile for the PKISERV.** datasets, and give userids and PKIGRP access to it
- Gives access to various profile for user PKISRVD 
- Creates a started task profile and assigns a userid to it RDEFINE STARTED &PKISERVD..* STDATA(USER(&PKISRVD.)) 
- Creates a resource  IRR.RPKISERV.PKIADMIN and gives group PKIGRP access to it.  Userids in this group can issue PKI Admin commands,

### INSTANCE - instance related resources
- Generates a certificate authority certificate (Local CA)
- Creates a certificate signed by the Local CA for the Registration Authority
- Create a CA keyring and connects the Local CA certificate as (as the default) and the RA certicate
- Creates a web user certificate
- Creates web user keyring 
- Defines an RDATALIB profile for the keyring
- Connects the web user certificate, and the Local CA certificate to the keyring
- Exports the certificates

## HTTPD  Web server configuration files.

I had problems getting the web server part of PKIServer to work.  I've taken the IBM supplied entries, restructured them and paramerised them.  I find them much easier to work with.

With the Apache Web Server you can define a variable 

    Define PKIAppRoot /usr/lpp/pkiserv
then use it, for example
 
    ${PKIAppRoot}/PKIServ/ssl-cgi-bin*
resulting in
 
    /usr/lpp/pkiserv//PKIServ/ssl-cgi-bin

You can do condtitional processing

    <IfDefine CA1AdminAuth>  
       ${CA1AdminAuth} 
    </IfDefine> 

### The files are

- httpd.conf . This is the sample provided by IBM, but with  Include conf/colin.conf added at the bottom.  All other changes are made outside of the httpd.conf file
- colin.conf . This sets some defaults, loads some modules and includes pki.conf
- pki.conf
    - Define some constants
        - \#define my host name 
        - Define sdn 10.1.1.2 
        - Define PKIAppRoot /usr/lpp/pkiserv 
        - Define PKIKeyRing START1/MQRING 
        - Define PKILOG "/u/mqweb3/conf" 
        - \#Define PKISAFAPPL "OMVSAPPL"  this is the default
        - Define PKISAFAPPL "ZZZ" 
        - Define serverCert  "SERVEREC" 
        - Define pkidir  "/usr/lpp/pkiserv" 
        - \# the format of the trace entry 
        - Define elf "[%{u}t] %E: %M"
    - Define the CA instance specific stuff
        - \# define the CA specific stuff 
        - Define CA1Port 443 
        - Define CA1 PKIServ|Customers 
        - Define CA1PATH  "_PKISERV_CONFIG_PATH_PKIServ          /etc/pkiserv" 
        - Define CA1AdminAuth "  Require saf-group SYS1 "
        - \# uncomment these if you want the traces
        - \#Define _PKISERV_CMP_TRACE         0xff
        - \#Define _PKISERV_CMP_TRACE_FILE    /tmp/pkicmp.%.trc 
        - \#Define _PKISERV_EST_TRACE         0xff 
        - \#Define _PKISERV_EST_TRACE_FILE    /tmp/pkiest.%.trc
    - Include the virtual host files
        - Include conf/80.conf        
        - Include conf/443.conf
        - Include conf/1443.conf 
- pkisetenv.conf  this does a setenv for LIBPATH, and if defined, does a SetEnv for the _PKISERV... traces
- pkissl.conf .  This file is included in the 443.conf and 1443.conf config files.  
    - this contains the list of SSLCipherSpecs.   Ive extended the IBM list because the onese I needed  were not in the list
    - sslenable - to turn ssl on
    - defines the SAF key to uses ( KeyFile/saf ${PKIKeyRing} ).
    - defines the certificate the server should use (SSLServerCert ${serverCert}
- 80.conf this is the default port 80 - no protection.   The traffic is not encrypted, so I recommend this is not used.
- 443.conf  This is the definitions for TLS.   
    - this has  *SSLClientAuth  Required* saying the client needs to authenticate either by certificate or by userid and password
    - the section on authorised commands checks to see if ${CA1AdminAuth} is defined, if so, us it.  This allows you to configure pki.conf and say the userid needs to be in the saf group SYS1.
    - It has *SAFAPPLID ${PKISAFAPPL}* . This means users need read access to the specified APPLID. 
    - It has *SAFRunAs  %%CERTIF%%* which says use the certificate id if it has one.   The IBM version has this as *SAFRunAs $$CLIENT$$*
- 1443.conf this contains defintions for client access.   The IBM provided version looks to me to be insecure
    - it has the same *SAFAPPLID ${PKISAFAPPL}* so authorised users must have access to the applid 


  

