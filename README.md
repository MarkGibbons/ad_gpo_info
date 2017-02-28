
duction</h3>
<p>We needed to be able to read GPOs attached to the OUs holding our computer objects.  I couldn't find clear documentation about the process so compiled these notes.  We are using RHEL 7 and  AD 2008.  These methods solved my problems, use at your own risk without any warranty or promise that these methods are at all safe to use.</p>
<h3>
  <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;">GPO structure</span>
</h3>
<h3>
  <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;"> </span>
  <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;">OU structure</span>
</h3>
<p style="margin-left: 30.0px;">
  <span style="color: rgb(0,0,0);font-size: 16.0px;">Our computer objects are kept in an OU tree with a structure similar to this:</span>
</p>
<p style="margin-left: 30.0px;">
  <span style="color: rgb(0,0,0);font-size: 16.0px;">   cn -&gt; ou=type, ou=app, ou=computers, ou=unix</span>
</p>
<p>
  <span style="color: rgb(0,0,0);font-size: 16.0px;"> </span>
  <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;">Packages</span>
</p>
<p style="margin-left: 30.0px;">RHEL 7 packages to install.  This list is probably sufficient but may include extra packages.</p>
<ul>
  <li>adcli</li>
  <li>authconfig</li>
  <li>krb5-workstation</li>
  <li>openldap-clients</li>
</ul>
<h5>Join active directory</h5>
<h6 style="margin-left: 30.0px;">Kerberos</h6>
<p style="margin-left: 60.0px;">Kerberos is a protocol for network authentication. Authentication tokens are strongly encrypted.  We end up using one clear text password read from a secured file.</p>
<h6 style="margin-left: 30.0px;">/etc/krb5.keytab</h6>
<p style="margin-left: 60.0px;">The adcli command may be used to join a server to active directory.  As part of the join host level credentials are created and are stored in /etc/krb5.keytab.  The credentials in the keytab may be used to create a ticket granting ticket.</p>
<h6 style="margin-left: 30.0px;">/etc/krb5.conf</h6>
<ac:structured-macro ac:macro-id="c2fa551b-9b99-4882-9e85-2db7fcd9d08e" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log
[libdefaults]
 default_realm = NORDSTROM.NET
 dns_lookup_realm = true
 dns_lookup_kdc = true
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
[domain_realm]
 nordstrom.net = NORDSTROM.NET
 .nordstrom.net = NORDSTROM.NET]]></ac:plain-text-body>
</ac:structured-macro>
<p> </p>
<h6 style="margin-left: 30.0px;">Join command</h6>
<p style="margin-left: 60.0px;">This command reads a password from the password file.  You could also distribute a key tab file with established host credentials.  </p>
<ac:structured-macro ac:macro-id="b9cc0b25-a3e6-4dad-9e8e-4d5a346e9b46" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[# tr d "\012" <passwordfile adcli join -U <user> --stdin-password -O <container' -D nordstrom.net ]]></ac:plain-text-body>
</ac:structured-macro>
<h5>kinit</h5>
<h6 style="margin-left: 30.0px;">krbtgt</h6>
<p style="margin-left: 60.0px;">The ticket granting ticket is created using the credentials from the keytab.  TGT is usually only valid for a limited amount of time/</p>
<h6 style="margin-left: 30.0px;">/tmp/krb5cc_0</h6>
<p style="margin-left: 60.0px;">The ticket granting ticket and service principle tickets are kept in this file.</p>
<h6 style="margin-left: 30.0px;">Service principle</h6>
<p style="margin-left: 60.0px;">Using ldapsearch will cause a ticket for an ldap service principle to be created.</p>
<h6 style="margin-left: 30.0px;">kinit to refresh the TGT.  </h6>
<p style="margin-left: 60.0px;">Use the credentials in the key tab to refresh the krbtgt ticket.</p>
<ac:structured-macro ac:macro-id="e276f54b-02d5-4d69-b52c-582479e3a57b" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[kinit -k -t /etc/krb5.keytab]]></ac:plain-text-body>
</ac:structured-macro>
<h6 style="margin-left: 30.0px;">klist</h6>
<p style="margin-left: 60.0px;">Can be used to list the currently active tickets.</p>
<ac:structured-macro ac:macro-id="27494780-5836-4619-a076-9c1c5ade0396" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[Ticket cache: FILE:/tmp/krb5cc_0
Default principal: host/y0319t11043.nordstrom.net@NORDSTROM.NET
Valid starting       Expires              Service principal
02/18/2017 11:23:27  02/18/2017 21:23:27  krbtgt/NORDSTROM.NET@NORDSTROM.NET
	renew until 02/25/2017 11:23:27
02/18/2017 11:24:21  02/18/2017 21:23:27  ldap/dc7.nordstrom.net@NORDSTROM.NET
	renew until 02/25/2017 11:23:27]]></ac:plain-text-body>
</ac:structured-macro>
<p style="margin-left: 60.0px;"> </p>
<h5>Domain Controller</h5>
<p>        Nslookup or equivalent may be used to find domain controllers.  In my case nordstrom.net is a round robin c-name.  It doesn't play well with ldapsearch.  I pull a specific server from the list and use that. </p>
<ac:structured-macro ac:macro-id="a59ebdae-87b6-4de1-a4e5-8af645f7bbf7" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[   # nslookup nordstrom.net ]]></ac:plain-text-body>
</ac:structured-macro>
<p> </p>
<h5>Server OU</h5>
<h6>        Find the server OU.  </h6>
<p style="margin-left: 60.0px;">From that name put together a table parent OUs.</p>
<ac:structured-macro ac:macro-id="49ce5ae6-d42a-483f-b207-4dfc40fc181c" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[# ldapsearch -h 'dc7.nordstrom.net' -s sub -b 'ou=unix,dc=nordstrom,dc=net' -Y GSSAPI "(samaccountname=Y0319P01$)"]]></ac:plain-text-body>
</ac:structured-macro>
<p style="margin-left: 30.0px;">distinguishedName: CN=y0319p01,OU=Prod,OU=UNIXALL,OU=Computers,OU=UNIX,DC=nord<br/> </p>
<h5>GPO attached to OU</h5>
<p>        Find the parent OUs, search each of them for attached GPOs.</p>
<p style="margin-left: 30.0px;">ou=prod,ou=unixall,ou=computers,ou=unix</p>
<p style="margin-left: 30.0px;">ou=unixall,ou=computers,ou=unix</p>
<p style="margin-left: 30.0px;">ou=computers,ou=unix</p>
<p style="margin-left: 30.0px;">ou=unix</p>
<p style="margin-left: 30.0px;">Search each OU for gPlink, get cn in policies,system,nordstrom,net</p>
<p> </p>
<ac:structured-macro ac:macro-id="10c7c73b-a8f9-43e2-b268-bbdad742786f" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[# ldapsearch -h 'dc7.nordstrom.net' -s base -b 'ou=prod,ou=unixall,ou=computers,ou=unix,dc=nordstrom,dc=net' -Y GSSAPI "(objectclass=organizationalunit)"]]></ac:plain-text-body>
</ac:structured-macro>
<p> </p>
<p style="margin-left: 30.0px;">gPLink: [LDAP://cn={84C43D09-B377-421E-82C2-FFFFFFFFF},cn=policies,cn=system,DC=nordstrom,DC=net;0]</p>
<h5>smbclient </h5>
<p style="margin-left: 30.0px;">We can use smbclient to list the file structure or retrieve files from the policy.  Here's an example of retrieving a file.  You can treat smbclient much like an ftp client.</p>
<p> </p>
<ac:structured-macro ac:macro-id="d0c57d8a-b6e9-4938-975b-e3951ec390e8" ac:name="code" ac:schema-version="1">
  <ac:plain-text-body><![CDATA[# smbclient '\\dc7\SYSVOL' --directory 'nordstrom.net\Policies\{84C43D09-B377-421E-82C2-FFFFFFFFF}' -c 'get file.xml' -k]]></ac:plain-text-body>
</ac:structured-macro>
<p> </p>
<h3>Acknowlegements</h3>
<p> </p>
<p style="margin-left: 30.0px;">RHEL 7 documentation from Red Hat</p>
<p style="margin-left: 30.0px;">
  <u>
    <a href="http://unix.stackexchange.com/questions/134577/get-a-kerberos-service-ticket-from-the-command-line">http://unix.stackexchange.com/questions/134577/get-a-kerberos-service-ticket-from-the-command-line</a>
  </u>
</p>
<p style="margin-left: 30.0px;">
  <u>
    <a href="http://www.roguelynn.com/words/explain-like-im-5-kerberos/">http://www.roguelynn.com/words/explain-like-im-5-kerberos/</a>
  </u>
</p>
<p style="margin-left: 30.0px;">Craig Barrett</p>
<p style="margin-left: 30.0px;">
  <u>
    <span style="color: rgb(0,51,102);">Kerberos: The Definitive Guide </span>
  </u>
  <span style="color: rgb(0,51,102);"> Jason Garman, O'Reilly Media, Inc. August 2003</span>
</p>
<p style="margin-left: 30.0px;">
  <span style="color: rgb(0,51,102);">
    <u>LDAP System Administration</u> Gerald Carter O'reilly Media, Inc March 2003</span>
</p>
<p style="margin-left: 30.0px;">
  <span style="color: rgb(0,51,102);">Countless web searches and weeks of confusion.</span>
</p>
<p> </p>
<p> </p>


