# Webservice de petició de treballs per a ajuntaments

PeticioTreballs és un web service que permet demanar treballs en diferit i recuperar els fitxers que genera. 

Cal accedir al webservice mitjançat l'endpoint REST. L’especificació es pot obtenir del [fitxer swagger disponible en aquest repositori](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/swagger%20PeticioTreballsREST.json).

El servei es troba a les següents URL's:
* Producció: https://wsreal.orgt.diba.cat/PeticioTreballsREST
* Pre-produció: https://wsproves.orgt.diba.cat/PeticioTreballsREST

Respecte a la seguretat, cal tenir en compte:
1. L’accés al webservice serà via https amb certificat:
   - A producció s'accedeix al webservice mitjançant certificat d'Òrgan a nom de l'Ajuntament. Concretament, el certificat ha de tenir el CIF de l'Ajuntament com a valor del camp VinculatedCompanyCIF. A producció es comprovarà que les dades enviades corresponen a l’Ajuntament corresponent al certificat. 
   - A pre-producció es pot accedir al webservice amb un certificat que no sigui d'òrgan. 
1. **Previ a les proves cal comunicar el certificat utilitzat a l’ORGT ja que és necessari instal·lar la clau pública als servidors de la ORGT.** Vegeu el procés de sol·licitud a la [pàgina principal](https://github.com/organisme-de-gestio-tributaria/organisme-de-gestio-tributaria). És necessari fer-ho amb el certificat que s'utilitzarà tant a pre-producció com a producció.
1. A més del certificat, cal utilitzar autenticació bàsica amb l'usuari i password que l'ajuntament utilitza a la ORGT per a tots els endpoints excepte /treballs/nomTreball GET  i  /municipis. 

Els endpoints disponibles són:

1. **treballs/{nomTreball} GET** Obté les dades necessàries per tal de crear una petició d'un treball. L'estructura retornada es pot fer servir, omplint els atributs ValorParametre, per demanar el treball amb el corresponent endpoint. En cas d'error, es retorna una instància de "Retorn" informada amb el CodiRetorn i DescripcioError. Només es poden obtenir les dades de treballs pensats per a Ajuntaments, és a dir, que tenen almenys un paràmetre amb el codi de l'Ajuntament.
    
    Aquest endpoint només té un paràmetre en el path:
    - nomTreball. (8 caràcters màxim). Exemple: xbentri
     
    Retorna codi http 400 quan el nom de treball és incorrecte. <br>
    Aquest endpoint juntament amb el de municipis no requereix autenticació bàsica. Sí requereix identificar-se amb un certificat d'òrgan prèviament compartit amb l'ORGT.


1. **treballs/{nomTreball} POST** Realitza la petició d'executar un treball en diferit proporcionant les dades necessàries per a l'execució. Les dades de la petició es poden obtenir amb el endpoint anterior i emplenant els ValorParametre. Si ha hagut un error en demanar la petició, retorna informada l'estructura amb el CodiRetorn i DescripcioError. La petició s’entén feta per l’Ajuntament associat al certificat amb el qual s’accedeix (que és el mateix que el proporcionat en l'autenticació bàsica). Només es poden demanar treballs pensats per a Ajuntaments, és a dir, que tenen almenys un paràmetre amb el codi de l'Ajuntament. Cal emplenar tots els paràmetres, cap d'ells no pot ser null, si és opcional cal posar "".
    
    Aquest endpoint té com a paràmetres:
   - Paràmetre del path: nomTreball. (8 caràcters màxim). Exemple: xbentri
   - Body: dadesPeticio. Petició amb les dades, [podeu veure un exemple aquí](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%201%20peticio%20treball%20POST.json). Les dades de la petició es poden obtenir amb el endpoint anterior i emplenant els ValorParametre. Les dades consisteixen en el nom del treball, el codi de client que identifica l'ajuntament i els paràmetres del treball juntament amb el seu valor. Podeu obtenir una llista dels codis de client i el nom del municipi a través de l'endpoint /municipis.

    Codis d'error de retorn:
    - 400: Cal proporcionar el nom del treball i les dades de la petició       
    - 401: Falta autenticació, autenticació incorrecta o no autoritzat pel treball demanat
    - 404: El client no existeix  
    - 406: El client correspon a un municipi no adherit
    - 500: Error en la gestió de la petició
   
    Quan s'ha donat d'alta la petició, es retorna codi 200 amb el número de petició, [podeu veure un exemple aquí](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%201%20resposta%20amb%20num%20peticio.json).


1. **treballs/{nomTreball}/peticio/{idPeticio} GET** Obté l'estat d'execució d'una petició. Els estats poden ser "Pendent", "Finalitzat" o "Erroni". Si el treball ha finalitzat i ha generat fitxers, es retorna també el nom de tots els fitxers generats pel treball. Quan el treball està "Pendent", vol dir que encara no s'ha iniciat la execució o que s'està executant. Es comprova que només es poden consultar peticions fetes pel mateix Ajuntament que consulta el resultat, d’acord amb el certificat digital aportat en fer la consulta. 
    
    Aquest endpoint té dos paràmetres al path:
   - nomTreball. (8 caràcters màxim). Exemple: xbentri
   - idPeticio. Número de petició obtingut prèviament al realitzar la petició.
     
    Codis d'error de retorn:
    - 400: Nom de treball o client incorrectes
    - 401: Falta autenticació, autenticació incorrecta o no autoritzat pel treball demanat
    - 404: El client no existeix  
    - 406: El client correspon a un municipi no adherit
    - 500: Error en la gestió de la petició
      
    Amb el codi http 200 es retorna el número de petició i el seu estat actual. [Podeu veure un exemple d'estat pendent](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%202%20resposta%20pendent.json) i [un exemple d'estat finalitzat amb els fitxers generats](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%203%20resposta%20finalitzat%20amb%20fitxers.json). En cas de retornar fitxers, de cada fitxer s'informa:
    - Nom: és el nom que identifica el fitxer i que cal especificar en el següent endpoint per obtenir el seu contingut.
    - Descripcio: Descripció del contingut del fitxer.

        
1. **treballs/{nomTreball}/peticio/{idPeticio}/fitxers/{nomFitxer} GET** Obté un fitxer resultat d'una petició. El resultat s'envia com application/octet-stream  i "Content-Disposition: attachment; filename="{fitxer}". La codificació és ISO-8859-1. La petició ha d'haver estat feta des del mateix Ajuntament que està obtenint el fitxer. 
    
    Aquest endpoint té tres paràmetres al path:
   - nomTreball. (8 caràcters màxim). Exemple: xbentri
   - idPeticio. Número de petició obtingut prèviament al realitzar la petició. 
   - nomFitxer. Nom del fitxer generat pel treball que ja ha finalitzat. Els noms de fitxer s'obtenen amb l'anterior endpoint.
     
    Codis d'error de retorn:
    - 400: Nom de treball o client incorrectes
    - 401: Falta autenticació, autenticació incorrecta o no autoritzat pel treball demanat
    - 404: Fitxer inexistent
    - 406: El client correspon a un municipi no adherit
    - 500: Error en la gestió de la petició


1. **municipis** Endpoint sense paràmetres que permet obtenir la llista de municipis amb el codi de client associat. En les crides als webservices de la ORGT, un municipi s'identifica pel codi de client. Aquest endpoint permet obtenir-los. La llista té el format `[{"Nom": "Municipi 1", "CodiClient": "001"}, {"Nom": "Municipi 2", "CodiClient": "002"}, ...]`. Aquest endpoint juntament amb el d'obtenir descripció d'un treball no requereix autenticació bàsica. Sí requereix identificar-se amb un certificat d'òrgan prèviament compartit amb l'ORGT.


## Exemples de crides i respostes
A continuació es presenten diversos exemples de crides i respostes. Podeu trobar més informació a l'[especificació swagger](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/swagger%20PeticioTreballsREST.json).

| Endpoint | Mètode HTTP | Exemples |
|---|---|---|
| 1. Obté descripció dades | GET | [URL petició](https://wsproves.orgt.diba.cat/PeticioTreballsREST/treballs/xbentri) <br> [Resposta](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%200%20resposta%20amb%20descripcio.json)
| 2. Crea petició treball | POST | [URL petició](https://wsproves.orgt.diba.cat/PeticioTreballsREST/treballs/xbentri) <br> [Dades a enviar a l'endpoint com a body de la petició](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%201%20peticio%20treball%20POST.json) <br> [Resposta amb número de petició](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%201%20resposta%20amb%20num%20peticio.json)
| 3. Consulta estat (pendent) | GET | [URL petició](https://wsproves.orgt.diba.cat/PeticioTreballsREST/treballs/xbentri/peticio/2027605) <br> [Resposta estat pendent](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%202%20resposta%20pendent.json)
| 4. Consulta estat (finalitzat) | GET | [URL petició](https://wsproves.orgt.diba.cat/PeticioTreballsREST/treballs/xbentri/peticio/2027605) <br> [Resposta estat finalitzat i fitxers](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%203%20resposta%20finalitzat%20amb%20fitxers.json)
| 5. Consulta fitxer | GET | [URL petició](https://wsproves.orgt.diba.cat/PeticioTreballsREST/treballs/xbentri/peticio/2027642/fitxers/Beneficis%20fiscals%20i%20impacte%20en%20recaptaci%C3%B3_Beneficis_Fiscals_Resum_IAE_001_2024_4_204151.xls) <br> [Resposta amb contingut de fitxer](https://github.com/organisme-de-gestio-tributaria/PeticioTreballs/blob/main/Exemples/exemple%201%20-%20pas%204%20resposta%20contingut%20fitxer.xls)


# a tenir en compte (revisar i esborrar)
- oferir endpoint amb la llista de municipis en format json?
- Per evitar penalitzar els usuaris d'oficines, les peticions s'assignaran a una cua exclusiva per a aquest tipus de treballs.
- Per temes d'auditoria, es faran logs de les peticions dels treballs, tot indicant el nom que figura al certificat (SubjectName).
- Està previst que en un futur es puguin passar fitxers sencers com a paràmetre: els paràmetres de tipus "FitxerCarpetaXarxa permeten posar el contingut codificat en base 64 del fitxer que es vol processar. [↑](#footnote-ref-1)
