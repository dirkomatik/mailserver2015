#Mein neuer Mailserver

Hier dokumentiere ich die Installation meines neuen Mailservers im Jahr 2015 auf Basis von [Arch Linux][arch] in einer [Jiffybox][jiffybox].

Als Komponenten kommen zum Einsatz:

* [Opensmtpd](https://www.opensmtpd.org/)
* [dovecot](http://dovecot.org/)

Optional
* [Thunderbird Autokonfiguration](https://developer.mozilla.org/de/docs/Mozilla/Thunderbird/Autoconfiguration)
* [Procmail](http://www.procmail.org/)
* [Spamassassin](http://spamassassin.apache.org/)

Ich will einen Mailserver mit [Maildir][maildir]-Format und normalen Linux Benutzern.

Diese Beschreibung entstand mittels langer Recherchen, Notizen während des Einrichtung und vieles auch, nachdem ich den Server aufgesetzt habe.
Es kann gut sein, dass trotz aller Sorgfalt, Fehler in meiner Ausführungen enthalten sind.

##Grundlegendes

###Voraussetzungen für diese Anleitung sind

* Nach Anleitung von [Stefan-IT](https://github.com/stefan-it/JiffyArch) aufgesetzte [Jiffybox][jiffybox] mit [Arch Linux][arch] drauf
* Donain mit der Möglichkeit Nameservereinträge zu ändern/anzulegen
* gute Unix/Linux Kentnisse
* gute Kenntnisse im Umgang mit SSL-Zertifikaten

[arch]:     https://www.archlinux.de/
[jiffybox]: http://www.df.eu/de/cloud-hosting/cloud-server/
[maildir]:  https://de.wikipedia.org/wiki/Maildir



Für alle Befehle die im meinen Notizen aufgeführt sind, werden Root-Rechte verlangt!

Wenn von editieren die Rede ist, meine Ich das bearbeiten von Textdateien mittels eines Texteditors. Das kann sein vi, vim, emacs oder aber namo.
Nimmt den Editor, mit dem ihr am besten klar kommt.


###Lizenz

Meine Notizen stehen unter der **Creative Commons Lizenz** *Namensnennung - Weitergabe unter gleichen Bedingungen 4.0 International* **CC-BY-AS**.

![cc-by-sa-logo](https://i.creativecommons.org/l/by-sa/4.0/88x31.png "Creative Commons Lizenzvertrag")

Der Volle Text und einzelne Bedingungen können unter folgender Adresse nachgelesen werden:

http://creativecommons.org/licenses/by-sa/4.0/


###Fragen, Fehler und Anregungen

Für Fragen, Fehler und Anregungen kann gerne der Issue-Tracker von Github verwendet werden.


#Konfiguration

##Benutzer anlegen

Zunächst legen wir die Linux Benutzer an.

	useradd -m -s /bin/bash max
	useradd -m -s /bin/bash peter

Als nächstes Vergeben wir dem neuen Benutzer ein Passwort. **Und notieren uns dieses!**.

	passwd max

## SSL Zertifikat generieren

**Achtung**: Der [CommonName (CN)][cn] muss dem tatsächlichen Domainnamen gleichen, weil sonst dovecot meckert.

Dieser [Befehl][sslbefehle] legt ein Selbst-signiertes SSL-Zertifikat für 10 Jahre an.

	openssl req -x509 -nodes -days 3650 -newkey rsa:4096 -keyout /etc/ssl/private/mail.example.com.key -out /etc/ssl/certs/mail.example.com.crt

[sslbefehle]: https://www.sslshopper.com/article-most-common-openssl-commands.html
[cn]: https://icertificate.eu/wiki/Was_ist_ein_Common-Name

## DNS-Einträge vornehmen

In den DNS-Einstellungen müssen ein A, AAAA und SPF-Record gesetzt werden.

	;; ANSWER SECTION:
	example.com.		3600	IN	MX	100 mail.example.com.

	;; ADDITIONAL SECTION:
	mail.example.com.	3600	IN	A	134.xxx.xxx.xxx
	mail.example.com.	3600	IN	AAAA	2a00::FF:FF:235F


	txt "v=spf1 mx a ~all"


##Opensmtpd installieren

Jetzt wird Opensmtpd installiert

	pacman -S opensmtpd

##Opensmtpd basiskonfiguration

Zuerst editieren wir die '/etc/smptd/smtpd.conf':

	pki mail01.dirkomatik.de certificate "/etc/ssl/certs/mail.example.com.crt"
	pki mail01.dirkomatik.de key "/etc/ssl/private/mail.example.com.key"

	listen on eth0 port 25 tls pki mail.example.com
	listen on eth0 port 587 tls-require pki mail.example.com auth mask-source

	
	table vdoms             "/etc/smtpd/vdoms"
	table vusers            "/etc/smtpd/vusers"
	
	accept from any for domain <vdoms> virtual <vusers> deliver to maildir
	accept for any relay

danach legen wir die Dateien `/etc/smtpd/vdoms` und `/etc/smtpd/vusers` an.

/etc/smtpd/vdoms:

	example.com

/etc/smtpd/vusers:

	max@example.com               max
	postmaster@example.com        max
	admin@example.com             max
	abuse@example.com             max


Die Konfiguration kann man mit folgenden Befehl überprüfen

	smtpd -n

Nun aktivieren und starten wir den Mail-Dienst mit den Befehlen:

	systemctl enable smtpd

	systemctl start smtpd

###Quellen:

Für die Konfiguration habe ich Folgende Seiten quer gelesen:

* [calomel.org][calomel]
* [guillaumevincent.com][guillaumevincent]
* [coderwall.com][coderwall]

[archopensmtpd]: https://wiki.archlinux.org/index.php/OpenSMTPD
[calomel]: https://calomel.org/opensmtpd.html
[guillaumevincent]:http://guillaumevincent.com/2015/01/31/OpenSMTPD-Dovecot-SpamAssassin.html
[coderwall]:https://coderwall.com/p/eejzja/simple-smtp-server-with-opensmtpd




##Dovecot installieren

Für den Mailempfang via IMAP nehmen wir dovevot. Hierfür [installieren][archdovecot] wir:

	pacman -S dovevot

Anschließend wird die example-Konfig übernommen. Und an unsere Verhältnisse angepasst.

	cp /etc/dovecot/dovecot.conf.sample /etc/dovecot/dovecot.conf

Die Datei `/etc/dovecot/dovecot.conf` hat bei mir folgende wichtige Einträge drin:

	protocols = imap
	listen = *, ::

Nachfolgend müssen noch ein paar Konfig-Dateien mehr angefasst werden:

In `/etc/dovecot/conf.d/10-auth.conf` steht bei mir:

	auth_mechanisms = plain

In `/etc/dovecot/conf.d/10-mail.conf` steht bei mir:

	mail_location = maildir:~/Maildir

In `/etc/dovecot/conf.d/10-ssl.conf` steht bei mir:

	ssl = yes
	ssl_cert = </etc/ssl/certs/mail.example.com.crt
	ssl_key = </etc/ssl/private/mail.examole.com.key

In `/etc/dovecot/conf.d/15-lda.conf` steht bei mir:

	postmaster_address = postmaster@example.com

Das müssten nun alle von mir angepassten Konfigurationsdateien sein.


[archdovecot]: https://wiki.archlinux.org/index.php/Dovecot

#Optionale Konfiguration

##Thunderbird Autokonfiguration

Thunderbird versucht beim Einrichten eines bestehenden Mailkontos die richtigen Maileinstellungen automatisch zu hinterlegen,
oder aber zu [erraten][thauto]. Das Raten klappt aber meist eher nicht so gut.

Bei der Einrichtung schaut Thunderbird erst in seiner Datenbank nach. Anschließend wird per http folgende Adresse abgefragt:

	http://example.com/.well-known/autoconfig/mail/config-v1.1.xml

Wenn man also einen Webserver auf dieser Domain betreibt, kann man entsprechend den Pfad anlegen und die Datei `config-v1.1.xml` anlegen:

```xml
<?xml version="1.0" encoding="UTF-8"?>
	
<clientConfig version="1.1">
  <emailProvider id="example.com">
    <domain>example.com</domain>
    <displayName>example.com Mailserver</displayName>
    <displayShortName>example.com</displayShortName>
    <incomingServer type="imap">
      <hostname>mail.example.com</hostname>
      <port>143</port>
      <socketType>STARTTLS</socketType>
      <authentication>password-cleartext</authentication>
      <username>%EMAILLOCALPART%</username>
    </incomingServer>
    <outgoingServer type="smtp">
      <hostname>mail.example.com</hostname>
      <port>587</port>
      <socketType>STARTTLS</socketType>
      <authentication>password-cleartext</authentication>
      <username>%EMAILLOCALPART%</username>
    </outgoingServer>
    <documentation url="https://mail.example.com/mailserver">
      <descr lang="de">Allgemeine Beschreibung der Einstellungen</descr>
      <descr lang="en">Generic settings page</descr>
    </documentation>
  </emailProvider>
</clientConfig>
```

Das Dateiformat ist [hier][thautofile] beschrieben.

[thauto]: https://developer.mozilla.org/de/docs/Mozilla/Thunderbird/Autoconfiguration#Configuration_server_at_ISP
[thautofile]: https://developer.mozilla.org/en-US/docs/Mozilla/Thunderbird/Autoconfiguration/FileFormat/HowTo

##Procmail

Procmail ist ein Mailfilter, mit deren Hilfe wir unsere Mails filtern können. Zusätzlich kann mit Procmail auch Spamassassin eingebunden werden.

Zuerst installieren wir [Procmail][archproc] mit:

	pacman -S procmail

Danach legen wir die [globale Konfiguration][doveproc] an `/etc/procmailrc` an. Dies auskommentierten Zeilen sind später für Spamassassin.

	SHELL="/bin/bash"
	SENDMAIL="/usr/sbin/sendmail -oi -t"
	#LOGFILE="$HOME/procmail.log"
	DELIVER="/usr/lib/dovecot/deliver"
	DROPPRIVS=yes
	# fallback:
	DEFAULT="$HOME/Maildir/"
	MAILDIR="$HOME/Maildir/"
	
	# Spamassassin
	#:0fw: spamassassin.lock
	#* < 256000
	#| /usr/bin/vendor_perl/spamassassin


Zusätzlich verwende ich noch eine lokale Procmail-Datei `/home/username/.procmailrc` in meinem Homeverzeichnis.

	SHELL="/bin/bash"
	SENDMAIL="/usr/sbin/sendmail -oi -t"
	LOGFILE="$HOME/procmail.log"
	DELIVER="/usr/lib/dovecot/deliver"
	DROPPRIVS=yes
	# fallback:
	DEFAULT="$HOME/Maildir/"
	MAILDIR="$HOME/Maildir/"
	
	* ^X-Spam-Level: \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*
	| $DELIVER -m Junk
	
	:0 w
	* ^X-Spam-Status: Yes
	| $DELIVER -m Junk
	
	# Mailinglisten
	:0 w
	* ^(To:|Cc:).*Foo@lists.hs-offenburg.de
	| $DELIVER -m Foo
	
	
	# Default verhalten
	:0 w
	| $DELIVER

Für den Aufbau der Regeln bitte die Quellen konsultieren.

Damit unser Mailsystem nun Procmail verwenden kann, ist eine Zeile in unserer Opensmtpd-Konfiguration `/etc/smtpd/smtpd.conf` wie folgt zu ändern:

	#accept from any for domain <vdoms> virtual <vusers> deliver to maildir
	accept from any for domain <vdoms> virtual <vusers> deliver to mda "/usr/bin/procmail"

Die Änderung wird nach einem Neustart wirksam.

	systemctl restart opensmtpd

[doveproc]: http://wiki2.dovecot.org/procmail
[archproc]: https://wiki.archlinux.org/index.php/Procmail

##Spamassassin

Spamassasin wird wie die anderen Komonenten auch über Pacman installiert.

	pacman -S spamassassin

Nach der Installation muss Spamassassin auf stand gebracht werden.

	sa-update

Zuerst muss Spamassassin konfiguriert werden. In meiner Konfiguration `/etc/mail/spamassassin/local.cf` steht drin:

	rewrite_header Subject *****SPAM*****
	report_safe 1

Nun passen wir die globale Konfiguration von Procmail `/etc/procmailrc` an und kommentieren die [Spamassesin Abschnitt][spamproc] raus:

	# Spamassassin
	:0fw: spamassassin.lock
	* < 256000
	| /usr/bin/vendor_perl/spamassassin


[spamproc]: https://wiki.apache.org/spamassassin/UsedViaProcmail
