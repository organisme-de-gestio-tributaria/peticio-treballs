# PeticioTreballs

Web service de petició de treballs des d'Ajuntament

# Especificació

PeticioTreballs és un web service que permet demanar treballs en diferit i recuperar els fitxers que genera.

Es tracta d'un web service REST securitzat amb certificat, que rep i retorna les dades en format JSON. Hi ha dues URLs d’accés:

- Pre-producció: <https://wsproves.orgt.diba.cat/PeticioTreballs/PeticioTreballs.svc/rest>.
- Producció: <https://wsreal.orgt.diba.cat/PeticioTreballs/PeticioTreballs.svc/rest>.

El web service té les següents operacions:

- ObteDadesPeticio: obté els paràmetres d'un treball. L'estructura retornada (DadesPeticio) es pot fer servir, omplint els atributs ValorParametre, per demanar el treball amb el mètode DemanaTreball. Si es produeix un error, l'estructura Retorn ve informada amb el CodiRetorn i DescripcioError. Només es poden obtenir les dades de treballs pensats per a Ajuntaments, és a dir, que tenen almenys un paràmetre amb el codi de l'Ajuntament.
- DemanaTreball: a partir de l'estructura DadesPeticio amb els ValorParametre emplenats, crea una petició del treball perquè es faci en diferit. L'estructura DadesPeticio es pot omplir des de zero, o obtenir-la amb el mètode ObteDadesPeticio i emplenant els ValorParametre.<sup>[\[1\]](#footnote-1)</sup> Retorna un ResultatPeticio on, si tot va bé, l'atribut NumeroPeticio està informat amb el número assignat a la petició. Si ha hagut un error en demanar la petició, retorna informada l'estructura Retorn. La petició s’entén feta per l’Ajuntament associat al certificat amb el qual s’accedeix. Des de pre-producció, com que el certificat pot ser el d’una empresa desenvolupadora, cal informar l'atribut CodiClient de DadesPeticio amb el codi ORGT de l'Ajuntament pel qual es demana el treball. Només es poden demanar treballs pensats per a Ajuntaments, és a dir, que tenen almenys un paràmetre amb el codi de l'Ajuntament. Cal emplenar tots els paràmetres, cap d'ells no pot ser null, si és opcional cal posar "".
- ObteResultatExecucio: a partir del número de la petició, retorna el resultat de la seva execució. Es comprova que només es poden consultar peticions fetes pel mateix Ajuntament que consulta el resultat, d’acord amb el certificat digital aportat en fer la consulta. Des de pre-producció, com que el certificat pot ser el d’una empresa desenvolupadora, cal informar el paràmetre addicional codiClient amb el codi ORGT de l'Ajuntament pel qual es demana el treball.

El resultat de l’execució es retorna a l’estructura ResultatExecucio amb la següent informació:

- - Estat, pot ser:
        - 0 : Pendent, el treball encara no ha entrat a executar-se o ha entrat però encara no ha acabat. Cal tornar a cridar ObteResultatExecucio més tard.
        - 1: Finalitzat, el treball ha finalitzat correctament.
        - 2: Erroni, el treball ha acabat amb error o ha estat anul·lat. Cal posar-se en contacte amb l'ORGT aportant el número de petició per saber què ha passat.
    - Fitxers: només ve informat quan l'estat és Finalitzat. És una estructura que al seu torn conté una llista d'elements de tipus InformacioFitxer. Cada un d'ells correspon a un fitxer de sortida generat pel treball. De cada fitxer es retorna:
      - Nom: és el nom que identifica el fitxer, i que s’especificarà per obtenir el seu contingut amb el mètode ObteFitxer.
      - Descripcio: Descripció del contingut del fitxer.
- ObteFitxer: a partir del número de la petició i el nom del fitxer, obté el fitxer en forma de "byte stream", que es baixarà al client amb el nom indicat. La petició ha d'haver estat feta des del mateix Ajuntament que està obtenint el fitxer. Des de pre-producció, com que el certificat pot ser el d’una empresa desenvolupadora, cal passar un paràmetre addicional codiClient amb el codi ORGT de l'Ajuntament que està obtenint el fitxer. Els errors possibles són:
  - NotFound (HttpStatusCode=404). El número de petició no existeix o no pertany a l'Ajuntament.
  - ServiceUnavailable (HttpStatusCode=503). La petició està pendent de finalitzar. Cal tornar a cridar el mètode ObteFitxer més tard.
  - NotImplemented (HttpStatusCode=501). La execució de la petició ha estat errònia o anul·lada.
  - InternalServerError (HttpStatusCode=500). S'ha produït un error no controlat, cal contactar amb l'ORGT.

# Seguretat

## Pre-producció

A pre-producció s'accedeix al web service amb certificat que ha de facilitar l'empresa desenvolupadora a l'ORGT perquè l'afegeixi com a persona de confiança.

## Producció

A producció s'accedeix al web service mitjançant certificat d'Òrgan a nom de l'Ajuntament. Concretament, el certificat ha de tenir al camp VinculatedCompanyCIF el CIF de l'Ajuntament. Aquest certificat l'ha de facilitar a l'ORGT perquè l'afegeixi com a persona de confiança. A més, cal haver omplert el formulari d'adhesió al web service per part de l'Ajuntament.

El treball ha d'estar autoritzat a l'usuari USU-INET des de l'opció I10, pestanya "Opcions assignades", sinó s'obtindrà un error "USU01082". Queda pendent per decidir com saber si el treball és d'Ajuntament i si està autoritzat a aquell Ajuntament en concret. Actualment disposem de la I97, on s'autoritzen els treballs a determinats perfils. Hi ha diversos perfils d'Ajuntament, tots comencen amb "AJ".

Es podria plantejar que en la petició s'especifiqués l'usuari d'Ajuntament que es fa servir per demanar el treball per la U90 del WTP. El problema és que no podem evitar que posin el nom d'un altre usuari del mateix Ajuntament, a no ser que accedeixi amb un certificat que l'identifiqui com a usuari d'aquell Ajuntament.

Per evitar penalitzar els usuaris d'oficines, les peticions s'assignaran a una cua exclusiva per a aquest tipus de treballs.

Per temes d'auditoria, es faran logs de les peticions dels treballs, tot indicant el nom que figura al certificat (SubjectName).

# Exemple d'ús

- Cal que ens instal·lem el Mozilla Firefox, i afegir l'extensió RESTClient per poder fer peticions al web service.
- Obrim el navegador Firefox i executem l'extensió RESTClient. Al desplegable "Method" posem "GET", i al camp "URL" posem <https://wsproves.orgt.diba.cat/PeticioTreballs/PeticioTreballs.svc/rest/obtedadespeticio?nomTreball=xbentri> i premem el botó "SEND".

Ens demanarà el certificat amb el qual volem accedir, hem de posar el certificat de l'empresa desenvolupadora que prèviament s'ha comunicat a l'ORGT perquè l'afegeixi a la seva llista de confiança. Pot ser que la finestra que demana el certificat quedi en segon pla i sembli que la petició no acaba mai, hem de seleccionar la finestra del certificat a la barra de tasques per activar-la i entrar-lo.

A la secció Response ens surt la informació sobre els paràmetres del treball, en format JSON:

{"CodiClient":null,"NomTreball":"xbentri","Retorn":null,"ValorsParametres":\[{"DefinicioParametre":{"Llargaria":3,"Nom":"cdclie","Tipus":1},"ValorParametre":null},{"DefinicioParametre":{"Llargaria":4,"Nom":"cdexer","Tipus":1},"ValorParametre":null},{"DefinicioParametre":{"Llargaria":1,"Nom":"trimes","Tipus":0},"ValorParametre":null}\]}

- Canviem el desplegable "Method" a "POST"
- Agafem la informació de la secció Response i l'enganxem al camp "Body", modifiquem aquesta informació, posant on diu "Null" les dades concretes de la petició (en groc), o copiem aquest exemple tal qual:

{"CodiClient":"001","NomTreball":"xbentri","Retorn":null,"ValorsParametres":\[{"DefinicioParametre":{"Llargaria":3,"Nom":"cdclie","Tipus":1},"ValorParametre":"001"},{"DefinicioParametre":{"Llargaria":4,"Nom":"cdexer","Tipus":1},"ValorParametre":"2023"},{"DefinicioParametre":{"Llargaria":1,"Nom":"trimes","Tipus":0},"ValorParametre":"1"}\]}

- Canviem la URL a <https://wsproves.orgt.diba.cat/PeticioTreballs/PeticioTreballs.svc/rest/DemanaTreball>
- A dalt de la pàgina despleguem "Headers" i seleccionem "Custom Header". Al camp “Name” posem "Content-Type", al camp "Attribute Value" posem "text/json", i premem el botó "Okay". Veurem que a la secció “Headers” ens ha afegit aquesta informació.
- Premem el botó "SEND", i ens retornarà a "Response" el número de petició i el Retorn a Null, per indicar que tot ha anat bé:

{"NumeroPeticio":5256650,"Retorn":null}

- S'ha afegit la petició a la U90 del treball xbentri demanat. Es pot comprovar des del WTP, entrant a la U90 i mirant les peticions de l'usuari USU-INET o de l'oficina 002. Com que ara per ara cada usuari només pot veure les seves peticions o les de la seva oficina, potser caldrà designar una persona o Servei que pugui consultar aquestes peticions. Veurem que la petició està pendent, ja que a proves no s’executen els treballs automàticament.
- Podem comprovar l'estat de la petició posant "GET" a "Method" i escrivint a URL <https://wsproves.orgt.diba.cat/PeticioTreballs/PeticioTreballs.svc/rest/ObteResultatExecucio?numeroPeticio=5256650&codiClient=001> (substituïu “5256650” pel número de petició que ens hagi donat). En prémer el botó “SEND” ens retornarà:

{"Estat":0,"Fitxers":null,"Retorn":null}

L'estat "0" vol dir que el treball està pendent d'execució. Per tant, el camp "Fitxers" encara no conté res. El "Retorn" és nul per indicar que ha anat bé.

- A producció el treball s’executarà quan li toqui. A proves, cal avisar a Informàtica perquè s’encarregui d’executar el treball i fer que els fitxers de resultat vagin a la U91.
- Un cop s'executa el treball i acaba correctament, en tornar a consultar rebrem un resultat diferent:

{"Estat":1,"Fitxers":{"Fitxer":\[{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_IBI_BC_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_IBI_RU_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_IBI_UR_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_ICIO_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_IIVTNU_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_IVTM_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Detall_TOTS_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_IBI_BC_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_IBI_RU_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_IBI_UR_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_ICIO_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_IIVTNU_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_IVTM_001_2023_1.xls"},{"Descripcio":"","Nom":"Beneficis_Fiscals_Resum_TOTS_001_2023_1.xls"}\]},"Retorn":null}

- L'estat és "1" (Finalitzat), i ens retorna la llista de fitxers que tenim disponibles. De cada fitxer tenim la seva descripció i el nom amb el qual es descarregarà. La descripció està en blanc a l'entorn de pre-producció, però a producció vindrà informada.
- Pot passar que retorni estat “1” (Finalitzat), però la llista de fitxers estigui buida. Això passa perquè hi ha un temps de retard des que finalitza el treball fins que els fitxers de resultat estan disponibles. Si aquesta situació persisteix durant més d’una hora, cal avisar a Informàtica.
- Per obtenir un dels fitxers, per exemple el primer, posem "GET" a "Method" i a URL posem <https://wsproves.orgt.diba.cat/PeticioTreballs/PeticioTreballs.svc/rest/ObteFitxer?numeroPeticio=5256650&nom=Beneficis_Fiscals_Detall_IBI_BC_001_2023_1.xls&codiClient=001> (substituïu “5256650” pel número de petició que ens hagi donat).
- El Mozilla Firefox ens detecta que la resposta és un fitxer binari, i ens demana si volem canviar el tipus de resposta, premem el botó que diu "Set response type to Blob, and send again". Veurem que a "Response" ens ha posat el contingut del fitxer en format base64, el podem descarregar prement el botonet de la dreta ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABYAAAAVCAIAAADNQonCAAAAAXNSR0IArs4c6QAAAVpJREFUOE9j+Pr165kzZ169evWfFABUD9QF1AvUxHD79m1S9UPsAuoC6gUymD5+/CgqKspAOgDqAuoF6mMiXS+6DrxG/L29e+HChcuPPMVrD14jXtw4d/369Ys3n5FvBHGexOWKdwdnNTfPPQUKLob7W5qbm2cf+YTDRFxGCNl4GHD++P0PpO3Pz78Clu6WfCQawcAs51uYYS0CtIJD2iUpy0WOGZe3WDAlrq/tWHPtN0ic3zgmNfwDC++T9W3NYC+xaoVUBGuiacHikS8fPgBTLgg8OzR73ubNs+YdfcPBxfyTmYvj24cvmFZiMYKZGSH47/e3b8AA+fPp03eG758+sXDxYPEOMLeQkr9Q1EL00jqBU5a0iNMNVsXEz8//+vVrEnTAlAJ1AfWCjJCSknr06BGppgDVA3UB9QKNYAQG6bdv3549ewYpP4gEQPuB+rm4uKBGEKkNlzIA3zk44s4KJJIAAAAASUVORK5CYII=). Ens surt un diàleg per desar el fitxer, li diem "beneficis fiscals.xls", i el podrem obrir amb l'Excel.
- Això que hem fet des de Mozilla Firefox es pot fer des d’altres clients com SoapUI o Postman, o des d'un programa desenvolupat per l'Ajuntament, de cara a automatitzar la petició de treballs i el processament dels fitxers de sortida.

1. Està previst que en un futur es puguin passar fitxers sencers com a paràmetre: els paràmetres de tipus "FitxerCarpetaXarxa permeten posar el contingut codificat en base 64 del fitxer que es vol processar. [↑](#footnote-ref-1)
