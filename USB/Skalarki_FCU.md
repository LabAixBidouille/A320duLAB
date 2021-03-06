# Description du protocole USB du périphérique Skalarki Flight Control Unit

La société Skalarki produit des périphérique USB pour réaliser par soi-même des cockpits pour des simulateurs d'Airbus A320. Chaque module 
correspond à l'une des parties du cockpit. 

Le FCU (Flight Control Unit) correspond à ce que l'on appelle le pilote automatique. Il permet par exemple de régler une consigne pour un cap, une vitesse ou une altitude. 

![FCU](http://www.fscockpit.eu/data/images/fcu01.jpg)

Pour commencer l'exploration des périphériques Skalarki, le FCU est un bon candidat car il comporte des entrées (bouton, intérupteur, rotacteur, ...), des 
témoins lumineux et des afficheurs.

## Typologie des échanges
Dans le logiciel *SkalarkiIO Profiler 5*, les échanges avec le périphériques sont classé en quatre catégories : 
- **Inputs**
- **ADC**
- **Outputs**
- **Displays**

Les deux premières sont des entrées et les deux dernières des sorties.

### Inputs
Les **Inputs**, comme le nom l'indique correspond aux entrées des actionneurs simples (bouton, intérupteur, rotacteur, ...). 
Leur valeur est digitale, c'est à dire que dans qu'une entrée ne peut avoir que la valeur `true` ou `false`. Pour les actionneurs 
complexes avec plus de deux positions, le logiciel utilise plusieurs entrées différentes pour coder les positions.

![](skalarki_FCU_inputs.png)

Dans le cas du module FCU, il y a au total 96 inputs différentes.

### ADC
Les **ADC** correspondent à des entrées analogiques. Généralement ce sont des potentiometres ou des capteurs de position. 
La valeur est proportionnelle à la position.
![](skalarki_FCU_ADC.png)

Le FCU n'a aucun ADC donc pour le présent module nous n'irons pas plus loin sur l'analyse de ce type d'entrées.

### Outputs
Les **Outputs**, correspondent principalement à des voyants lumineux. Leur état est binaire (allumé, éteint). 

![](skalarki_FCU_outputs.png)

Dans le cas du module FCU, il y a au total 96 outputs différentes. Ils ont la particularité d'être groupés en 6 **GROUP** de 16 sorties. Dans le logiciel *SkalarkiIO Profiler 5*, ces groupes nous permettent d'activer 16 voyants en une seule opération. Dans la partie protocole la raison de ce groupement sera plus visible.

### Displays
Les témoins lumineux ne sont pas suffisants pour représenter toutes les informations affichables sur un cockpit. 
Dans un A320, il y a des afficheurs 7 segments pour rendre visibles ces informations. 
![](skalarki_FCU_displays.png)

Le module FCU possède 24 afficheurs qui semblent eux aussi être arrangés en **GROUP** de 16 dans le logiciel. L'interaction sur ces groupes ne faisant rien, il n'est pas possible de comprendre en l'état l'utilité de cette fonctionnalité.

## Périphérique USB
Le Universal Serial Bus (USB) est une norme relative à un bus qui sert à connecter des périphériques. Le bus USB permet de connecter des périphériques à chaud (quand l'ordinateur est en marche). Apparu en 1996, ce connecteur s'est généralisé dans les années 2000 pour connecter souris, clavier d'ordinateur, imprimantes, clés USB et autres périphériques.

### Généralités
Cette norme étant très générale, elle comporte un certain nombre de niveau d'abstraction qui peuvent être déroutant en première lecture. Dans le cas du périphérique qui est le notre, on verra qu'une grande partie de cette complexité sera mise de coté car non indispensable. 

Sur le bus USB, il y a toujours un seul hôte et plusieurs périphériques.

#### Structure logique d'un périphérique USB
Dans la norme USB, un périphérique a une structure logique complexe. Il est découpé en une hiérarchie avec plusieurs niveau. Le premier niveau est le périphérique (Device). Un Device possède des configurations possibles. Chaque configuration va être associé à des Interfaces. Une interface est pour nous une sorte de périphérique logique. Pour communiquer avec ces interfaces, il est définit des point de terminaison (End point). Chaque End point est utilisé pour un et un seul type de transfert et dans un seul sens.

Pour expliquer au système ce que le périphérique est capable de faire, à chaque niveau logique est associé un type de descripteur. Cette organisation permet au périphérique physique de comporter plusieurs sous périphériques. Par exemple, pour une imprimante multifonction, un seul périphérique physique contient d'un point de vu logique un périphérique de capture d'image, un périphérique d'impression, un lecteur de cartes SD,... Dans un tel cas, l'organisation pourrait-être un simple Device, constitué d'une configuration et autant d'interfaces que de périphériques logiques. Pour chaque périphérique, des end-point correspondant à l'usage du périphérique.

#### Contrôle et configuration d'un périphérique

Avant d'aller dans les détails, l'hôte reconnaît et installe un appareil lorsque vous le branchez. Lorsque vous branchez un périphérique USB, l'hôte sait (en raison d'une astuce electronique), qu'un dispositif a été branché.

L'hôte signale une réinitialisation USB à l'appareil, afin de garantir un état connu à la fin de la remise à zéro. Dans cet état, le dispositif répond à l'adresse par défaut 0. Jusqu'à ce que le dispositif soit réinitialisé, l'hôte empêche les données d'être envoyé. Il ne réinitialise un seul appareil à la fois, donc il n'y a aucun danger que deux dispositifs puissent répondre à l'adresse 0.

L'hôte va ensuite envoyer une demande au endpoint 0, l'adresse de l'appareil 0 à savoir sa taille maximale de paquet. Il peut découvrir cela en utilisant la commande `Get Descriptor (Device)`.

En règle générale, l'hôte réinitialisent maintenant à nouveau le dispositif. Il envoie alors une demande d'adresses, avec une adresse unique. Après cette requête, l'appareil prend une nouvelle adresse. A ce stade, l'hôte est  libre de réinitialiser d'autres appareils récemment branchés.

Typiquement, l'hôte va maintenant commencer à interroger le dispositif pour obtenir autant de détails que nécéssaire. 
Pour ce faire, il va envoyer l'une des requête suivante :

- Get Device Descriptor
- Get Configuration Descriptor
- Get String Descriptor

Quand le dispositif est dans un état adressé, mais non configuré, et est autorisé à répondre aux demandes standard. Une fois que l'hôte a récupéré l'ensemble de ces données, il va charger le pilote de périphérique approprié. Le pilote de périphérique envoie une configuration à l'appareil, avec une requête `Set Configuration`. Le dispositif est maintenant dans l'état configuré, et peut commencer à être utilisé. Désormais, il peut répondre à des demandes spécifiques, en plus des demandes standards vu précédement.

Dans la norme USB, il y a quatre types de transfert différents:
- Transferts de contrôle
- Transferts d'interruption
- Transferts en vrac
- Transferts isochrones

Le seul type de transfert disponible pour un périphérique non configuré est le transfert de contrôle. 

#### Les descripteurs

##### Rôle des descripteurs 

Il existe sur le marché de nombreux périphériques USB. Les créateurs de la norme souhaitait s'adapter à un grand nombre de cas imaginable sans connaitre de manière exhaustive tous les périphériques USB existant sur terre. En plus le bus USB devant être plug and play, il fallait que le périphérique soit intérogeable de manière standard. Lorsqu'un périphérique est connecté, il doit fournir à l'hôte toutes les informations nécessaires à son identification. Ces informations sont appelé les descripteurs.

Les descripteurs sont regroupés en 4 catégorie correspondant au niveau de la structure logique d'un périphérique : 

- Device descriptor
- Configuration descriptor
- Interface descriptor 
- Endpoint descriptor
Pour mieux comprendre le but de ces descripteurs, tout en aidant à la meilleure compréhension du sujet, le FCU va être utilisé comme exemple.

## Communication avec le FCU

### Récupération des descripteurs
Sous Linux, la commande `lsusb` récupère les informations sur les périphériques USB connectés au système. Elle permet donc entre-autre de lire les différents descripteurs.

Sans argument, la commande retourne la liste des périphériques, leur position sur les bus, leur adresse et un descritif court.
 
``` sh
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 04f3:21d5 Elan Microelectronics Corp. 
Bus 001 Device 002: ID 0a5c:6410 Broadcom Corp. 
Bus 001 Device 006: ID 04d8:0050 Microchip Technology, Inc. 
Bus 001 Device 004: ID 0c45:6713 Microdia 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
Dans cet exemple, le périphérique qui nous intéresse correspon à la ligne : 

```
Bus 001 Device 006: ID 04d8:0050 Microchip Technology, Inc. 
```

Cette ligne nous indique que notre périphérique est sur bus USB n°1 et qu'il s'est vu attribuer le numéro de périphérique 6.
Une fois cette information identifiée, on peut appeler la commande `lsusb` avec les arguments suivants.

```
lsusb -D /dev/bus/usb/001/006  
Device: ID 04d8:0050 Microchip Technology, Inc. 
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            0 (Defined at Interface level)
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0         8
  idVendor           0x04d8 Microchip Technology, Inc.
  idProduct          0x0050 
  bcdDevice            0.00
  iManufacturer           1 Skalarki Electronics .
  iProduct                2 Skalarki Custom USB Device
  iSerial                 0 
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           39
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0 
    bmAttributes         0xc0
      Self Powered
    MaxPower              500mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           3
      bInterfaceClass       255 Vendor Specific Class
      bInterfaceSubClass    255 Vendor Specific Subclass
      bInterfaceProtocol    255 Vendor Specific Protocol
      iInterface              5 (error)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x01  EP 1 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0040  1x 64 bytes
        bInterval               1
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0040  1x 64 bytes
        bInterval               1
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x82  EP 2 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0040  1x 64 bytes
        bInterval               1
Device Status:     0x0001
  Self Powered
```
Le résultat de cette commande donne la liste hiérarchisée des descripteurs du périphérique. 

### Device descriptor
Le descripteur de périphérique donne toute les informations au niveau le plus général possible. A ce niveau, 
les attributs principaux sont les suivants : 

- `bLength` : Taille du descripteur en byte
- `bDescriptorType`  : Type du descripteur (0x01 pour les Device descriptor)
- `bcdUSB` : Numéro de Version USB du périphérique
- `bDeviceClass` : Classe du périphérique (si la valeur 0x00, c'est aux interface de définir la classe)
- `bDeviceSubClass` : Sous-classe de prériphérique
- `bDeviceProtocol` : Protocole utilisé
- `bMaxPacketSize` : Taille maximale des packets pour le endpoint 0
- `idVendor` : Identificateur unique du fabriquant (attribué par le consortium USB)
- `idProduct` :  Identificateur produit (choisit par le fabricant)
- `bcdDevice` : Numéro de version du périphérique
- `iManufacturer` : Indice de la chaine de description du fabricant 
- `iProduct` : Indice de la chaine de description du produit         
- `iSerial` : Indice de la chaine contenant le numéro de série
- `bNumConfigurations` : Nombre de configurations existantes pour le périphérique

Pour le FCU, voici les données associées :

```
Device: ID 04d8:0050 Microchip Technology, Inc. 
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            0 (Defined at Interface level)
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0         8
  idVendor           0x04d8 Microchip Technology, Inc.
  idProduct          0x0050 
  bcdDevice            0.00
  iManufacturer           1 Skalarki Electronics .
  iProduct                2 Skalarki Custom USB Device
  iSerial                 0 
  bNumConfigurations      1
```

La majorité des propriétés sont explicites et ne nécéssite pas plus d'explication. Pour le *Vendor Id*, on voit que le fabricant est *Microchip* et pas *Skalarki*. Ceci s'explique car en réalité la gestion USB du FCU repose un microcontroleur PIC. Le fabriquant du FCU n'a pas achété de licence au consortium USB et s'est simplement contenté de développer un périphérique spécifique à son propre besoin. C'est d'ailleurs pour cela que le driver est parfois très compliqué à faire fonctionner sous Windows.

### Configuration descriptor
La configuration est le second niveau de la hiérarchie USB. Ce niveau est le moins simple à imaginer car en définitive, très peu de périphériques ont plus d'une configuration. Pour schématiser grossièrement, un périphérique USB avec plusieurs configuration est un périphérique qui peut être vu différement en fonction des besoin. Ce qu'il faut bien comprendre, c'est qu'un périphérique a au plus une seule configuration à un instant donné.

- `bLength` : Taille du descripteur en byte
- `bDescriptorType` : Type du descripteur (0x02 pour les configuration descriptor)
- `wTotalLength` : Taille totale en byte du descripteur de configuration ainsi que tous les descripteurs subordonnés
- `bNumInterfaces` : Nombres d'interfaces associées à cette configuration
- `bConfigurationValue` : Identificateur de la configuration (à utiliser avec les requêtes `Set Configuration` et `Get Configuration`)
- `iConfiguration` : Indice de la chaine de description de la configuration
- `bmAttributes` : Information sur la gestion de l'alimentation

Pour le FCU, voici les données associées :

```
Configuration descriptor :
    bLength                 9
    bDescriptorType         2
    wTotalLength           39
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0 
    bmAttributes         0xc0
```
Pour l'exemple, le niveau configuration n'étant pas réellement utilisé, le descripteur nous apporte le stric minimum.

### Interface descriptor
Dans l'univers USB, l'interface correspond à un périphérique logique. C'est très souvent utiliser pour les périphériques (au sens Device USB) composites ou pour ceux qui ont plusieurs manières de communiquer avec l'hôte. Par exemple, un téléphone Android comporte deux interfaces, la première est celle utiliser pour la communication avec le debuger (ADB) et une seconde pour le transfert de fichier(MTP). 

C'est généralement à ce niveau qu'est définit la classe du périphérique et sa sous-classe.

- `bLength` : Taille du descripteur en byte
- `bDescriptorType` : Type du descripteur (0x04 pour les interface descriptor)
- `bInterfaceNumber` : Identifiant de l'interface
- `bAlternateSetting` : Identifiant utiliser pour retrouver les réglages alternatifs de l'interface
- `bNumEndpoints` : Nombre de End-point associés à l'interface
- `bInterfaceClass` : Classe du périphérique logique correspondant à l'interface
- `bInterfaceSubClass` : Sous-classe du périphérique logique correspondant à l'interface
- `bInterfaceProtocol` : protocole du périphérique logique correspondant à l'interface
- `iInterface` : indice de la Chaine de description de l'interface

Pour le FCU, voici les données associées :

```
Interface Descriptor:
  bLength                 9
  bDescriptorType         4
  bInterfaceNumber        0
  bAlternateSetting       0
  bNumEndpoints           3
  bInterfaceClass       255 Vendor Specific Class
  bInterfaceSubClass    255 Vendor Specific Subclass
  bInterfaceProtocol    255 Vendor Specific Protocol
  iInterface              5 (error)
```
Comme pour le niveau *configuration*, le niveau *interface* nous apporte que peu d'information sur le FCU. La première information importante est que le périphérique n'appartient à aucune classe préexistante, donc pas de possibilité de connaitre son mode de communication et surtout de pouvoir se baser sur des pilotes génériques. La seconde information importante, est que le périphérique sera constitué de 3 points de terminaison (endpoint).

### Endpoint descriptor
Le dernier niveau d'un périphérique USB est le niveau *endpoint*. Comme son nom l'indique cela correspond aux points de terminaison de l'arbre des descripteurs. Chaque endpoint correspond à un point d'entrée ou de sortie avec le périphérique. 
Les entrée, sont très généralement déclinée en deux en fonction du type de transfert choisi.

- `bLength` : Taille du descripteur en byte
- `bDescriptorType` : Type du descripteur (0x05 pour les endpoint descriptor)
- `bEndpointAddress` : Adresse du Endpoint. Le bit de poid fort de l'adresse définit le sens de communication (0: OUT, 1: IN)
- `bmAttributes` : Attributs du endpoint. Permet de définir le type de transfert (Bulk, insochrone, interruption, ...), le type de synchronisation et l'usage (Donnée ou commande) 
- `wMaxPacketSize` : Taille maximale des packets admissible par le point de terminaison 
- `bInterval` : interval de service


Pour le FCU, voici les données associées :

```
Endpoint Descriptor:
  bLength                 7
  bDescriptorType         5
  bEndpointAddress     0x01  EP 1 OUT
  bmAttributes            2
    Transfer Type            Bulk
    Synch Type               None
    Usage Type               Data
  wMaxPacketSize     0x0040  1x 64 bytes
  bInterval              1
 
```

```
Endpoint Descriptor:
  bLength                 7
  bDescriptorType         5
  bEndpointAddress     0x81  EP 1 IN
  bmAttributes            2
    Transfer Type            Bulk
    Synch Type               None
    Usage Type               Data
  wMaxPacketSize     0x0040  1x 64 bytes
  bInterval               1
```

```
Endpoint Descriptor:
  bLength                 7
  bDescriptorType         5
  bEndpointAddress     0x82  EP 2 IN
  bmAttributes            3
    Transfer Type            Interrupt
    Synch Type               None
    Usage Type               Data
  wMaxPacketSize     0x0040  1x 64 bytes
  bInterval               1
```
On voit que pour envoyer des données au FCU, il faudra passer par des écritures brutes sur le endpoint `0x01`. Pour lire l'état du FCU, il y aura deux possibilités en fonction du besoin de réactivité. La première option sera de faire une lecture brute sur le endpoint `0x81`. Cette lecture brute permet de récupérer les données sur le périphérique.

Pour lire les changements d'états du périphérique (interractions utilisateurs), il faut utiliser le mode interruption. Dans ce cas, c'est le endpoint `0x82` qui devra être utilisé.


### Description du protocole de communication
Bien que simple comme périphérique dans sa structure, le FCU dispose d'un grand nombre d'entrées/sorties. Le travail de décodage complet du protocole est donc fastidieux s'il est pris frontalement. Pour éviter cela, on va essayer d'explorer progressivement le protocole pour faire apparaitre sa logique interne et plus facilement déduire la structure des messages.

Pour ce faire, un logiciel d'analyse du protocole USB est utilisé. Il permet de voir comment le périphérique communique avec son hôte lors d'une interaction physique.

#### Inputs
Les *inputs* sont les relativement simples à tester car à chaque interaction physique il y a une interruption déclanchée.

Par exemple lors de l'appuie sur Input 008 :
```
000001: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FE EE
```
On relache Input 008 :
```
000003: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FF EE
```

Appuie sur Input 009 :
```
000005: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FD EE
```
On relache Input 009 :
```
000007: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FF EE
```
Appuie sur Input 010 :
```
000009: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FB EE
```
On relache Input 010 :
```
000011: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FF EE
```
Appuie sur Input 011 :
```
000013: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 F7 EE
```
On relache Input 011 :
```
000015: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FF EE
```
Appuie sur Input 012 :
```
000017: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 EF EE
```
On relache Input 012 :
```
000019: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FF EE
```
Chaque message reçu comporte 4 octets dans notre session d'exemple les 2 premiers octets sont toujours `0x36` et `0x41` (la notation `0x` vient du langage C dans lequel les litéraux commençants ainsi sont des nombres en hexadécimal). Dans un  premier temps, ils vont donc être ignorés. Pour mieux comprendre le protocole, les deux derniers octets vont être écrits en binaire.

- Appuie sur Input 08 : `1111 1110 1110 1110`
- Appuie sur Input 09 : `1111 1101 1110 1110`
- Appuie sur Input 10 : `1111 1011 1110 1110`
- Appuie sur Input 11 : `1111 0111 1110 1110`
- Appuie sur Input 12 : `1110 1111 1110 1110`

Tout d'abord, quand on relache un bouton poussoir, on voit que tout revient dans l'état précédent. Cet état particulier doit avoir un sens mais dans un premier temps il n'est pas rechercher.

Pour l'activation d'une entrée, il est visible qu'à chaque fois qu'une input est appuyée, l'un des bits passe de 1 à 0. Si on regarde l'indice du bit, on voit que quand l'on appuie sur l'Input `i` c'est le i eme bit qui change de valeur.

Il semble donc que chaque message donne la valeur de 16 entrée différentes. Pour vérifier cette hypothèse, appuyons sur les entrées de 8 à 12 en même temps.

Appuie sur Input 08, 09, 10, 11, 12 :
```
000017: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 E0 EE
```
On relache  :
```
000019: Bulk or Interrupt Transfer (UP), (1. Device: SKALARKI I/O GLARE) Status: 0x00000000
Pipe Handle: 0xde0790e0 (Endpoint Address: 0x82)
Get 0x4 bytes from the device
 36 41 FF EE
```
En décodant en binaire les deux derniers octets(`1110 0000 1110 1110`), l'hypothèse est vérifiée. Les bits de rang 8 à 12 sont passés à 0 comme prévu. En se basant sur ce raisonnement, les deux `0` dans le dernier octet de chaque message doivent indiquer que les entrèes 0 et 4 sont activées. Une vérification sur le logiciel *SkalarkiIO Profiler 5* le confirme et conforte notre raisonement.

Le message envoyé par le endpoint d'intérruption `0x82` est donc relativement simple à comprendre au premier abord. Comme les sorties, les entrées sont groupées par 16. Chaque groupe doit avoir une adresse qui devrait être contenue dans les deux premiers octets du message. En continuant les essais sur des entrèes supérieures à la 15, la logique d'adressage devrait apparaitre simplement s'il y en a une.

Intuitivement, en extrapolant les données déjà récupérées, les groupes devraient avoir la structure d'adressage suivante : 

| Numéro de groupe | Entrées          | Adresse    |
| -----------------| -----------------| ----------:|
| 1                | 0  - 15          | `0x36 0x41`|
| 2                | 16 - 31          | `0x36 0x43`|
| 3                | 32 - 47          | `0x36 0x45`|
| 4                | 48 - 63          | `0x36 0x47`|
| 5                | 64 - 79          | `0x36 0x49`|
| 6                | 80 - 96          | `0x36 0x4b`|

Pour vérifier si c'est bien le cas et pour s'épargner des reboots intenpestifs sous windows, on va écrire un petit programme python utilisant [PyUSB](PyUSB.md) qui récupère les données sur le Endpoint `0x82`.

``` python
#!/usr/bin/python
# -*- coding:utf8 -*-

import sys
import usb.core
import usb.util

# Connexion spécifique FCU

dev = usb.core.find(find_all=False, idVendor=0x04D8, idProduct=0x0050)
interface = 0
if dev.is_kernel_driver_active(interface) is True:
    # tell the kernel to detach
    dev.detach_kernel_driver(interface)
    # claim the device
    usb.util.claim_interface(dev, interface)

def main():
    while True:
        try:
            data = dev.read(0x82, 4, 5000)
            commande = (data[0]<<8) + (data[1] & 0xF0) 
            groupe = data[1] & 0x0F
            etat = data[2:4]
            bits =  [int(etat[i//8] & 1 << i%8 != 0) for i in range(len(etat) * 8)]
            print 'Commande : {} , Groupe : {}, Etat : {}'.format(hex(commande), groupe, bits)
        except usb.core.USBError as e:
            data = None
            if e.args == ('Operation timed out',):
                continue

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print 'Killed by user'
        # release the device
        usb.util.release_interface(dev, interface)
        # reattach the device to the OS kernel
        dev.attach_kernel_driver(interface)
        sys.exit(0)            
```

#### ADC
#### Outputs
#### Displays
