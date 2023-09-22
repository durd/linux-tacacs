# TACACS login for Linux servers

This documentation is only for Debian and related distros, ie Ubuntu. This instruction is written from view of an Ubuntu 20.04 Focal Fossa.

## The problems

SSH kollar själv om en användare finns eller inte mha NSS. Och om SSH inte får att användaren redan finns så skickas SSH till PAM att lösenordet är dåligt. OAVSETT vad PAM säger sen.

Detta innebär tre alternativ, vi skapar användaren, oavsett vad PAM/tacplus säger. Detta leder dock lätt till en väldigt långsam DDoS då användar-databasen och `/home/` fylls med användare som faktiskt inte finns, baserat på felskrivningar när man loggar in.  
Alternativ två är att en annan användare lägger till användaren manuellt. Detta innebär dock extra-jobb för Managed Services eller Consulting som måste komma ihåg rutiner, och lägga till användaren i alla servrar denna ska ha access till.
Det tredje alternativet är låta simulera att en användare finns, oavsett vad. Fördelen med denna metod är att SSH låter PAM göra sitt, och vi kan utifrån svaret från PAM/tacplus lägga till användaren med hemkatalog och grupper. Blir svaret nekande från PAM/tacplus så avbryts processen så att användaren inte läggs upp på servern.  
En nackdel är att man blir bortkopplad från servern första gången, detta för att det blir ett moment 22 där du loggar in med en användare som inte finns förän du loggar in.  
En acceptabel nackdel i mina ögon. Användaren behöver bara logga in igen.

## Prerequisites

Förutom Debian eller t ex Ubuntu så förutsätter nedan klipp och klistra en del saker:
* sed:
  * Att `/etc/pam.d/sudo` och `/etc/pam.d/sshd` ser ut som dom ska för att `sed` ska göra sitt
  * Gäller även `nsswitch.conf`
  * sed går att ändra och anpassa

Finns paket att installera istället för git-clone, använd paket istället. Det överlever uppgraderingar bättre.

## Features & limitations

Features:
* TACACS+ authentication, authorization, accounting
* SSH pubkey login
* Local user and password login

Limitations:
* None as far as I know

### libnss-ato

Kräver att en användare finns lokalt, specifikt en användare med samma UID som i `libnss-ato.conf`. Vi lägger upp system-kontot `tac_helper` och läser in dess `passwd` till `libnss-ato.conf`.

## Chain of events

Följande händer för en användare som inte finns sedan tidigare på servern.
1) Användaren försöker logga in (med rätt lösenord) till servern
   1) `NSS` med `libnss-ato` berättar för SSH att användaren finns
   2) SSH låter PAM autentisera användaren, i vårt fall med TACACS+
   3) Om TACACS ger OK, så körs ett script mha `pam_exec.so`
      1) Detta script tar bort `libnss-ato` konfen från `nsswitch.conf`. Annars kan inte användaren läggas till.
      2) Användaren skapas lokalt, utan lösenord eller person-info
      3) Användaren läggs med i grupperna `adm, sudo`
      4) `libnss-ato` konf läggs till igen i `nsswitch.conf`
   4) SSH försöker logga in användaren, men då SSH-sessionen redan börjat så failar den pga att användaren inte fanns då
      1) Användaren blir frånkopplad
   5) Användaren loggar in igen, och blir inloggad
2) Användaren försöker bli sudo
   1) `PAM/tacplus_sudo` körs och där räcker det att användaren autentiseras från TACACS-servern för att bli `sudo`

## Packages

```
sudo -E bash
apt update
apt install build-essential dh-make p7zip p7zip-full p7zip-rar unzip
git clone https://github.com/donapieppo/libnss-ato.git
```

## Installation

Glöm inte att ändra `<serverip>` och `<secret>` till dina värden.

```
cd libnss-ato/
fakeroot debian/rules binary
cd ../
dpkg -i libnss-ato_0.2-1_amd64.deb
adduser --system tac_helper
grep tac_helper /etc/passwd > /etc/libnss-ato.conf

cat <<EOF > /usr/sbin/pam_verify
#!/usr/bin/env bash

#echo "User: ${PAM_USER}; Password: ${PAM_PASSWORD}; Type: ${PAM_TYPE}; Service: ${PAM_SERVICE}; AVP: ${PRIV_LVL}"
# echoing env was a lot simpler, still not sure if all variables are printed...
#env

# Only if PAM_TYPE and PAM_SERVICE is correct, and the homedir already doesn't exist
# do we do the following:
# * Remove libnss-ato config from nsswitch.conf
# * Add user without password and no name information
# * Add user to groups sudo and adm
# * Adds libnss-ato config back to nsswitch.conf
if [[ "${PAM_TYPE}" == "auth" ]] && [[ "${PAM_SERVICE}" == "sshd" ]] && [ ! -d "/home/${PAM_USER}" ]; then
  #useradd -m -s /bin/bash ${PAM_USER}
  sed -e 's/^passwd:.*ato/passwd:         files systemd/' -e 's/^shadow:.*ato/shadow:         files/' -i /etc/nsswitch.conf
  adduser --disabled-password --gecos "" ${PAM_USER}
  adduser ${PAM_USER} sudo
  adduser ${PAM_USER} adm
  sed -e 's/^passwd:.*systemd/passwd:         files systemd ato/' -e 's/^shadow:.*files/shadow:         files ato/' -i /etc/nsswitch.conf
fi
EOF

chmod u+x /usr/sbin/pam_verify

cat <<EOF > /etc/pam.d/tacplus
auth       requisite pam_tacplus.so server=<serverip> secret=<nyckel>
#auth       sufficient pam_exec.so log=/tmp/debug.log /usr/sbin/pam_verify debug
auth       sufficient pam_exec.so /usr/sbin/pam_verify
account    required pam_tacplus.so server=<serverip> secret=<nyckel> service=ppp protocol=ip
session    required pam_tacplus.so server=<serverip> secret=<nyckel> service=ppp protocol=ip
EOF

cat <<EOF > /etc/pam.d/tacplus_sudo
auth       sufficient pam_tacplus.so server=<serverip> secret=<nyckel> prompt=Password:
EOF

sed -e 's/^@include common-auth/@include tacplus_sudo\n@include common-auth/' -i /etc/pam.d/sudo
sed -e 's/^# PAM configuration for the Secure Shell service/# PAM configuration for the Secure Shell service\n@include tacplus/' -i /etc/pam.d/sshd
```
