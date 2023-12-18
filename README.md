Hneď nazačiatku, aby som si zjednodušil prácu, tak som sa rozhodol, že všetko budem robiť v kontajneroch vnútri automaticky vytvoreného KVM stroja.
Tento stroj vytváram stiahnutím debian qcow2 cloud image-u, a základnou konfiguráciou cez cloud-init.
To mi malo uľahčiť prácu, lebo to umožňuje pri plnení úlohy hocikedy začať od čistého stroja. Počas riešenia úlohy som vďaka tomu narazil na viacero problémov.

1.  Nefunkčnosť cloud-init na konfiguráciu debian-genericcloud qcow2 imagu (musel som použiť debian-generic)
    Tento image neobsahuje SATA driver, vyriešiť by sa to malo dať aj použiťím IDE mechaniky.
    Problém sa týkal aj virt-install nástroja.

    https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=992119

2.  K docker kontajnerom sa ťažko zvonku pristupovalo, musel by som na VM zverejniť docker socket a použiť ansible docker connector.
    Tak som prešiel na LXC a prihlasujem sa cez SSH, VM používam ako jump-hosta, keďže nemám prístup na vnútornu LXC sieť.
    Vzhľadom na to, že táto úloha je myslená na ansible, tak je LXC pravdepodobne aj vhodnejší nástroj.

3.  Prístup k vnútorným službám tiež nie je priamo možný, niekoľko dôležitých portov som presmeroval pomocou iptables.

### Prehľad playbookov:

ansible-playbook setup_vm.yml
 - stačí ak sa pustí pod normálnym použivateľom, musí byť v libvirt skupine (nie je nutný sudo/become)
 - vytvorí VM s názvom mann-uloha
 - MAC adresa VM je nastavená pevne, to by malo zaručiť IP adresu 192.168.122.156, ak nie, tak je potrebný search&replace v inventory.yml (nenašiel som pekné riešenie)
 - všetky súbory čo sa nachádzajú na hostiteľskom systéme, vrátane VM disku, sa nachádzaju vo ./vm/

ansible-playbook destroy_vm.yml
 - odstráni VM zo systému
 - zmaže všetko vo ./vm/ okrem clean.qcow2 (čistý nekonfigurovaný debian cloud image)

ansible-playbook uloha.yml
 - vykoná obsah zadania vnútri VM pomocou LXC kontajnerov


### Prehľad kontajnerov a ich služieb:
monitoring 10.0.3.2
 - prometheus
 - prometheus-node-exporter
 - daemontools skript (spolu s logmi v /services/monitor_script)

nginx-upstream 10.0.3.3
 - nginx:80 (hostuje /var/cache/apt/archives s autoindexom)
 - nginx:8080 (stub)
 - prometheus-nginx-exporter
 - prometheus-node-exporter

nginx-upstream 10.0.3.4
 - nginx:80 (cache v /var/www/cache/)
 - nginx:8080 (stub)
 - prometheus-nginx-exporter
 - prometheus-node-exporter

etcd[1-5] 10.0.3.1[1-5]
 - etcd
 - prometheus-node-exporter 

 ##### Presmerované porty
    192.168.122.156:9090 ---> monitoring:9090   prometheus webui
    192.168.122.156:1080 ---> nginx-upstream:80
    192.168.122.156:2080 ---> nginx-cache:80

### Setup LXC kontajnerov
 - vygenerované statické DHCP priradenia podľa ansible inventára
 - do kontajnera je nainštalovaný python3 a vložený autorizovaný SSH kľúč (minimum aby sme mohli pokračovať ďalej zvnútra pomocou ansible)
 - pridanie ssh kľúčov kontajnerov do lokálneho ./vm/known_hosts

### Monitoring
 - základný prometheus bez bonusových úloh (monitorované 8x node, 5x etcd, 2x nginx, 1x prometheus)
 - veľmi jednoduchý shell skript na monitorovanie load1 a nginx_stub
 - daemontools svscan štartovaný cez systemctl
 - svscan naštartuje náš skript spolu s multilogom (logy v /services/monitor_script/log/main/)

### Nginx
 - upstream hostuje /var/cache/apt/archives/
 - cache+keepalive nastavený podľa dokumentácie nginx
 - pod /var/www/cache sa objavujú súbory
 - ss zobrazuje otvorené spojenie (keepalive vyzerá, že funguje)
 - tls som nenastavoval, ale ak áno tak by som použil https://ssl-config.mozilla.org/

 ### Etcd
 - Podľa dokumentácie je potrebných n/2+1 uzlov pre správne fungovanie clustra
 - Takže v našom prípade potrebujeme 5
 - Použil som základnú konfiguráciu z dokumentácie so static discovery
 - Spojenia nadviazané, v prometheus a logoch to vyzerá dobre
 - Ale nemám veľké skúsenosti, takže nemôžem s istotou povedať, že je to plne funkčné

### SSH
 - Aby som nezasahoval do systému, konfigurácia ssh je v ./ssh_config
 - Prilasovací kľúč bolo jednoduchšie vygenerovať, ako použiť existujúci. Nachádza sa v ./vm/id_ed25519
 - Manuálne SSH (použije náš vygenerovaný kľúč a known_hosts)

        ssh -F ssh_config root@192.168.122.156