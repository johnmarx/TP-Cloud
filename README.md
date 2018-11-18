# TP-Cloud-Swarm-monitoring-NFS

## vagrantFiles: configuration. 
### objectif: ajouter un disc/VM et mettre en place une connexion bridge.

pour le disque on ajoute ces lignes au vagrantFiles:
```
datadisk1 = "./data$i.vdi"
        unless File.exist?(datadisk1)
          vb.customize ['createhd', '--filename', data10, '--size', 10 * 1024]
        end
        vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 0, '--device', 0, '--type', 'hdd', '--medium', datadisk1]
```
Pour l'interface Bridge:

`ip = "192.168.1.#{i+100}"
    config.vm.network :public_network, ip: ip, bridge: "Remote NDIS based Internet Sharing Device #5"` 
## Configuration Docker Deamon

Pour ajouter la condition "--experimental" et "--metrics_addr", on a deux options: En ligne de commande au démarrage du deamon(qu'on peux automatiser via le vagrantFile) ou plus simplement en ajoutant ces conditions au fichier `/etc/docker/daemon.json `
On ajoute donc ces lignes au fichier: 

```
{
    "experimental": true,
    "metrics-addr": "0.0.0.0:9323"
}
```
+: Experimental permet d'utiliser docker stack deploy complet.
## Configuration du swarm ou orchestration.
Ont initialise la première VM Core-01 avec Swarm elle devient alors le Premier Manager :
```
* docker swarm init --advertise-addr <IP_CURRENT_VM>
```
Pour promouvoir Core-02 et Core-03, lancez cette commande sur les deux autres managers :
```
* docker swarm join-token manager --> on obtient un token pour rejoindre les managers
```
Maintenant nous ajoutons les worker avec le même principe sauf que le commande change, à exécuter sur Core-04 et Core-05
```
* docker swarm join-token worker --> un token pour joindre les workers
```
ex token: 00njd3oxv8nmjhteejaald3yzbef7osl1-ad7b1k8k3bl3aa3k3q13zivqd


On verifie `Docker node ls`
```

* ID                           HOSTNAME  STATUS  AVAILABILITY  


* 565slwe7fwequc75i35lwig54    worker3   Ready   Active 
* ad7b1k8k3bl3aa3k3q13zivqd    worker2   Ready   Active
* 25ksdma45wdiiadj45wda544f    manager3  Ready   Active        
* 5oof62fetd4gry7o09jd9e8wd    manager2  Ready   Active        
* 5pm9f2pzr8ndijqkkblkgqbsf  * manager1  Ready   Active        
* 9yq4cmfew052wp3eikj8ljjp4    worker1   Ready   Active




```
## Weawe Cloud 
weawe cloud est une interface de monitoring qui utilise weawe Scope pour lancer des conteneur sur notre hote. weawe scope est une remote. On utilise d ailleurs un Token.
## NFS
NFS est l'abréviation de Network File System. Donc Stockage et disponibilite des donner sur un reseaux,  imaginer pour les systeme linux, NFS tourne aussi sur windows et mac. Il permet de rendre disponible des donnees sur un reseaux(nfs est compatible ipv6) et de definir des regle a sa creation sur l'hote ( 0 : - - - (aucun droit)
1 : - - x (exécution)
2 : - w - (écriture)
3 : - w x (écriture et exécution)
4 : r - - (lecture seule)
5 : r - x (lecture et exécution)
6 : r w - (lecture et écriture)
7 : r w x (lecture, écriture et exécution) )

Nous pouvons configurer NFS, tout comme swarm, la config du deamon docker, directement dans le vagrantFiles pour limiter l'erreur humaine. Neamnoins, NFS est limite si les donnees sont plus lourdes que prévu.

```
yum install nfs-utils



mkdir /srv/data/



chmod -R 755 /srv/data
chown nfsnobody:nfsnobody /srv/data



systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap

cat /etc/exports <-- pour les machine clients

/srv/data   192.168.10.0/24(rw,sync,no_root_squash,no_all_squash)



systemctl restart nfs-server

firewall-cmd --permanent --zone=public --add-service=nfs
firewall-cmd --permanent --zone=public --add-service=mountd
firewall-cmd --permanent --zone=public --add-service=rpc-bind
firewall-cmd --reload`
```
NFS est donc configurer

Pour les clients, encore un peut de configuration mais cette fois avec un fichier config.ini utiliser au vagrant up:
```
{
  "ignition": {
    "config": {},
    "timeouts": {},
    "version": "2.1.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {
    "units": [
      {
        "contents": "[Unit]\nBefore=remote-fs.target\n[Mount]\nWhat=192.168.30.101:/srv/data\nWhere=/srv\data\nType=nfs\n[Install]\nWantedBy=remote-fs.target",
        "enable": true,
        "name": "var-www.mount"
      }
    ]
  }
} 
```
Lors du vagrant up, le répertoire sera crée sur nos hôtes.

## Registry disponibilité local des images

Sur core-01 on lance:
```
`docker service create --name registry --publish published=5000,target=5000 registry:2`
```
 

