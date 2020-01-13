---
title: "MAAS - Datacenter"
author: [Marc Sànchez Pifarré, GEINF (UDG-EPS)]
date: 4 de Desembre de 2019
subject: "Udg - Eps"
tags: [MAAS]
subtitle: "Tutor de la pràctica : Antonio Bueno"
titlepage: true
titlepage-color: 4a148c
titlepage-text-color: FFFFFF
titlepage-rule-height: 4
...

\newpage

# MAAS (Metal as a service)

## Context

Una empresa ISP de Girona (l'anomenarem Fibrix) vol entrar al mercat de l'emmagatzemament web i facilitar als seus clients la possibilitat de poder allotjar-hi les seves pròpies màquines virtuals. De fet aquesta empresa té l'objectiu d'unificar el domini d'una connexió bona de dades, no només venen el servei de connexió sinó també afegint els serveis de datacenter, concretament la possibilitat que una altra empresa tecnològica del sector pugui contractar amb el mateix ISP un servei de cloud. 

Fibrix ha aconseguit accés a un seguit de màquines físiques provinents d'una altra empresa cloud del sector. Aquesta altra empresa ha de mantenir un volum molt gran de feina i ha decidit invertir en Fibrix regalant-li un hardware usat i una mica antiquat. 

El hardware regalat és un hardware limitat en capacitats. Concretament ha rebut servidors [DELL POWEREDGE 860](https://www.dell.com/downloads/emea/products/pedge/es/pe860_spec_sheet.pdf) [1]. 

Els servidors que ha aconseguit Fibrix són servidors professionals, de marca propietària i cara però quant a tecnologia es podrien anomenar vellots. Tenen la possibilitat de tenir 1 processador XEON doble nucli (dual core) amb freqüència de gir màxima de 2.66GHz i memòria RAMM de fins a un màxim de 8Gb. 

Però se'n adonen que aquests servidors tenen quelcom molt bo, Compatibilitat estàndard de [BMC](https://docs.bmc.com/docs/bcm120/introducing-remote-management-570589616.html) [2] amb [IPMI](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface) [3] 1.5. Que ve a ser, gestió i control remot de la màquina sense necessitat de sistema operatiu. 

# Possibilitats en funció del hardware. 

**Opció 1**

La primera possibilitat, també la més vanal, seria pensar a muntar una màquina física per cada client. Però....

- De seguida ens quedaríem sense màquines. 
- Capacitat d'ampliació limitada per client.
- El cost de la inversió en noves màquines s'hauria de repartir als clients.

Mal negoci.

**Opció 2**

Dockerització (IAAS - Infraestructure as a service). Muntar una sèrie de servidors docker i que cada client pugui desplegar les seves aplicacions web, ja siguin html/css o qualsevol altre llenguatge i tecnologia mitjançant una interfície web d'administració. 

Aventatges:

- Docker permet crear instàncies virtuals sobre màquines físiques. 
- Docker permet crear xarxes internes i overlays entre altres dockers. 
- Docker ocupa només el que s'ha desplegat i és escalable. 
- Molts...

Docker éns aportaria escalabilitat i fins i tot ens podríem plantejar l'alta disponibilitat realitzant bé la feina. 

Inconvenients: 

Docker està pensat per què els seus nuclis de treball suportin càrregues suficients per a mantenir l'estructura de les aplicacions + les rèpliques dels demès. En el nostre cas les màquines no són gaire grosses i per tant el marge d'escalabilitat és estret i no seria possible implementar una bona qualitat de servei. 

Tanmateix el producte que es vol vendre als clients són màquines senceres, poden ser windows Server, Ubuntu, CentOs. **Docker no està pensat per suportar la integració de Sistemes operatius complets, està pensat per ser instal·lat sobre un kernel i emular-lo**.

**Opció 3**

El muntatge d'un cluster, capaç de tenir la flexibilitat d'afegir i treure màquienes físiques per poder-lo fer escalable. Ens ha de permetre la creació de màquines virtuals mitjançant imatges i el desplegament d'aquestes. Tanmateix ens ha de subministrar eines per a la comprovació de l'estat del hardware de les màquines físiques. 

Aquí entra [MAAS](https://maas.io) [4].

Una plataforma per crear datacenters basada en virtualització i feta exclusivament pensant en grans infraestructures que volen tenir la possibilitat d'escalar. La clau de l'èxit és la manera que té maas d'aprofitar la màquina física utilitzant BMC i IPMI. Les màquines no requereixen sistema operatiu i posen a disposició del hardware tots els recursos de què disposen. 

L'empresa Fibrix decideix que aquesta és la solució més viable al seu objectiu.

# Com funciona MAAS? 

Maas defineix uns conceptes importants per funcionar. 

- Region.
- Controller.
- Fabric.
- Pool. 

L'esquema seria quelcom similar a: 

![Rack and region scheme.](https://discourse.maas.io/uploads/default/optimized/1X/5fc8edb2243aa4d4ac6ba7981a7b917fec27c480_2_690x450.png){ width=350px }

**Region**

Una regió és definida com un espai físic on s'hi situen un seguit de màquines, una empresa TIER 3, per aconseguir tenir el TIER 3 està obligat a tenir rèpliques de les màquines a 3 regions diferents. És part de l'alta disponibilitat. 

L'empresa Fibrix, a causa del fet que és una start up i que encara no ha pogut desenvolupar el sistema està definida directament com una regió on hi podrà haver 1 o més controladors. 

El servidor de regió ha de ser el que gestiona les peticions d'operació. És a dir, ha de ser capaç de gestionar el DNS per tot el que es consideri regi, en definitiva és el punt de gestió del datacenter. 

**Controller**

Els controladors són els racks on hi poden haver diferents màquines físiques. Proveeix serveis d'alta capacitat de xarxa a tots els servidors del rack. En essència cobreix les necessitats que hi pugui haver en un rack de servidors físic dins d'un datacenter. 

Se n'ocupa de serveis com per exemple : 

- El wake on lan de les seves màquines. 
- L'arrencada per PXE. 
- Gestiona els serveis BMC amb IPMI o Virsh. 
- Gestiona la virtualització entre les diferents màquines. 
- Se n'encarrega del dhcp de la xarxa i manté la subxarxa. 

En general cada rack controller ha de tenir la seva subxarxa, ja sigui mitjançant el pas d'una NAT o sense. 

**Fabric** 

Totes les màquines físiques dels rack controllers es poden agrupar per generar el que s'anomenen Fabric. Cada Fabric consta d'una sèrie de màquines físiques que per simplicitat direm que es troben al mateix rack, o en general són les que estan dins d'una VLAN. 

En aquesta capa encara estem treballant amb màquines físiques i el concepte de virtualització encara no ha aparegut. 

**Pod**

Aquí apareix el concepte de virtualització, i és que, totes les màquines físiques que gestiona maas acaben component el que s'anomena un POD. Dins d'aquest pod és on es poden generar màquines virtuals. La composició i creació dels pod es pot fer o bé amb RSD (Rack Scale Design) o bé amb Virsh (Virtualització). Hi ha un apartat en aquest document on està més detallat.

# Hardware

Com a hardware tenim un seguit de servidors DELL POWEREDGE 860 amb un dual Core i Majoritàriament 4Gb de RAMM. 

El hardware de què disposem està preparat per treballar en mode cluster, definirem el mode cluster com els pc's que disposen de : 

- WAKE ON LAN 
- BMC Support
- Boot per PXE
- Virtualization support

## Utilització del hardware

MAAS necessita els fabric's, és a dir, les màquines físiques, únicament com a màquines que presten el seu hardware. No cal que tinguin cap sistema operatiu. Per fer-ho, primer de tot utilitza el wake on lan per encentre-les. Un cop s'han encés, s'ha de configurar el BMC ( Baseboard Management Controllers) [6] de la màquina, en aquest cas el sistema de Baseboard Management Controllers és IPMI versió 1.5, això ho estipula el fabricant. BMC es configura per aconseguir una ip en xarxa, per tant s'ha de permetre que la màquina arrenqui des de PXE des de la BIOS. Curiosament aquests servidors estan programats per què si hi ha hardware que està habilitat però que no està posat (un disc sense endollar per actiu a la bios), que directament no arrenqui. 

## Configuració prèvia del hardware

A l'hora d'afegir màquines al hardware s'han de prendre diferents mesures per poder garantir que el node controller té control total sobre elles. Per tant s'han de realitzar els següents passos. 

Configurar l'arrencada en xarxa des de la bios, MAAS ha de poder apagar i encendre les màquines quan ho requereixi per poder tenir un escalat més flexible. 

![PXE Configuration](images/pxe.jpeg){ width=350px }

Configurar el Wake on lan mitjançant Ctr + S, per poder arrencar-les mitjançant la interfície de xarxa es fa servir el protocol : 

![Acces WOL](images/wol-access.jpeg){ width=350px }

Activant-lo n'hi ha prou. 

![WOL](images/wol.jpeg){ width=350px }

Finalment s'ha de configurar el BMC (IPMI) accedint al seu panell amb Ctr + E. Volem que MAAS (el region controller) utilitzi les altres màquines únicament com a recursos hardware, sense fixar-nos en el sistema operatiu que porten. Com que les màquines ho permeten utilitzem IPMI però es podrien configurar mitjançant VIRSH [7], un sistema de cluster sobre virtualització. Aquest sistema sí que requereix que hi hagi un sistema operatiu a les màquines worker amb el sistema de virtualització QEMU [7] instal·lat.  

![BMC](images/bmc.jpeg){ width=350px }

# Connectivitat 

El sistema a plantejar és la màquina Rack / Region controller, que té dues interfícies de xarxa, estigui connectada amb la interfície 1 a internet (amb IP pública). El controlador de regió conté el DNS local per poder resoldre les màquines físiques i virtuals de la mateixa regió mentre que el controlador de rack conté el DCHP per poder assignar ip's a les màquines que el componen. 

Per tant des de l'exterior, cap a la xarxa del datacenter podem afirmar que hi ha un NAT. Internament, MAAS gestiona les IP's sense problemes generant subxarxes per cada Rack Controller. Aquestes subxarxes són configurables des de la interfície gràfica. 

En el nostre cas, utiltizem el rang d'adreces que conté la xarxa 192.168.1.0/24. 

La connectivitat entre el node rack i la resta de workers és mitjançant un switch ethernet. 

# Instal·lació de MAAS 

Instal·lar Maas es pot fer directament des dels paquets apt. La idea no és fer un tutorial d'instal·lació sinó veure quins són els diferents problemes que apareixen a l'hora d'intentar muntar un datacenter. 

Afegim el repositori 

```
sudo apt-add-repository -yu ppa:maas/stable
```

i Instal·lem.

```
sudo apt update
sudo apt install maas
```

En el nostre cas, per simplicitat, muntarem tant el Region com el Rack en una sola màquina. De fet és com aconsellen que es faci si no requereixes duplicar la infraestructura en diferents llocs geogràfics diferents. Si es requerís es podria muntar un Region controller en una màquina física i un Rack controller en una altra màquina física. 

Més informació a [5]. 

## Primera configuració

El primer que s'ha de fer un cop s'ha instal·lat MAAS és la inicialització. En el nostre cas li he de dir que volem que es comporti tant de region com de rack controller, això passa per fer-ho amb la següent instrucció. 

```
sudo maas init -mode all
```

Finalment, abans d'entrar a la interfície gràfica d'administració, hem de crear l'usuari root que hi tingui accés. 

```
sudo maas createadmin --username=$PROFILE --email=$EMAIL_ADDRESS
```

on : 

- $PROFILE : És el nom d'usuari
- $EMAIL_ADDRESS : és el correu electrònic de l'usuari. 

Podem inicialitzar el password mitjançant aquesta mateixa instrucció passant-li --password=$PASSWORD, o podem directament fer que el demani sense posar-lo. 

## Accés

MAAS dóna accés a la configuració mitjançant la url http://${API_HOST}:5240/MAAS on API_HOST és l'adreça de la màquina. Un cop en aquesta url iniciem sessió amb l'usuari i el password que s'han inicialitzat anteriorment. 

# Muntar el cluster mitjançant UI

MAAS realitza una exploració de les màquines que pugui haver-hi a les xarxes en les quals ell pertany, té les interfícies en mode promiscu per poder detectar màquines noves i per poder resoldre-les. De fet a la finestra de network discovery apareixen aquestes màquines. 

![Network discovery](images/discover.png){ width=350px }

S'han configurat 3 màquines utiltizant la descripció dels passos de hardware que s'han comentat anteriorment. 

Hem realitzat un filtre perquè no surtin totes les de la xarxa externa (totes les màquines de la xarxa 84.88.154.0/23). Si despleguem una d'aquestes màquines mitjançant la fletxa descendent podem veure la seva adreça mac, que necessitarem per afegir-les al cluster com a machines.

De moment no en tenim cap. 

![MAAS Empty Machines](images/machines.png){ width=350px }

Al botó add hardware de la part superior dreta de la pantalla, si el despleguem podem afegir una machine, la diferència entre machine i chassis es pot trobar a [8], Add Nodes.  Afegim un node : 

![Commissioning](images/commissioning.png){ width=350px }

I tot seguit ens salta a la finestra machines i ens mostra l'estat en el qual es troba aquesta màquina. En aquest moment està en estat Commissioning. És a dir, està comprovant els recursos de què disposa aquesta màquina worker, tot seguit quan acabi el commission, passarà els testos de hardware, principalment comprova ramm i disc. 

Veiem el llistat de coses que revisa... 

![Commissioning single image](images/commissioning1.png){ width=350px }

Un cop s'ha enllaçat la màquina, ha passat els testos i el commission llavors passa a l'estat de Ready. A partir d'aquí afegirem més màquines al cluster. Utilitzem aquest punt com un checkpoint on a partir d'ara pot ser que les màquines canviïn. 

L'estat actual en el qual es deixen les màquines és el següent : 

# Virtualització - Pods i màquines virtuals.

Un cop maas coneix les màquines, té accés a elles i pot manegar-les per arrencar-les o fer un deploy (posar-hi un ubuntu per exemple), hi ha dos tipus de tractar la virtualització de les màquines però totes passen per la creació del que s'anomena un POD. Un Pod és un servidor virtual que permet crear màquines a dins. Existeixen dues tecnologies implementades des de MAAS que permeten crear pods. 

- Virsh
- RSD (Rack Scale Design)

## Rack Scale Design 

En definitiva Rack Scale Design és una tecnologia d'intel que permet la disgregació de hardware dins d'un cluster, generant nodes que tinguin un únic propòsit. Per exemple es poden destinar nodes al manteniment de la xarxa, nodes que s'utilitzin només en termes de computació i memòria i nodes que s'utiltizin només per emmagatzematge. 

![Rack Scale design Scheme](images/rsd-scheme.png){ width=350px }

Rack Scale design ha estat alliberat per Intel actualment està en versió OpenSource. Podeu trobar l'article que ho explica a la referència [9]. Rack Scale design estandaritza l'ús de la tecnologia intel per a la pura utilització del hardware. S'utilitza el protocol Redfish [10] que permet l'estandardització de les comunicacions entre servidors mitjançant una interfície REST (HTTP).

Per poder fer servir aquest sistema cal que el hardware suporti l'arrencada per rsd i el manteniment d'aquest. El millor resum que he vist que expliquen com funciona RSD es pot trobar a [11].

## Virsh 

Virsh és un sistema de virtualització per software. Permet que els diferents nodes que componen el cluster siguin servidors de màquines virtuals. Cada node incorpora un servidor de virtualització sobre KVM [12]. Aquest sistema ens dóna una limitació, i és que cada servidor virtual només pot treballar amb màquines virtuals limitades al hardware de la mateixa màquina física, veure [13].

Hem de fer una instal·lació neta de Ubunutu a cada una d'elles per poder tenir un sistema de virtualització. És necessari que per poder virtualitzar cada un dels nodes tingui la capacitat de fer-ho i per tant ens requereix que s'instal·li un Ubuntu. Es pot instal·lar Ubuntu de manera automàtica a totes les màquines de cop mitjançant l'opció deploy. 

![Add Nodes](images/deploy.png){ width=350px }

En el nostre cas s'han creat dos servidors de virtualització anomenats eager-maggot i wanted-rabbit. 

![PODS](images/pods.png){ width=500px }

Llavors, aquestes màquines que contenen Ubuntu amb un KVM server són màquines totalment manegades per maas? No. Les màquines que formen part del cluster ara són màquines independents que s'han instal·lat mitjançant maas però són totalment accessibles i contenen un sistema operatiu complet. De fet per poder fer el login de les màquines MAAS ens permet afegir una clau pública al nostre usuari. Aquesta clau pública serà automàticament afegida com a clau d'autenticació del node i només qui contingui la clau privada podrà fer login. En el nostre cas, com que tenim un NAT hem generat un parell de claus des del node controlador i hem afegit la clau pública com a clau de l'usuari. 

D'aquesta manera quan aquest usuari faci el deploy sobre una màquina i instal·li un Ubuntu podrà directament fer login amb la seva clau privada. Per connectar-nos a la màquina que s'acaba d'instal·lar doncs utilitzem la instrucció: 

```
ssh ubuntu@192.168.1.24 -i ~/.ssh/id_rsa
```

Per defecte, l'usuari que es crea com usuari de la màquina conté el nom d'Ubuntu. Amb aquest és amb el que podem iniciar sessió. Si hem marcat la casella que permet fer un servidor de virtualització quan hem fet el deploy també tindrem accés al servidor de virtualització mitjanḉant qemu+ssh. Per exemple la següent instrucció: 

```
$ virsh -c qemu+ssh://ubuntu@192.168.1.24/system list --all
 Id    Name                           State
----------------------------------------------------

```

## Creació de màquines virtuals

En aquest moment veiem que no hi ha cap màquina virtual creada dins del servidor de virtualització. Per crear una màquina virtual, simplement cal entrar dins del pod i prèmer el botó take action -> compose. 

![new virtual machine](images/newVM.png){ width=500px }

El sistema virtual no ens permet crear una màquina virtual que tingui més hardware que la màquina host, però sí que ens permet crear més d'una màquina amb menys hardware que sumant el global continguin més hardware que el host, és a dir, podria perfectament crear : 3 màquines de 1 core cada una. Aquesta acció es permet des de la configuració del POD. Se li aplica un multiplicador tal i com s'explica a [14]. Òbviament el que no podem és encendre-les totes a la vegada. 

En aquest cas s'ha creat una màquina virtual amb 1Gb de RAMM, 1 nucli i 20Gb de ramm. Un cop creada ens apareix a la llista de machines però amb un tag diferent a les reals. 

![New virtual machine added](images/loved.png){ width=500px }

A partir d'aquest moment, aquesta màquina forma part del cluster de màquines controlables per el mateix MAAS però en comptes de ser una màquina real és una màquina virtual. Si fem la mateixa actució que amb les altres ara per exemple hi podem fer un deploy d'una imatge de CentOS 7. 

![Deploy CentOS](images/deploy-centos.png){ width=500px }

Un cop s'hagi realitzat el deploy, l'accés a aquesta màquina serà de la mateixa manera que s'ha realitzat amb les altres. Mitjançant la clau privada de l'usuari bcds en aquest cas, ja que aquest usuari hi té pujada una clau pública. 

Podem veure la màquina virtual creada entre les difrerents màquines físiques. 

![Deploy CentOS](images/centos.png){ width=500px }

Per defecte l'accés a la màquina virtual és mitjanḉant SSH, per defecte maas quan fa el deploy, el fa directament amb la clau ssh pública de l'usuari i li instal·la el ssh server. Així un cop finalitzat l'usuari podria accedir a la màquina directament com ho fan proveïdors de serveis com digital ocean. 

# Des del punt de vista de l'ISP. 

I amb tot això ja podriem assolir l'objectiu, faltaria alguna com es detallarà a continuació, però en general la funcionalitat ja estaria resolta. 

## Creació dinàmica de màquines virtuals per un usuari

El mateix ISP ara està preparat per subministrar accés a usuaris a una infraestructura on es poden crear màquines virtuals. Es pot assignar Pods a un usuari que no sigui administrador però que tingui el control de les seves màquines virtuals de la mateixa manera que es pot implementar una interfície gràfica que permeti la gestió de les accions de l'usuari. 

## Accés mitjançant API

Per accedir a realitzar accions des d'una interfície d'usuari client es pot fer a partir de la API key que té assignada cada usuari. Si es genera un usuari nou, juntament se li genera una API key per poder realitzar crides a la API Web del MAAS. Aquesta API key es pot configurar mitjançant la secció preferències de l'usuari. 

## DNS

Com a DNS el que implementa directament el region controller és un DNS local. És capaç de resoldre les seves màquines i cada cop que es genera una màquina nova també afegeix un registre A al DNS local. A partir d'aquest moment s'ha de configurar el DNS per què pugui resoldre a l'exterior. 

Un altre punt important és la propagació dels subdominis al llarg d'internet. L'ISP requereix de la creació de registres dns per cada màquina que convisquin tots sota la mateixa IP. El punt que falta per implementar seria el que s'anomena servei de Meshing. Que simplement gestiona les peticions que reben des de la WAN i les redirigeix cap a la LAN resolent de manera local quina és la màquina que té associada aquest dns. Podem veure com implementa maas el networking a la referència [15].

# Conclusions

A nivell d'aprenentatge aquest treball incorpora knowledge basat en adminsitració de sistemes, autenticació mitjançant claus públiques/privades, xarxes, patrons de disseny i virtualització. A part de tot el coneixement basat en arquitectura d'arrencades del sistema. 

En general he aprés com configurar arrencades del sistema, com fer arrencades per xarxa, wake on lan, gestió de hardware de màquines mitjançant BMC i composició de l'arquitectura client servidor d'un sistema virtual basat en KVM. 

El punt d'implementació més interessant pel que crec que es podria seguir investigant és la implementació del cluster mitjançant la eina OpenStack [16]. Open stack permet la clusterització i creació de datacenters orientats a diferents tipus de infraestructures. En realitat es basa a IAAS (Infraestructure as a service). Es pot basar sobre maas i es pot muntar a partir d'aquest mateix document, la implementació de openstack pot comportar l'us de hardware especialitzat que tingui les opcions per poder construïr un cluster amb virtualització aplicada a nivell de infraestructura i no de hardware directament. 

# Referències 

- [1]  - [Dell poweredge 860](https://www.dell.com/downloads/emea/products/pedge/es/pe860_spec_sheet.pdf)
- [2]  - [BMC](https://docs.bmc.com/docs/bcm120/introducing-remote-management-570589616.html)
- [3]  - [IPMI](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface)
- [4]  - [MAAS](https://maas.io)
- [5]  - [MAAS Installation](https://maas.io/docs/install-from-packages)
- [6]  - [BMC - IMPMI 1.5](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface)
- [7]  - [Virsh - QEMU](https://en.wikipedia.org/wiki/QEMU)
- [8]  - [Add Nodes](https://maas.io/docs/add-nodes)
- [9]  - [Intel finally releases its Rack Scale Design to Open Source](https://www.datacenterknowledge.com/archives/2016/07/29/intel-finally-releases-rack-scale-design-open-source)
- [10] - [Redfish wikipedia](https://en.wikipedia.org/wiki/Redfish_(specification))
- [11] - [Rack Scale Design: Composable Hardware][https://wiki.openstack.org/wiki/Valence#Rack_Scale_Design:_Composable_Hardware]
- [12] - [KVM Installation](https://help.ubuntu.com/community/KVM/Installation)
- [13] - [LibVirt -> tools -> virsh.pod](https://github.com/libvirt/libvirt/blob/master/tools/virsh.pod)
- [14] - [Overcommit resources](https://maas.io/docs/manage-composable-machines#heading--overcommit-resources)
- [15] - [Maas networking](https://maas.io/docs/networking)
- [16] - [OpenStack webpage](https://www.openstack.org/)