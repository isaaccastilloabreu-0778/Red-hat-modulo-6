# Red-hat-modulo-6
comandos del modulo 6
# PRACTICA 1: CIFRADO CON GPG

1. Instalar gpg y verificar version
sudo dnf install -y gnupg2
gpg --version

2. Crear directorio y archivo
mkdir ~/practica_cifrado
cd ~/practica_cifrado
echo "Este es mi mensaje secreto" > secreto.txt
cat secreto.txt
ls -l

3. Cifrar el archivo (pide contrasena dos veces)
export GPG_TTY=$(tty)
gpg -c secreto.txt
rm secreto.txt
ls -l

4. Intentar acceder al archivo cifrado
cat secreto.txt.gpg
reset

5. Descifrar y mostrar el contenido
gpg -d secreto.txt.gpg
gpg -o secreto.txt -d secreto.txt.gpg
cat secreto.txt
# PRACTICA 2 - IPTABLES Y FIREWALL-CMD (RED HAT 10)
IP del servidor usada en las pruebas: 192.168.1.30

PASO 1: DEJAR LOS 3 SERVICIOS FUNCIONANDO
sudo dnf install -y vsftpd nmap-ncat
sudo systemctl enable --now httpd vsftpd sshd
systemctl status httpd vsftpd sshd --no-pager
sudo firewall-cmd --permanent --add-service={http,ftp,ssh}
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo ss -tlnp | grep -E ':(80|21|22)\s'
nc -4zv -w 5 192.168.1.30 21
nc -4zv -w 5 192.168.1.30 22
curl -4 --max-time 5 http://192.168.1.30
Navegador: abrir http://192.168.1.30 (debe cargar)

PASO 2: BLOQUEAR CON IPTABLES
sudo systemctl stop firewalld
sudo dnf install -y iptables-nft
sudo iptables -I INPUT 1 -p tcp --dport 80 -j DROP
sudo iptables -I INPUT 1 -p tcp --dport 21 -j DROP
sudo iptables -I INPUT 1 -p tcp --dport 22 -j DROP
sudo iptables -L INPUT -n -v --line-numbers
nc -4zv -w 5 192.168.1.30 21
nc -4zv -w 5 192.168.1.30 22
curl -4 --max-time 5 http://192.168.1.30
sudo iptables -L INPUT -n -v
Navegador: recargar http://192.168.1.30 con Ctrl + F5 (debe fallar)

PASO 3: HABILITAR DE NUEVO CON IPTABLES
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
sudo iptables -D INPUT -p tcp --dport 21 -j DROP
sudo iptables -D INPUT -p tcp --dport 22 -j DROP
sudo iptables -L INPUT -n -v --line-numbers
nc -4zv -w 5 192.168.1.30 21
nc -4zv -w 5 192.168.1.30 22
curl -4 --max-time 5 http://192.168.1.30
Navegador: recargar http://192.168.1.30 (debe cargar)

PASO 4: BLOQUEAR CON FIREWALL-CMD
sudo systemctl start firewalld
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --remove-service={http,ftp,ssh}
sudo firewall-cmd --list-all
nc -4zv -w 5 192.168.1.30 21
nc -4zv -w 5 192.168.1.30 22
curl -4 --max-time 5 http://192.168.1.30
Navegador: recargar http://192.168.1.30 (debe fallar)

PASO 5: HABILITAR DE NUEVO CON FIREWALL-CMD
sudo firewall-cmd --add-service={http,ftp,ssh}
sudo firewall-cmd --list-services
nc -4zv -w 5 192.168.1.30 21
nc -4zv -w 5 192.168.1.30 22
curl -4 --max-time 5 http://192.168.1.30
Navegador: recargar http://192.168.1.30 (debe cargar)
sudo firewall-cmd --runtime-to-permanent
# PRACTICA 3 - INSTALACION DE IDS SNORT 3 EN RED HAT 10

PASO 1 - INSTALAR REPOSITORIOS Y DEPENDENCIAS
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install -y git cmake gcc gcc-c++ make autoconf automake libtool pkgconf-pkg-config flex bison libpcap-devel pcre2-devel zlib-devel openssl-devel hwloc-devel luajit-devel libmnl-devel libnfnetlink-devel libtirpc-devel

PASO 2 - INSTALAR LIBDAQ (CAPTURA DE PAQUETES)
mkdir -p ~/snort_src && cd ~/snort_src
git clone https://github.com/snort3/libdaq.git
cd libdaq
./bootstrap && ./configure && make -j$(nproc) && sudo make install
printf "/usr/local/lib\n/usr/local/lib64\n" | sudo tee /etc/ld.so.conf.d/local.conf
sudo ldconfig

PASO 3 - INSTALAR LIBDNET (DEPENDENCIA DE SNORT)
sudo dnf install -y libdnet-devel || (cd ~/snort_src && git clone https://github.com/ofalk/libdnet.git && cd libdnet && ./configure && make -j$(nproc) && sudo make install && sudo ldconfig)

PASO 4 - COMPILAR E INSTALAR SNORT 3
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig
cd ~/snort_src
git clone https://github.com/snort3/snort3.git
cd snort3
./configure_cmake.sh --prefix=/usr/local
cd build
make -j$(nproc)
sudo make install
sudo ldconfig

PASO 5 - ENLACE PARA QUE SUDO ENCUENTRE SNORT Y VERIFICAR VERSION
sudo ln -s /usr/local/bin/snort /usr/bin/snort
snort -V

PASO 6 - CREAR LAS 4 REGLAS DE DETECCION (IP DEL HOST: 192.168.1.30)
sudo mkdir -p /usr/local/etc/snort/rules
sudo tee /usr/local/etc/snort/rules/local.rules > /dev/null <<'EOF'
alert icmp any any -> 192.168.1.30 any (msg:"ICMP - Ping detectado hacia el host"; sid:1000001; rev:1;)
alert tcp any any -> 192.168.1.30 21 (msg:"Trafico hacia puerto 21 FTP"; sid:1000002; rev:1;)
alert tcp any any -> 192.168.1.30 22 (msg:"Trafico hacia puerto 22 SSH"; sid:1000003; rev:1;)
alert tcp any any -> 192.168.1.30 80 (msg:"Trafico hacia puerto 80 HTTP"; sid:1000004; rev:1;)
EOF

PASO 7 - REVISAR EL ARCHIVO DE REGLAS
cat /usr/local/etc/snort/rules/local.rules

PASO 8 - VALIDAR LA CONFIGURACION Y LAS REGLAS
sudo snort -c /usr/local/etc/snort/snort.lua -R /usr/local/etc/snort/rules/local.rules -T

PASO 9 - EJECUTAR SNORT (DEJAR ESTA TERMINAL ABIERTA)
IFACE=$(ip route | awk '/default/ {print $5}')
echo $IFACE
sudo snort -c /usr/local/etc/snort/snort.lua -R /usr/local/etc/snort/rules/local.rules -i $IFACE -A alert_fast -k none

PASO 10 - PROBAR LAS REGLAS DESDE WINDOWS (ABRIR POWERSHELL ESCRIBIENDO: powershell)
ping 192.168.1.30
Test-NetConnection 192.168.1.30 -Port 21
Test-NetConnection 192.168.1.30 -Port 22
Test-NetConnection 192.168.1.30 -Port 80

PASO 10 ALTERNATIVO - PROBAR DESDE CMD DE WINDOWS
ping 192.168.1.30
curl --connect-timeout 3 ftp://192.168.1.30
curl --connect-timeout 3 192.168.1.30:22
curl --connect-timeout 3 http://192.168.1.30

SI CMAKE NO ENCUENTRA LIBDNET (EJECUTAR DENTRO DE ~/snort_src/snort3 EN LUGAR DEL configure_cmake DEL PASO 4)
./configure_cmake.sh --prefix=/usr/local --with-dnet-includes=/usr/local/include --with-dnet-libraries=/usr/local/lib

# PRACTICA 4: 2FA CON GOOGLE AUTHENTICATOR Y PAM PARA SSH (RHEL 10)

1. Verificar hora y sincronizacion
timedatectl
sudo systemctl enable --now chronyd

2. Instalar repositorios y Google Authenticator
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo crb enable
sudo dnf install -y google-authenticator qrencode-libs
rpm -ql google-authenticator | grep pam

3. Generar la clave del usuario (sin sudo, responder y, y, y, n, y)
whoami
google-authenticator
chmod 400 ~/.google_authenticator
ls -l ~/.google_authenticator

4. Configurar PAM
sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak
sudo sed -i '/^auth.*substack.*password-auth/a auth       required     pam_google_authenticator.so' /etc/pam.d/sshd
grep -n "auth" /etc/pam.d/sshd

5. Configurar SSH
sudo tee /etc/ssh/sshd_config.d/00-2fa.conf > /dev/null << 'EOF'
UsePAM yes
KbdInteractiveAuthentication yes
PasswordAuthentication no
AuthenticationMethods keyboard-interactive
EOF
sudo sshd -t
sudo systemctl restart sshd
sudo sshd -T | grep -Ei 'usepam|kbdinteractive|authenticationmethods|passwordauth'

6. Obtener la IP de la VM
ip -4 addr show

7. Demostracion (desde la terminal del HOST, no la VM)
ssh isaac@IP_DE_LA_VM

8. Revertir si algo falla
sudo cp /etc/pam.d/sshd.bak /etc/pam.d/sshd
sudo rm /etc/ssh/sshd_config.d/00-2fa.conf
sudo systemctl restart sshd
