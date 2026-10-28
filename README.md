# Worrell-testnet-2
<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/9de16704-2637-4ba5-abaa-6a34d8410047" />


> worrell-testnet-2 genesis ceremony'sine gentx göndererek katılmak için gayriresmi topluluk rehberi. Sunucu kurulumunu, aynı makinede başka node'larla birlikte çalışmayı, gentx oluşturmayı ve pull request açmayı anlatır. Resmi rehber https://github.com/worrellchain/networks reposundaki `worrell-testnet-2` klasöründedir ve her zaman önceliklidir. Komutlar Ubuntu sunucuda denendi. Kullanım kendi sorumluluğundadır, her şeyi mutlaka resmi rehberle karşılaştır.

## İçindekiler

- [Önemli tarihler](#önemli-tarihler)
- [Ağ parametreleri](#ağ-parametreleri)
- [Kurulum yaklaşımı](#kurulum-yaklaşımı)
- [Başlamadan önce: değişkenleri ayarla](#başlamadan-önce-değişkenleri-ayarla)
- [Adım 1: Hangi portların boş olduğunu kontrol et](#adım-1-hangi-portların-boş-olduğunu-kontrol-et)
- [Adım 2: v0.2.0 binary'sini indir](#adım-2-v020-binarysini-kendi-klasörüne-indir)
- [Adım 3: Alias, init ve pre-genesis](#adım-3-alias-oluştur-nodeu-init-et-ve-pre-genesis-dosyasını-koy)
- [Adım 4: Node'u kendi portlarına taşı](#adım-4-nodeu-kendi-portlarına-taşı)
- [Adım 5: Operatör hesabı](#adım-5-operatör-hesabı)
- [Adım 6: Gentx oluştur](#adım-6-gentx-oluştur)
- [Adım 7: Consensus key yedeği](#adım-7-consensus-key-yedeği)
- [Adım 8: Gentx'i pull request ile gönder](#adım-8-gentxi-pull-request-ile-gönder)
- [Adım 9: P2P portunu firewall'da aç](#adım-9-p2p-portunu-firewallda-aç)
- [Adım 10: client.toml'u doğru node'a yönlendir](#adım-10-clienttomlu-doğru-nodea-yönlendir)
- [Genesis yayınlandıktan sonra](#genesis-yayınlandıktan-sonra)
- [Sorun giderme](#sorun-giderme)
- [Son notlar](#son-notlar)

## Önemli tarihler

| Adım | Tarih (UTC) |
|---|---|
| Gentx penceresi açılır | 27 Ekim 2026, 10:00 |
| Gentx penceresi kapanır | 6 Kasım 2026, 10:00 |
| Final genesis yayınlanır (sha256 ve peer listesiyle) | 7 Kasım 2026 |
| Ağ başlangıcı | 10 Kasım 2026, 08:00 |
| worrell-testnet-1 kapanır | 24 Kasım 2026 |

## Ağ parametreleri

| Parametre | Değer |
|---|---|
| Chain ID | worrell-testnet-2 |
| Denom | uworrell (1 WORRELL = 1.000.000 uworrell) |
| Binary | v0.2.0 |
| Her gentx hesabına verilen miktar | 500 WORRELL |
| Self-delegation aralığı | 1 ile 500 WORRELL (önerilen 495) |
| Unbonding süresi | 3 gün |
| Minimum komisyon | %5 |
| Minimum gas price | 0.025uworrell |
| Maksimum validator | 100 |

## Kurulum yaklaşımı

Sunucunda zaten başka node'lar çalışıyorsa (worrell-testnet-1 dahil) testnet-2'yi tamamen ayrı tut, böylece mevcut hiçbir şey bozulmaz:

- Ayrı binary: `~/worrell2/bin/worrelld` (v0.2.0). Diğer node'ların kullandığı binary'nin üstüne asla yazma.
- Ayrı home: `~/.worrell-testnet-2` (testnet-1 node'u varsayılan olarak `~/.worrell` kullanır).
- Ayrı portlar: bu rehber 45xxx aralığını kullanır, ama boş olan herhangi bir aralık olur.
- Ayrı systemd servisi (genesis sonrası).

Aşağıdaki komutlar Adım 3'te oluşturulan `w2` alias'ını kullanır. Bu alias, v0.2.0 binary'si ve testnet-2 home klasörü için bir kısaltmadır.

## Başlamadan önce: değişkenleri ayarla

Yer tutucuları kendi değerlerinle değiştir:

```bash
export MONIKER="<moniker-adin>"
export KEY_NAME="<anahtar-adin>"
export SERVER_IP="<sunucunun-dis-ip-adresi>"
```

Dış IP'ni `curl -4 ifconfig.me` ile öğrenebilirsin. Değişkenler sadece açık olan terminal oturumunda yaşar, yeni bir oturum açarsan tekrar ayarla.

## Adım 1: Hangi portların boş olduğunu kontrol et

Dinlenen tüm portları ve hangi process'in kullandığını listele:

```bash
ss -tlnp | awk 'NR>1{n=split($4,a,":"); print a[n], $NF}' | sort -n | uniq
```

Kimsenin kullanmadığı bir aralık seç. Bu rehber 45xxx kullanıyor. Portların boş olduğunu teyit et:

```bash
ss -tlnp | grep -E ':(45656|45657|45658|45060|45317|45090|45091)\b'
```

Çıktı boş olmalı. Başka bir node bu aralığın bir kısmını kullanıyorsa farklı bir aralık seç ve Adım 4'ü ona göre değiştir.

## Adım 2: v0.2.0 binary'sini kendi klasörüne indir

```bash
mkdir -p ~/worrell2/bin && cd ~/worrell2
curl -L -o v0.2.0_linux_amd64.tar.gz https://github.com/worrellchain/worrell/releases/download/v0.2.0/v0.2.0_linux_amd64.tar.gz
sha256sum v0.2.0_linux_amd64.tar.gz
tar -xzf v0.2.0_linux_amd64.tar.gz -C ~/worrell2/bin
~/worrell2/bin/worrelld version
```

linux amd64 için beklenen sha256: `1ccccd39d57467236a2f64ee4809fee4de3cc3f1bc02c47e3e22aa9c6b414c76`. `version` çıktısı `v0.2.0` olmalı. Başka platformlar için resmi rehberdeki tarball ve checksum değerlerini kullan.

## Adım 3: Alias oluştur, node'u init et ve pre-genesis dosyasını koy

`~/worrell2/bin` klasörünü `PATH`'e eklemek yerine alias kullanıyoruz. PATH'i değiştirirsen sunucudaki her `worrelld` komutu v0.2.0'a gider ve diğer node'larla karışır. Alias sadece testnet-2'yi etkiler:

```bash
cat >> ~/.bashrc << 'EOF'

# worrell testnet-2
export T2_HOME=$HOME/.worrell-testnet-2
alias w2='$HOME/worrell2/bin/worrelld --home $T2_HOME'
EOF
source ~/.bashrc
w2 version
```

Şimdi node'u init et ve yayınlanan pre-genesis dosyasını indir:

```bash
w2 init "$MONIKER" --chain-id worrell-testnet-2 --default-denom uworrell
curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-2/pre-genesis.json -o $T2_HOME/config/genesis.json
sha256sum $T2_HOME/config/genesis.json
```

Beklenen sha256: `f5231dc924c762bdbe5aa0605ed6472288c9b71f4b175015e0559edccdd96b94`.

## Adım 4: Node'u kendi portlarına taşı

Bunu bir kez ve gentx'ten önce çalıştır, çünkü P2P portu gentx'in içine yazılıyor.

```bash
sed -i 's|26657|45657|; s|26656|45656|; s|26658|45658|; s|6060|45060|' $T2_HOME/config/config.toml
sed -i 's|1317|45317|; s|:9090|:45090|; s|:9091|:45091|' $T2_HOME/config/app.toml
grep -n -E "45[0-9]{3}" $T2_HOME/config/config.toml $T2_HOME/config/app.toml
```

| Amaç | Port |
|---|---|
| P2P | 45656 |
| RPC | 45657 |
| ABCI (proxy_app) | 45658 |
| pprof | 45060 |
| API | 45317 |
| gRPC | 45090 |

Grep çıktısında bu altı değeri görmelisin. Yorum satırındaki bir eşleşme sadece örnek adrestir, görmezden gelebilirsin.

## Adım 5: Operatör hesabı

Mevcut cüzdanını mnemonic ile geri yükle ya da `--recover` ifadesini çıkarıp yeni bir cüzdan oluştur ve çıkan mnemonic'i bir kenara yaz. Mnemonic'ini kimseyle paylaşma ve sadece kendi terminaline gir.

```bash
w2 keys add $KEY_NAME --recover
w2 keys show $KEY_NAME -a
w2 genesis add-genesis-account $(w2 keys show $KEY_NAME -a) 500000000uworrell
```

`add-genesis-account` sadece yerel genesis'i değiştirir ve gentx'i üretmek için gereklidir. Bu komut olmadan gentx komutu "account does not exist" hatası verir.

## Adım 6: Gentx oluştur

```bash
w2 genesis gentx $KEY_NAME 495000000uworrell --chain-id worrell-testnet-2 --moniker "$MONIKER" --commission-rate 0.05 --commission-max-rate 0.20 --commission-max-change-rate 0.01 --min-self-delegation 1000000 --ip $SERVER_IP --p2p-port 45656
```

Dikkat edilecekler:

- Self-delegation 1 ile 500 WORRELL arasında olmalı. 495 WORRELL, ücretler için birkaç WORRELL'i likit bırakır.
- `--chain-id` tam olarak `worrell-testnet-2` olmalı. Başka bir chain ID ile imzalanan gentx ağın başlamasını engeller ve genesis'ten çıkarılır.
- Komisyon oranı en az 0.05 olmalı.
- `--ip` ve `--p2p-port` gentx'in memo alanına yazılır ve peer adresin olarak yayınlanır. Diğer node'ların erişebildiği adresi ve gerçekten kullandığın P2P portunu (45656) gir.

Sonucu kontrol et:

```bash
cat $T2_HOME/config/gentx/gentx-*.json | jq '.body.messages[0] | {moniker: .description.moniker, validator_address, value}'
jq -r '.body.memo' $T2_HOME/config/gentx/gentx-*.json
```

`value` alanında `uworrell` ve `495000000` görmelisin. Memo `<node id>@<sunucu-ip-adresin>:45656` biçiminde olmalı.

## Adım 7: Consensus key yedeği

Gentx'in içindeki consensus key `priv_validator_key.json` dosyasından geliyor. Bu dosyayı kaybedersen ya da home klasörünü silersen gentx boşa gider.

```bash
cp $T2_HOME/config/priv_validator_key.json ~/priv_validator_key_testnet2.json.bak
```

Yedeği sunucu dışında güvenli bir yere de kopyala ve kimseyle paylaşma.

## Adım 8: Gentx'i pull request ile gönder

1. GitHub'da https://github.com/worrellchain/networks reposunu fork'la.
2. Kendi fork'unda **Add file, Create new file** de ve dosya adına şunu yaz: `worrell-testnet-2/gentx/<moniker-adin>.json`. Klasörler otomatik oluşur.
3. Sunucuda `cat $T2_HOME/config/gentx/gentx-*.json` çalıştır, tek satırlık JSON'u kopyala ve dosyaya yapıştır.
4. Dosya imzalı olduğu için tek karakter bozulursa geçersiz olur. Commit etmeden önce yapıştırmayı doğrulamak için sunucuda `cat > /tmp/check.json` yaz, aynı JSON'u yapıştır, Enter'a bas ve Ctrl+D ile bitir, sonra şunu çalıştır:

   ```bash
   cmp /tmp/check.json $T2_HOME/config/gentx/gentx-*.json && echo AYNI
   ```

5. Commit mesajı olarak `gentx: <moniker-adin>` kullan. Yeni bir branch'e commit edebilirsin (GitHub adını otomatik `patch-1` tarzında verir) ya da doğrudan fork'unun `main` branch'ine. İkisi de aynı sonucu verir, branch adı kontrol edilmez.
6. Önemli: karşılaştırma sayfası varsayılan olarak kendi fork'unun içinde açılır. Orada açılan pull request fork'una gider ve ekip onu görmez. **compare across forks** linkine tıkla ve base repository olarak `worrellchain/networks`, base olarak `main`, head repository olarak kendi fork'unu, compare olarak da gentx'in olduğu branch'i seç.
7. "Able to merge" yazısını, 1 commit ve 1 değişen dosyayı görmelisin. Başlığı `gentx: <moniker-adin>` yap ve **Create pull request**'e tıkla.

Repoda otomatik kontroller çalışmayabilir ve gentx dosyaları büyük ihtimalle pencere kapandıktan sonra işlenir. Resmi rehbere göre doğrulamayı geçemeyen pull request sebebiyle kapatılır ve pencere kapanmadan yenisini açabilirsin. Bir hesaptan sadece tek gentx kabul edilir, bu yüzden aynı hesap için ikinci bir pull request açma.

## Adım 9: P2P portunu firewall'da aç

```bash
ufw allow 45656/tcp
ufw status | grep 45656
```

Bir IPv4 ve bir IPv6 satırı görmelisin. Hosting sağlayıcında ayrı bir firewall paneli varsa 45656/tcp portunu oradan da aç.

## Adım 10: client.toml'u doğru node'a yönlendir

Yeni home'daki `client.toml` varsayılan olarak `localhost:26657`'ye bakar. Aynı makinedeki başka bir node varsayılan portları kullanıyorsa `w2 status` o node'un verisini döndürür. Okuma komutları zararsızdır, ama yanlış ağa transaction göndermek istemezsin. Düzelt:

```bash
sed -i 's|^node = .*|node = "tcp://localhost:45657"|' $T2_HOME/config/client.toml
sed -i 's|^chain-id = .*|chain-id = "worrell-testnet-2"|' $T2_HOME/config/client.toml
grep -n "^node\|^chain-id" $T2_HOME/config/client.toml
```

Sonra `w2 status` çalıştır. Testnet-2 node'un henüz çalışmadığı için `45657` portunda `connection refused` hatası almalısın. Başka bir ağın verisini görüyorsan ayar yanlıştır.

## Genesis yayınlandıktan sonra

1. **7 Kasım:** Final genesis'i indir ve hash'ini Telegram ve GitHub Discussions'taki duyuruyla karşılaştır:

   ```bash
   curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-2/genesis.json -o $T2_HOME/config/genesis.json
   sha256sum $T2_HOME/config/genesis.json
   ```

2. `config.toml` içinde `persistent_peers` alanına resmi rehberdeki founder peer'i ve genesis ile birlikte yayınlanan peer listesini ekle. Bu yazı hazırlanırken founder peer şuydu: `4ae3601a733c88a1b8f377774ff197dbd3e377c6@144.91.82.147:26656`. `app.toml` içinde `minimum-gas-prices = "0.025uworrell"` olsun.
3. `priv_validator_key.json` dosyasına dokunma. Gentx'in oluşturulduğu key olmalı.
4. Ayrı bir systemd servisi oluştur. Alias'lar servislerde çalışmaz, o yüzden binary'yi tam yoluyla çağır. Taslak, kendi kurulumuna göre uyarla:

   ```ini
   [Unit]
   Description=Worrell testnet-2 node
   After=network-online.target

   [Service]
   User=<kullanici-adin>
   ExecStart=/home/<kullanici-adin>/worrell2/bin/worrelld start --home /home/<kullanici-adin>/.worrell-testnet-2
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65535

   [Install]
   WantedBy=multi-user.target
   ```

5. Node'u **10 Kasım 08:00 UTC**'den birkaç dakika önce başlat. Genesis zamanından önce başlatılan node bekler, sonra diğer genesis validator'larla ilk bloğu birlikte üretir.
6. Başladıktan sonra kontrol et:

   ```bash
   w2 status | jq '.sync_info | {latest_block_height, catching_up}'
   w2 q staking validator $(w2 keys show $KEY_NAME --bech val -a)
   ```

   Validator `BOND_STATUS_BONDED` görünmeli.

## Sorun giderme

| Belirti | Neden ve çözüm |
|---|---|
| Başlangıçta `signature verification failed`, zincir ilerlemiyor | Genesis'teki bir gentx yanlış chain ID veya hesapla imzalanmış. Koordinatör onu kaldırıp genesis'i yeniden yayınlar. Yayınlanmamış genesis ile başlatma. |
| Genesis sha256 uyuşmuyor | Dosyayı repodan tekrar indir, düzenleme. |
| Gentx oluştururken `account ... does not exist` | Önce `add-genesis-account` çalıştır, aynı home ile. |
| `the provided denom ... does not match` | `init` komutunu `--default-denom uworrell` ile çalıştır. |
| `Address already in use` | Başka bir node aynı portu kullanıyor. Adım 1'deki port listesine bak. |
| `w2 status` başka bir ağın verisini gösteriyor | `client.toml` hâlâ 26657 portuna bakıyor. Adım 10'u uygula. |
| `w2 status` 45657'de `connection refused` veriyor | Testnet-2 node'un henüz çalışmıyor, genesis öncesi beklenen durum. |

## Son notlar

- Aynı zamanda bir worrell-testnet-1 node'u çalıştırıyorsan onu 24 Kasım'a kadar açık tut. Aktivitesi o tarihe kadar kaydediliyor. Home klasörüne ve binary'sine dokunma.
- Mnemonic'ini ve `priv_validator_key.json` dosyanı kimseyle paylaşma.
- Tarihler ve duyurular için Telegram grubu t.me/worrellvalidators ve GitHub Discussions takip edilmeli. Yalnızca resmi rehberdeki ve duyurulardaki saatler geçerlidir.
