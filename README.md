# TP_Linux_ESE_Cottu_Jaimes : Compte rendu

Ce ReadMe est un compte rendu complet du TP dans lequel toutes nos manipulations sont consignées étapes par étapes. Nos difficultées se trouvent à la fin du document et se situent majoritairement dans la section TP3. 

# Table des matières 
- [Mot de passe du SoC](#Mots-de-passe-de-la-carte) 
- [Commandes de la carte](#Commandes-de-la-carte) 
- [Préparation de la carte SD à mettre dans la cible](#Préparation-de-la-carte-SD-à-mettre-dans-la-cible) 
- [Découverte de la carte](#Découverte-de-la-cible)
- [TP2 Modules Kernel](#TP2-Modules-kernel)
- [TP3 Device Tree](#TP3-Device-Tree)
- [Difficultés](#Difficultés)

# Mots de passe de la carte SoC
- Login : root
- Password : rien (juste un appuis sur entrée)

# Commandes de la carte
- Reboot : Redémarer proprement la carte
- df -h : Information sur la taille occupée par les programmes
- ./expand_rootfs.sh : Etendre la zone de la mémoire allouée à la partition
- ifconfig : Infos sur la conf réseau de la carte

# Préparation de la carte SD à mettre dans la cible
On utilise le logiciel WIN32DiskImager pour flasher une image sur la carte. On peut alors allumer la carte et attendre que linux démarre. Pour pouvoir se connecter sur le système de base, on doit utiliser la liaison série car aucun autre moyen de connection n'est pour l'instant paramétré. Pour cela, on utilise la liaison UART to mini USB.   

On peut alors se connecter via une console (Putty en l'occurence) en paramétrant le bon port de communication et une vitesse de 115200 baud/sec.

## Observation des partitions mémoires

A l'aide de la commande `df -h`, on peut voir que la partition allouée à l'image n'est que de 3GB. L'image chargée précédement sur la carte fait 1.3GB, et prend donc 46% de cette place disponible.  Notre première opération va être d'augmenter la taille du système de fichiers pour profiter de l'ensemble de la carte SD. 

On utilise donc la commande `./expand_rootfs.sh`, puis on reboote la carte proprement (commande `reboot`). Une fois reconnecté, on tappe `./resize2fs_once`, puis `df -h` et on observe effectivemnt que le système de fichier dispose maintenant de 14GB soit presque l'entièreté de la carte SD. 

## Configuration réseau

On va maintenant configurer une liaison SSH sur la carte pour que l'on puisse se connecter à la SoC sans passer par une liaison série. On observe d'abord les informations réseau dont on dispose avec la commande `ifconfig`. Pour le moment, l'adresse de la carte est 192.168.88.27 . Nous allons modifier le fichier `/etc/network/interfaces`, et ajouter les lignes suivantes pour configurer la liaison SSH. Avant de faire cela, nous connectons la SoC sur un switch réseau. 

```
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d
auto eth0
iface eth0 inet dhcp
allow-hotplug eth0
```
Une fois ce fichier modifié, nous rebootons la carte, puis nous retournons voir la configuration de la carte (`ifconfig`). L'adresse de la SoC à bien changée, elle est maintenant dans le reseau de la salle et est '192.168.88.22'. En se connectant aussi au réseau de la salle, (dont le mot de passe est `ilovelinux`), on peut pinger la carte avec la commande `ping 192.168.88.22`.

## Connexion SSH

Après avoir vérifié dans le fichier `/etc/ssh/sshd_config` qu'il n'y avait pas de mot de passe pour se connecter en SSH, on peut le faire à partir de n'importe quelle console en tappant la commande `ssh root@192.168.88.22`. Nous avons pour notre part utilisé Putty. On peut ensuite fermer la liaison série. 

# Découverte de la cible

Les fichiers proc/iomem et proc/device-tree contiennent grosso-modo les mêmes informations. Elle sont juste présentées différement. Le fichier iomem est présenté selon une liste d'adresses par ordre croissants tandis que le device-tree est organisé par devices. 

## Cross Compilation depuis le PC vers la carte SoC

On utilise la puissance du PC pour compiler et ensuite téléverser le programme sur la SoC. On compile donc sur la machine virtuelle avec la ligne suivante : 

`arm-linux-gnueabihf-gcc hello.c -o hello.o` . 

On téléverse l'exécutable sur la cible avec la commande suivante :

 `scp chemin_sur_VM root@IP_DE_LA_CARTE_SOC:chemin_sur_SOC` . 

## Accès au matériel et chenillard

Comme il suffit d'écrire dans un fichier pour allumer ou éteindre une LED, on écrit un programme prennant en paramètre deux arguments, le numéro d'une LED et l'état qu'on veut lui assigner. Ce programme  ouvre le fichier correspondant, puis écrit la bonne valeur dedans sans oublier de le fermer.

```C
char file_name[80];
sprintf(file_name,"/sys/class/leds/fpga_led%d/brightness",led_number);
FILE * fid = fopen(file_name,"w+");
//gestion d'erreur d'ouverture fichier
fprintf(fid,"%d",state);
fclose(fid);
```

 Le but est ensuite de jongler entre l'appel à cette fonction et un delai dans la fonction main pour créer un chenillard.  

# TP2 Modules kernel

## Connexion SSH

Attention, pour se reconnecter à la SoC, if faut retourner voir l'adresse IP qui lui a été assignée (`ifconfig`), et se connecter à cette adresse là. 

## Re-mappage avec mmap

Nous allons créer une adresse vituelle pour nos fichier de LED afin de pouvoir les manipuler plus facilement. Pour cela, on utilise la fonction mmap(). 
```C
int fd = open("/dev/mem", O_RDWR);
//vérification ouverture fichier
p = (uint32_t*)mmap(NULL, 4, PROT_WRITE|PROT_READ, MAP_SHARED,fd, adresse_peripheral);
//vérification echec mmap
```
Avec `fd` le descripteur de fichier de la mémoire mappée, et la variable `adresse_peripheral` l'adresse du périphérique dans lequel on veut écrire. 

Ce qui est renvoyé par `mmap` est un pointeur dont la valeur est sur 8 bits et qui correspond aux registres présents à l'adresse spécifiée. Dans le cas où l'adresse du périphérique passée en argument est l'adresse des registres relatifs aux leds, il suffit de l'instruction suivante pour commander une LED. 

```C
*p = (state<<numero_LED);
```

Cette méthode est bien pour du prototypage rapide mais elle n'est néanmoins pas portable car elle requiert l'adresse du périphérique visé. 

## Compilation des modules noyau 

Pour compiler un module sur notre VM, il nous faut les sources exactes du noyau de la cible. On les charge donc avec les deux commandes suivantes : 

```
sudo apt install linux-headers-amd64
sudo apt install bc
```
On teste ensuite les commandes `insmod` (insert module), `rmmod` (remove module), `dmesg` (display messages).

On ajoute ensuite les lignes suivantes pour donner la possibilité à l'utilisateur de passer un paramètre au module. 
```C
static int param;
module_param(param, int, 0);
``` 
Dans le code d'exécution du module, on ajoute : 

```C
printk(KERN_INFO "Parametre : %d\r\n",param);
```
## Préparation à la compilation

On commence par récupérer la configuration du noyau de la SoC sur la machine virtuelle en transfèrant le fichier config.gz avec la commande suivante : 

`scp root@192.168.88.116:/proc/config.gz ~/linux-socfpga` 

Puis on décompresse ce fichier avec `gunzip config.gz` et `mv config .config` (pour renommer config en .config).

On lance ensuite les lignes suivantes depuis le fichier `/linux-socfpga/` où on a mis le fichier précédent : 

```
export CROSS_COMPILE=<chemin_arm-linux-gnueabihf->
export ARCH=arm
make prepare
make scripts
```
chemin = /usr/bin/arm-linux-gnueabihf + ajouter un tiret

Le but des deux premières lignes est de définir quelles sont les modalités de compilation du système externe (ici de la SoC). Ces lignes sont à écrire dans le terminal courant (à partir duquel on va exécuter le makefile pour la compilation croisée). 

`CROSS_COMPILE` définit l'outil de compilation du noyau sur lequel fonctionne la cible. Ici, le chemin finit par un tiret pour signifier que l'instruction s'applique à tous les compilateurs `arm-linux-gnueabihf` (en l'occurence gcc).

Pour finir, le mot clé `ARCH` permet de préciser l'architecture de la cible.  

Normalement, nous devrions passer ces arguments à la suite du mot clé "make" à chaque compilation. Pour éviter de le faire à chaque fois, on utilise le mot clé `export` pour passer ces variables en variable d'environnement. 

Cette manipulation n'est valable que pour le shell courant et doit donc être refaite pour chaque changement de shell.

## Compilation sur PC et exécution sur la carte

Avant de compiler puis d'exécuter, on modifie legèrement le makefile fournit en ajoutant les lignes suivantes : 

 ```C
 obj-m:=hello.o
KERNEL_SOURCE=/home/ensea/linux-socfpga
CFLAGS_MODULE=-fno-pic
 ```

A la compilation, tout fonctionne correctement à condition de bien avoir respecté les étapes précedentes et d'avoir bien tout recopié dans le terminal dans lequel on travaille. 

On écrit depuis la VM et le fichier se retrouve dans le dossier partagé. Or, on doit compiler depuis un autre endroit que le dossier partagé, il faut donc à chaque fois faire une copie dudit fichier à compiler. Cela est fait avec les commandes suivantes :

```C
rm -r <nom_dossier_a_suprimer>
cp -r ~/src/linux_TP2 ~/linux_TP2
```
La premiere ligne suprime le fichier écrit à la derniere compilation et la deuxième copie le nouveau. On peut ensuite écrire `make` dans le dossier contenant le module. 

Lors de l'execution sur la carte, si le message d'erreur `unknown symbol in module` apparait, l'erreur se situe probablement dans le makefile avec une mauvaise définition des étiquettes.

# TP3 : Device Tree

On va commencer par récupérer le fichier descripteur du Device Tree. On modifie la property "compatible" en "dev,ensea", puis on compile le Device Tree (.dts) en un fichier .dtb compréhensible par la SoC. On monte la partition de boot, et on vérifie que le Device Tree a bien été modifié. 

`dtc -O dtb -o soc_system.dtb soc_system.dts O majuscule`

Dans le fichier gpio_led.c, les fonctions read et write permettent de lire et d'écrire dans ce fichier. Elles intervienent lors des appels aux fonctions `echo` et `cat` en ligne de commande. 

La fonction `probe()` sert à créer une instance `platform_device()` pour chaque periphérique. Elle est appelée lors du parcours du fichier Descripteur Device Tree (.dts). A chaque nouveau périphérique, la fonction probe relative à ce périphérique sera appelée. Ici, elle va donc créer une instance pour le périphérique LED.

L'objectif de la fonction `remove()` est de faire le travail inverse de la fonction `probe()`. Lors du déchargement du module, la fonction `remove()` va être appelée pour nettoyer et supprimer l'instance du périphérique créée lors de l'appel à `probe()`.

Pour créer les fichiers de contrôle de notre chenillard dans le dossier `/proc`, il faut deja créer une structure `proc_dir_entry` et une structure `file_operations` par fichier. On doit ensuite écrire une fonction read et une fonction write par fichier. La liaison de ces deux fonctions va se faire au niveau du remplissage de la structure `file_operations` avec les deux lignes suivantes :

```C
static struct file_operations dir_fops = {
    .owner = THIS_MODULE,
    .read = dir_fops_read,
    .write = dir_fops_write,
};
```

On peut ensuite écrire dans un fichier (echo) ou lire son contenu (cat), et les fonction read et write relatives à ce fichier seront appelées. Pour cela,  il suffit d'écrire une des deux lignes suivante dans la console:

```
echo "1" > /proc/ensea/dir pour ecrire dans le fichier en question
cat .proc/ensea/dir pour lire dans le fichier 
```

Pour traduire les lignes de commandes de l'espace utilisateur vers l'espace du noyau ou inversement, on utilise les fonction `copy_to_user()` ou `copy_from_user()`. Ces deux fonctions prennent en argument un buffer de transfert, la longueur de donnée à transferer, ainsi que le buffer de l'espace du noyau.

Elles s'utilisent comme suit 

```C
static ssize_t dir_fops_read(struct file *file, char *dir_buf, size_t count, loff_t *ppos)
{
    int len = 0;
    char dir_message_read[128];
    len=sprintf(dir_message_read,"%d\n",dir_param);
    copy_to_user(dir_buf, (int *)&dir_message_read, len);
    *ppos += count;
    return len;
};
```
Ici, `dir_buf` passé en argument est la donné de l'espace du noyau que l'on transfert dans `dir_message_read` qui est dans l'espace utilisateur.

On s'attaque maintenant aux entrées dans /dev. Pour créer le fichier ensea-led, on s'inspire fortement du code de gpio-led.c qui nous est fourni. On créé donc une fonction `probe` et une fonction `remove` en plus de deux nouvelles fonction `read` et `write`. On crée une structure de `device`, puis une structure `device_id` pour indiquer quel device tree est supporté par ce driver. On recrée aussi une structure `fops` puisque c'est un nouveau fichier dans laquelle on spécifie nos deux nouvelles fonction read et write. 

Juste avant la fin du TP, nous compilons notre code qui compile sans erreurs, nous l'executons mais nous n'avons pas le temps de le tester, ni de le lier au chenillard fait précédement.  


# Difficultés

- Compréhension du Device Tree (TP3)
- Structure et liaison des fonction de lecture et d'ecriture pour les entrées dans /proc (TP3)
- Fonction copy_to_user : confusion des arguments avec les buffers servant uniquement d'intermédiaire pour le transfert entre ce qu'écrit l'utilisateur en ligne de commande et le transfert dans l'environement du noyau. (TP3) 
