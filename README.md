
<style>
img {
    max-width: 50%;
    margin: auto;
}
</style>

# MAAS (Metal as a service)

## Context

Una empresa ISP de girona (l'anomenarem Fibrix) vol entrar al mercat de l'emmagatzemament web i facilitar als seus clients la possibilitat de poder allotjar-hi les seves pròpies màquines virtuals. De fet aquesta empresa té l'objectiu de unificar el domini de una connexió bona de dades, no només venen el servei de connexió sinó també afegint els serveis de datacenter, concretament la possibilitat que una altre empresa tecnològica del sector pugui contractar amb el mateix ISP un servei de cloud. 

Fibrix ha aconseguit accés a un seguit de màquines físiques provinents de una altre empresa cloud del sector. Aquesta altre empresa ha de mantenir un volum molt gran de feina i ha decidit invertir en Fibrix regalant-li un hardware usat i una mica anticuat. 

El hardware regalat és un hardware limitat en capacitats. Concretament ha rebut servidors [DELL POWEREDGE 860](https://www.dell.com/downloads/emea/products/pedge/es/pe860_spec_sheet.pdf) [1]. 

Els servidors que ha aconseguit Fibrix son servidors professionals, de marca propietària i cara peró en quan a tecnologia es podrien anomenar vellots. Tenen la possibilitat de tenir 1 processador XEON doble nucli (dual core) amb freqüència de gir màxima de 2.66GHz i memòria RAMM de fins a un màxim de 8Gb. 

Però se n'adonen que aquests servidors tenen quelcom molt bo, Compatibilitat estàndard de [BMC](https://docs.bmc.com/docs/bcm120/introducing-remote-management-570589616.html) [2] amb [IPMI](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface) [3] 1.5. Que ve a ser, gestió i control remot de la màquina sense necessitat de sistema operatiu. 

# Possibilitats en funció del hardware. 

**Opció 1**

La primera possibilitat, també la més vanal, seria pensar en muntar una màquina física per cada client. Peró....

- De seguida ens quedariem sense màquines. 
- Capacitat d'ampliació limitada per client.
- El cost de la inversió en noves màquines s'hauria de reperectir als clients.

Mal negoci.

**Opció 2**

Dockerització (IAAS - Infraestructure as a service). Muntar una sèrie de servidors docker i que cada client pugui desplegar les seves aplicacions web, ja siguin html/css o qualsevol altre llenguatge i tecnologia mitjançant una interfície web d'administració. 

Aventatges :

- Docker permet crear instàncies virtuals sobre màquines físiques. 
- Docker permet crear xarxes internes i overlays entre altres dockers. 
- Docker ocupa només el que s'ha desplegat i és escalable. 
- Molts...

Docker éns aportaria escalabilitat i fins i tot ens podriem plantejar la alta disponibilitat realitzant bé la feina. 

Inconvenients : 

Docker està pensat per què els seus nuclis de treball suportin càrregues suficients com per mantenir l'estructura de les aplicacions + les rèpliques dels demés. En el nostre cas les màquines no son gaire grosses i per tant el marge d'escalabilitat és estret i no seria possible implementar una bona qualitat de servei. 

Tanmateix el producte que es vol vendre als clients son màquines senceres, poden ser windows Server, Ubuntu, CentOs. **Docker no està pensat per suportar la integració de Sistemes operatius complerts, està pensat per ser instal·lat sobre un kernel i emular-lo**.

**Opció 3**

El muntatge d'un cluster, capaç de tenir la flexibilitat d'afegir i treure màquienes físiques per poder-lo fer escalable. Ens ha de permetre la creació de màquines virtuals mitjançant imatges i el desplegament de les mateixes. Tanmateix ens ha de subministrar eines per a la comprovació de l'estat del hardware de les màquines físiques. 

Aquí entra [MAAS](https://maas.io) [4].

Una plataforma per crear datacenters basada en virtualització i feta exclusivament pensant en grans infraestructures que volen tenir la possibilitat d'escalar. La clau de l'èxit és la manera que té maas d'aprofitar la màquina física utiltizant BMC i IPMI. les màquines no requereixen de sistema operatiu i posen a disposició del hardware tots els recursos dels que disposen. 

L'empresa Fibrix decideix que aquesta és la solució més viable al seu objectiu.

# Com funciona MAAS? 

Maas defineix uns conceptes importants per funcionar. 

- Region.
- Controller.
- Fabric.
- Pool. 

L'esquema seria quelcom similar a: 

![Rack and region scheme.](https://discourse.maas.io/uploads/default/optimized/1X/5fc8edb2243aa4d4ac6ba7981a7b917fec27c480_2_690x450.png)

**Region**

Una regió és definida com un espai físic on s'hi situen un seguit de màquines, una empresa TIER 3, per aconseguir tenir el TIER 3 està obligat a tenir rèpliques de les màquines a 3 regions diferents. És part de la alta disponibilitat. 

L'empresa Fibrix, degut a què és una start up i que encara no ha pogut desenvolupar el sistema està definida directament com una regió on hi podrà haver 1 o més controladors. 

El servidor de regió ha de ser el que gestiona les peticions d'operació. És a dir, ha de ser capaç de gestionar el DNS per tot el que es consideri regi, en definitiva és el punt de gestió del datacenter. 

**Controller**

Els controladors son els racks on hi poden haver diferents màquines físiques. Proveeix serveis d'alta capacitat de xarxa a tots els servidors del rack. En essència cobreix les necessitat que hi pugui haver en un rack de servidors físic dins d'un datacenter. 

Se n'ocupa de serveis com per exemple : 

- El wake on lan de les seves màquines. 
- L'arrencada per PXE. 
- Gestiona els serveis BMC amb IPMI o Virish. 
- Gestiona la virtualització entre les diferents màquines. 
- Se n'encarrega del dhcp de la xarxa i manté la subxarxa. 

En general cada rack controller ha de tenir la seva subxarxa, ja sigui mitjançant el pas d'una NAT o sense. 

**Fabric** 

Totes les màquines físiques dels rack controllers es poden agrupar per generar el que s'anomenen Fabric. Cada Fabric consta de una sèrie de màquines físiques que per simplicitat direm que es troben al mateix rack, peró que no ho he pogut verificar encara. 

En aquesta capa encara estem treballant amb màquines físiques i el concepte de virtualització encara no ha aparegut. 

**Pool**

Aquí apareix el concepte de virtualització, i és que, totes les màquines físiques que gestiona maas acaben component el que s'anomena una piscina de recursos. Dins d'aquesta piscina és on es poden generar màquines virtuals. 



# Referències 

- [1] - [Dell poweredge 860](https://www.dell.com/downloads/emea/products/pedge/es/pe860_spec_sheet.pdf)
- [2] - [BMC](https://docs.bmc.com/docs/bcm120/introducing-remote-management-570589616.html)
- [3] - [IPMI](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface)
- [4] - [MAAS](https://maas.io)