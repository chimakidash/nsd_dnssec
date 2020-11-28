# nsd_dnssec
## clone this repo
```bash
git clone ..
```
## Run this service
- Run it
```bash
docker-compose up -d
```
- Generating DNSSEC keys and signed zone
```bash
docker-compose exec nsd keygen domain.tld

Generating ZSK & KSK keys for 'domain.tld'
Done.
```

- Then sign your dns zone (default expiration date is 1 month): 
```bash
docker-compose exec nsd signzone domain.tld

Signing zone for domain.tld
NSD configuration rebuild... reconfig start, read /etc/nsd/nsd.conf
ok
Reloading zone for domain.tld... ok
Notify slave servers... ok
Done.
```

- how to update the serial and your TLSA record (if you have one) programmatically
- modfy cron scripts(TBD)

```bash
#!/bin/bash

LETS_ENCRYPT_LIVE_PATH=/path/to/your/lets/encrypt/folder
fingerprint=$(openssl x509 -noout -in "${LETS_ENCRYPT_LIVE_PATH}/cert.pem" -fingerprint -sha256 | cut -c 20- | sed s/://g)

domain="domain.tld"
zonename="db.${domain}"
zonefile="/mnt/docker/nsd/zones/${zonename}"
serial=$(date -d "+1 day" +'%Y%m%d%H')
tlsa_line_number=$(grep -n TLSA $zonefile | cut -d : -f 1)
tlsa_dns_record="_dane IN TLSA 3 0 1 ${fingerprint}"
expiration_date=$(date -d "+6 months" +'%Y%m%d%H%M%S')

sed -i -e "s/20[0-9][0-9]\{7\} ; Serial/${serial} ; Serial/g" \
       -e "${tlsa_line_number}s/.*/${tlsa_dns_record}/" $zonefile

if docker exec nsd nsd-checkzone "$domain" /zones/"$zonename" | grep -q "zone ${domain} is ok"; then
  docker exec nsd signzone "$domain" "$expiration_date"
fi
```

 - Show your DS-Records (Delegation Signer):
```bash
docker-compose exec nsd ds-records domain.tld

> DS record 1 [Digest Type = SHA1] :
domain.tld. 600 IN DS xxxx 14 1 xxxxxxxxxxxxxx

> DS record 2 [Digest Type = SHA256] :
domain.tld. 600 IN DS xxxx 14 2 xxxxxxxxxxxxxx

> Public KSK Key :
domain.tld. IN DNSKEY 257 3 14 xxxxxxxxxxxxxx ; {id = xxxx (ksk), size = 384b}
```
- Restart the DNS server to take the changes into account:
```bash
docker-compose restart nsd
```

---