---
path: '/luku-10/2-java-virtuaalikone'
title: 'Java virtuaalikone (JVM)'
hidden: false
---

<div>
<lead>Tässä aliluvussa (10.2) esittelemme pääpiirteet Java-ohjelmien suorittamisesta Java virtuaalikoneen (JVM) avulla. Käymme läpi JVM:n perusrakenteen ja sen konekielen (Javan tavukoodi, Bytekode) käskytyypit. Tarkoitus ei ole antaa tyhjentävää esitystä JVM:stä, vaan esittää yleiskuva. Seuraavassa aliluvussa (10.3) käymme läpi vähän tarkemmin erilaiset tavat toteuttaa JVM ja kuinka Java-ohjelmien suoritetaan niissä. </lead>
</div>

[Java](https://fi.wikipedia.org/wiki/Java) on korkean tason luokkaperustainen olio-ohjelmointikieli. Java-kieliset ohjelmat voisi kääntää ja linkittää ajomoduuleiksi samalla tavalla kuin edellisessä luvussa 9 esitettiin. Näin ei kuitenkaan yleensä tehdä. Korkean tason kielten kääntäjän toteutuksessa on usein osana välikieli, joka toimii siltana kääntäjän _front endin_ ja _back endin_ välillä. Javassa tuo välikieli on nostettu näkyville erityisasemaan. Käännösmoduulit välitetään eteenpäin myös Javan välikielisessä muodossa eikä vain Java-kielisinä käännösmoduuleina tai niiden objektimoduuleina. Dynaamisessa linkityksessä Javan välikieliesitys on nostettu liki samanlaiseen asemaan kuin yleensä ovat dynaamisesti linkitettävät objektimoduulit.

Javan välikieli on nimeltään [Bytecode](https://en.wikipedia.org/wiki/Java_bytecode) tai [Java Bytecode](https://en.wikipedia.org/wiki/Java_bytecode). Siitä käytetään jatkossa nimiä tavukoodi tai Javan tavukoodi.

Javan tavukoodi on hypoteettisen Java virtuaalikoneen ([JVM](https://fi.wikipedia.org/wiki/Java), Java Virtual Machine) konekieltä, ihan samalla tavalla kuin ttk-91 konekieli on hypoteettisen ttk-91 suorittimen konekieltä. Toisin kuin ttk-91 ja sen konekieli, JVM ja sen tavukoodi ovat täydellisesti määriteltyjä.

## JVM ja pinokone
Java virtuaalikone on geneerinen suoritin, joka toimii rajapintana kaikille Java-ohjelmien erilaisille suoritusmuodoille. Kun useimmat todelliset suorittimet sisältävät rekistereitä laskutoimituksia varten, niin JVM:ssä kaikki laskentaa perustuu pinoon. Laskutoimitusten argumentit ovat aina pinon pinnalla ja aritmeettis-loogisssa operaatiossa ne korvautuvat laskutoimituksen tuloksella. Tyypillistä tällaisessa [pinokoneessa](https://en.wikipedia.org/wiki/Stack_machine) on, että aritmeettis-loogisissa operaatioissa ei tarvitse nimetä yhtään operandia, koska ne löytyvät oletusarvoisista paikoista pinon pinnalta. Operaation suorituksessa operandit otetaan pois pinosta ja tulos talletetaan pinoon. 

Rekisteripohjaiset suorittimet toimivat hyvin eri tavalla. Aritmeettis-loogisissa operaatioissa on yleensä nimetty kolme rekisteriä, joista yksi on tulosrekisteri. Tällä tavoin kummankaan operandin arvoa ei tarvitse tuhota. Ttk-91 suorittimessa aritmeettis-loogisissa operaatioissa oli vain kaksi nimettyä operandia ja tulos talletetaan aina ensimmäisen operandin (rekisterin) päälle. 

Esimerkki. Katsotaan lausekkeen _z = (x+y)(x+5)_ toteutusta suorittimilla, joissa aritmeettis-loogisissa operaatioissa on 0, 2 tai 3 operandia. Kolmen operandin esimerkki on [load-store](https://en.wikipedia.org/wiki/Load/store_architecture) arkkitehtuuri, jossa muistiviitekäskyt ovat erillään aritmeettis-loogisista operaatioista.

```
Pinokone        ttk-91          load-store

push x          load r1, x      load r1, x
push y          add  r1, y      load r2, y
add             load r2, x      load r3, =5
push x          add r2, =5      add r8, r1, r2
push 5          mul r1, r2      add r9, r1, r3
add             store r1, y     mul r10, r8, r9
mul                             store r10, z
pop z
```

JVM ei määrittele, onko suorituksessa rinnakkaisuutta vai ei. On täysin mahdollista, että yhdestä prosessista on samanaikaisesti suorituksessa useampi säie. Esimerkiksi kaksi peräkkäistä toisistaan riippumatonta metodin kutsua voisivat olla yhtä aikaa suorituksessa (moniytimisellä) laitteistolla, jos vain käytössä oleva JVM:n toteutus tämän sallii.

Javan virtuaalikone voidaan toteuttaa usealla eri tavalla, mikä tekee tästä mielenkiintoisen ohjelmien suoritusmallin. Suoritettava Java-ohjelma siis käännetään aina ensin tavukoodiksi, jonka jälkeen se suoritetaan _jollain tavalla toteutetulla_ JVM:llä. Periaatteessa on ainakin neljä tapaa toteuttaa JVM ja Java-ohjelmia voi sitten suorittaa noilla kaikilla tavoilla. Suoritustavat esitellään tässä lyhyesti ja seuraavassa aliluvussa tarkemmin. 

<!-- Kuva: ch-10-2-java-ohjelmien-suoritus -->

![Java-ohjelmien suoritustavat. Keskellä on vaakasuora viiva kuvaamassa Java virtuaalikonetta. Viivan yläpuolella on ylhäällä Java-ohjelma, jossa koodinpätkä  k=i+j. Sen alla on sama ohjelma tavukoodiksi käännettynä, sisältäen tavukoodin käskyt iload i, iload j, iadd ja istore k. Viivan alapuolella on kolme eri laatikkoa kuvaamassa eri suoritustapoja. Vasemmalla on tulkitseminen, jossa Intel Pentium II järjestelmässä suorituksessa oleva Java-tulkki lukee datana tavukoodisia käskyjä viivan yläpuolelta. Keskellä on JVM suoritin, jossa natiivisuorittimena on Java-suoritin ja suorituksessa oleva prosessi on tavukoodisessa esitysmuodossa, joka on luettu koodina viivan yläpuolelta. Oikealla on käännös ja JIT-käännös, joissa viivan yläpuolelta luetaan tavukoodi taas datana ja siiät muokataan kääntämällä ja linkittämällä Intel Pentium II:ssa suoritettava latausmoduuli.](./ch-10-2-java-ohjelmien-suoritus.svg)
<div>
<illustrations motive="ch-10-2-java-ohjelmien-suoritus" frombottom="0" totalheight="40%"></illustrations>
</div>

Tulkitseminen tarkoittaa sitä, että käytössä on ohjelma _JVM tulkki_ (tulkki, Java tulkki), jossa on kaikki JVM:n rakenteet toteutettu tulkin (ohjelman) tietorakenteina. Java-tulkki lukee datana tavukoodin käskyjä yksi kerrallaan ja sitten toteuttaa käskyjen aiheuttamat muutokset JVM:n tietorakenteissa. Esimerkiksi _iadd_-käsky poistaa pinon pinnalta kaksi kokonaislukua ja tallettaa pinon pinnalle niiden summan. Java tulkki on normaali korkean tason kielellä toteutettu ohjelma, joka on käännetty ja linkitetty käytössä olevaan järjestelmään. Järjestelmä voi käyttää mitä tahansa suoritinta, esimerkiksi Intelin Pentium II:sta. Suoritustapa on hyvin samanlainen kuin miten Titokone lukee ttk-91 konekäskyjä ja emuloi niiden aiheuttamia muutoksia simuloiduissa ttk-91 rekistereissa ja muistissa. 

JVM:n voi toteuttaa myös suoraan laitteistolla _JVM suorittimella_ (Java suorittimella), joka suorittaa tavukoodia konekäskyinä. Tässä tapauksessa latausmoduuli siis sisältää alkuperäisen Java-ohjelman tavukoodisen esityksen ja prosessin suoritusaikana tavukoodi luetaan koodina eikä datana. Tämä vastaa tilannetta, jossa (täydellisesti määritelty) ttk-91 toteutettaisiin oikeasti todellisena suorittimena.

Kolmas vaihtoehto on _kääntää ja linkittää_ ohjelman _tavukoodinen_ esitysmuoto "normaalitapaan" järjestelmän suorittimen konekielelle. Tämä vaihtoehto muistuttaa eniten edellisessä luvussa kuvattua menettelyä, mutta sitä ei yleensä käytetä. Menetelmä eroaa normaalista kääntämisestä siinä, että käännösmoduulit ovat tavukoodia eivätkä korkean tason kielen koodia.

Neljäs vaihtoehto perustuu _Just-In-Time käännökseen_, jossa kukin viitattu Javan _moduuli_ käännetään ja linkitetään paikalleen vasta tarvittaessa. Tämä on vähän samanlaista kuin dynaamisessa linkityksessä, mutta objektimoduulien asemesta uuden moduulin esitysmuotona on tavukoodi. Lähestymistapa on joustavaa, koska tavukoodi on geneeristä, mutta objektimoduulien koodi on aina jollain tietyllä konekielellä. Dynaamisen linkittäjän lisäksi tarvitaan nyt myös tavukoodin kääntäjä natiiviympäristön suorittimelle.

### JVM:n pino

Java virtuaalikoneessa on _pino_, jossa on ohjelman tietorakenteet, välitulokset ja aktivaatiotietueita vastaavat _kehykset_. Pinossa ovat siis mm. kaikki ohjelman käyttämät muuttujat ja laskennan välitulokset. Tämä on hyvin erilainen lähestymistapa kuin todellisissa suorittimissa yleinen tapa säilyttää usein tarvittavien muuttujien arvoja ja laskennan välituloksia nopeissa rekistereissä. Jos JVM toteutetaan rekisterikoneessa, niin toteutusta voi hidastaa se, että kaikki data on (ainakin teoriassa) muistissa olevassa JVM:n pinossa.

Tavukoodissa on pinolle normaalien "push/pop"-käskyjen lisäksi JVM:ssä omat "push/pop"-käskynsä myös _kehyksille_ (JVM:n "aktivaatiotietueille") , jolloin niitä ei tarvitse rakentaa ja purkaa sana kerrallaan. Kehysten käyttöä tarkennetaan ihan kohta.

JVM:n pinon ei tarvitse olla yhtenäisellä muistialueella, vaan se allokoidaan _keosta_ (kuten kaikki muutkin JVM:n tietorakenteet). Pinon koko voi olla rajallinen tai dynaamisesti laajennettavissa, jolloin pinon muistitilan loppuessa sille voidaan varata lisää muistitilaa keosta. Sama pätee kaikkiin muihinkin JVM:n varaamiin tietorakenteisiin. 

Pinoon osoittaa kaksi rekisteriä. SP (stack pointer) osoittaa pinon päällimmäiseen alkioon ja LV (local variables frame) nykykehyksen alkuun ja samalla sen ensimmäiseen paikalliseen muuttujaan. Kumpaankaan näistä rekistereistä (kuten ei muihinkaan JVM:n rekistereistä) ei mitenkään nimetä JVM:n konekäskyissä, vaan kaikki rekisteriviittaukset ovat implisiittisiä. Esimerkiksi add-käsky viittaa dataan aina SP:n kautta, vaikka SP:tä ei mitenkään nimetä konekäskyssä. SP:n käyttö tässä tapauksessa päätellään operaatiokoodista (add).

Allaolevan esimerkin lähtötilanteessa ollaan suorittamassa jotain Javan _metodia_ (aliohjelmaa), jossa on kolme kokonaislukuarvoista paikallista muuttujaa. Paikallisen muuttujan _i_ arvo on 111, _j_:n arvo on 222 ja _k_:n arvo on 700. Ne ovat pinossa tämän kutsukerran kehyksessä, jonka alkuun osoittaa LV. Rekisteri SP osoittaa pinon huipulle. Tavukoodissa seuraavana olevilla käskyillä lasketaan Javan lause "k=i+j;". Suoritusaikana tavukoodi on (tietenkin) vain numeerisia tavuja, mutta esimerkin vuoksi se esitetään tässä (myös) tekstuaalisessa muodossa. Koodinpätkän seitsemän tavun heksadesimaaliesityksen ja tekstuaalisen tavukoodin välillä on suoraviivainen vastaavuus.

```
Tavukoodi tekstuaalisena    tavuina        
          iload i           0x15 0x02      i:n osoite on LV+2
          iload j           0x15 0x03      j:n osoite on LV+3
          iadd              0x60
          istore k          0x36 0x04      k:n osoite on LV+4
```

<!-- Kuva: ch-10-2-yhteenlasku-pinossa -->

![Yhteenlasku pinossa. Viisi eri tilannetta pinon sisällöksi. Aluksi vasemmalla pinossa on päällä arvot 700, 222 ja 111 muuttujien k, j ja i arvoina. Näkyvään pinon osaan osoittaa LV sen pohjalle ja SP sen pinnalle. Muuttujan i osoite on LV+2, j:n osoite LV+3 ja k:n osoite LV+4. Käskyn "iload i" jälkeen pinon pinnalle on replikoitu i:n arvo 111. Käskyn "iload j" jälkeen pinon pinnalle on replikoitu myös j:n arvo 222. Käskyn iadd jälkeen em. arvot 111 ja 222 on poistettu pinoista ja sinne on laitettu niiden summa 333. Käskyn "istore k" jälkeen muuttujan k (osoitteessa LV+4) arvo on muuttunut arvoon 33 ja pinon päällä on nyt arvot 333, 222 ja 111 muuttujien k, j ja i arvoina.](./ch-10-2-yhteenlasku-pinossa.svg)
<div>
<illustrations motive="ch-10-2-yhteenlasku-pinossa" frombottom="0" totalheight="40%"></illustrations>
</div>

Pinokoneiden heikkoutena on, että jokainen aritmeettinen operaatio tuhoaa molemmat operandinsa. Argumenttien arvot täytyy aina kopioida pinon huipulle ennen aritmeettis-loogisia operaatioita ja niiden tulokset täytyy vastaavasti ottaa talteen operaation jälkeen. Lisäksi tarvitaan erilaisia replikointikäskyjä, joilla monistetaan pinon pinnalla olevia arvoja. 

Paikallisiin muuttujiin ja muihin tietorakenteisiin viitataan käyttäen niiden suhteellisia osoitteita LV:n suhteen. Tilanne on täysin vastaava kuin ttk-91:ssä aliohjelmien paikallisiin muuttujiin viittaaminen aktivaatiotietueen osoitteen (FP) suhteen.

Ensimmäinen käsky _iload i_ kopioi paikallisen muuttujan _i_ arvon 111 pinon huipulle ja toinen käsky _iload j_ kopioi vastaavasti paikallisen muuttujan _j_ arvon 222 pinon huipulle. Yhteenlaskukäsky _iadd_ ottaa argumentit pois pinosta, laskee niiden summan 333 ja tallettaa sen pinon huipulle. Lopulta pinoon talletuskäsky _istore k_ ottaa tuloksen pois pinosta ja tallettaa sen _k_:n arvoksi. 

Koska _iload_-käskyn parametrin arvo on useimmiten 0, 1, 2 tai 3, niin niitä varten [tavukoodin käskyissä](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings) on myös omat yhden tavun konekäskynsä *iload_0*,  *iload_1*, *iload_2* ja  *iload_3*. Käskyllä _istore_ on vastaavat optiot, mutta siinäkin vain parametriarvoihin 0-3. Näitä käskyjä käyttäen em. koodinpätkän saisi tavukoodina mahtumaan vain 5 tavuun.


```
Tavukoodi tekstuaalisena    tavuina        
          iload i           0x1c           i:n osoite on LV+2
          iload j           0x1d           j:n osoite on LV+3
          iadd              0x60
          istore k          0x36 0x04      k:n osoite on LV+4
```

Tämä näyttää vähän tehottomalta, kun operandeja pitää kopioida pinon pinnalle laskutoimituksia varten, molemmat operandit tuhoutuvat aritmetiikkaoperaatioissa ja kaikki dataviitteet kohdistuvat muistissa olevaan pinoon. Rekisteripohjaisissa suorittimissa JVM:n emulointi onkin vaikeaa, kun suoritinarkkitehtuurien peruslähtökohdat ovat niin erilaisia pinokoneella ja rekisterikoneella. 

### JVM:n keko, metodialue ja vakioallas
JVM:ssä kaikki muistinhallinta on keskitetty JVM:n omaan kekoon. Aina kun ohjelma tarvitsee lisää muistitilaa (esim. uudelle Javan _oliolle_ eli luokan ilmentymälle Java-operaatiolla _new_), niin JVM:n toteutusympäristö varaa sen tästä keosta. Vastaavasti, jos JVM itse tarvitsee lisää muistitilaa (esim. pinoa varten), niin myös se varataan täältä. 

JVM:ssä ei ole mitään varatun tilan vapauttamiskäskyä, vaan tila vapautuu uusiokäyttöön _automaattisen roskienkeruun_ kautta. Se tarkoittaa, että aika ajoin (a) laskenta pysähtyy, (b) roskienkeruu käynnistyy ja vapauttaa aikaisemmin varatun mutta ei enää käytössä olevan muistitilan ja (c) lopulta laskenta voi jatkua. Roskienkeruu tapahtuu esimerkiksi siten, että ensin merkitään kaikki muistialueet vapaiksi. Sitten käydään läpi kaikki ohjelman ja JVM:n sillä hetkellä käytössä olevat muistialueet merkiten ne samalla varatuiksi. Lopuksi otetaan uusiokäyttöön jäljelle jääneet vapaaksi merkityt alueet. Ymmärrettävästi tämä voi viedä aikaa. Roskienkeruu on ongelmallista, koska se pysäyttää laskennan satunnaisiin aikoihin ja voi kestää pitkäänkin. 

Pinossa on jokaiselle metodin kutsulle sitä vastaava kehys, jonka päällä voi vielä olla siinä metodissa käytössä olevat välitulokset. Pinoa käytetään siis sekä metodin kutsurakenteen toteutukseen että laskennan välitulosten tallentamiseen. 

<!-- Kuva: ch-10-2-muistialueet -->

![JVM:n muistialueet ja rekisterit. Kolme aluetta: metodialue, pino ja vakioallas. Metodialueen keskellä on jossain metodi B:n koodi ja PC osoittaasinne jonnekin. Pinossa on kehykset pääohjelmalle main, metodille A ja metodille B. Kunkin kehyksen päällä on siihen liityvöät välitulokset. LV osoittaa viimeksi kutsutun metodin B alkuun ja SP osoittaa pinon pinnalle. Oikeall on vakioallas, jonka alkuun osoittaa CPP.](./ch-10-2-muistialueet.svg)
<div>
<illustrations motive="ch-10-2-muistialueet" frombottom="0" totalheight="40%"></illustrations>
</div>

JVM:ssä on SP:n ja LV:n lisäksi vain kaksi muuta rekisteriä. Rekisteri PC on tavanomainen paikanlaskuri ja osoittaa seuraavaksi suoritettavaan (tavukoodiseen) käskyyn nykyisessä metodissa. Metodien koodit on talletettu omalle metodialueelleen (JVM Method Area), joka on yhteinen kaikille yhden prosessin säikeille. 

Rekisteri CPP (Constant Pool Pointer) osoittaa _vakioaltaaseen_ (constant pool), jossa on kaikki ohjelman käyttämät vakiot ja muut symboliset viitteet. Vakioaltaan tietoihin viitataan käyttäen niiden suhteellista osoitetta CPP:n suhteen. Jokaiselle Javan luokalle (class) ja liittymälle (interface) on oma vakioaltaansa, joka on suoritusaikainen esitystapa tiedoston _class constant pool_ taulukolle. Tämä vastaa vähän symbolitaulua (tai sen osaa). Vakioaltaassa on useita eri tyyppisiä vakioita, kuten esimerkiksi tavalliset literaalit ja suoritusaikana ratkottavat attribuutit dynaamista linkitystä (JIT) varten. Vakioaltaat varataan tietenkin keosta.

Jos suoritettavia säikeitä on useita, niin kaikilla on omat rekisterinsä, pinonsa ja vakioaltaansa. Niillä on kuitenkin yhteinen metodialue ja keko.

### Metodin kutsu
Metodin kutsukäsky on _invokevirtual_ ja se luo uuden kehyksen pinoon. Ennen käskyä _invokevirtual_ kutsuja laittaa pinoon viitteen kutsuttavan _olion_ luokkaan ja parametrien arvot. Käskyssä _invokevirtual_ annetaan parametrina viite kutsuttavaan metodiin ja käskyn _invokevirtual_ suorituksen jälkeen uusi kehys on valmis. 

Esimerkki. Tarkastellaan metodia _A_, jossa on paikalliset muuttujat _x_ ja _y_. Metodissa _A_ on seuraavana Java-lausetta _y=Obj.B(x, 5)_ vastaava tavukoodinen kutsu. Kutsun voisi toteuttaa vaikkapa seuraavalla tavalla.

```
Metodi A

...
getstatic #35       0xb2 0x00 0x23   viite olion Obj luokkaan on CPP+35:ssä
iload  x            0x1b             parametri 1, muuttujan x arvo, osoite LV+1
bipush 5            0x10 0x05        parametri 2, vakio 5
invokevirtual #37   0xb6 0x00 0x25   viite metodiin B on CPP+37:ssä
```

<!-- Kuva: ch-10-2-metodin-kutsu -->

![Vasemmalla on pinon alkutilanne ja oikealla pinon tila kutsun "invokevirtual B" jälkeen. Alkutilanteessa on metodin A kehys, jonka alkuun osoittaa LV. Metodin A kehyksesä on kentät "link pointer", "parametri 1", paikalliset muuttujat ml. x ja y, A:n kutsujan PC ja LV. Sen päällä on on kutsuttavan metodin B viite ja parametrit 1 ja 2.  Oikealla puolella metodin A kehyksen päälle on rakennettu metodin B kehys siten, että kutsuttavan metodin B viitteen paikalle on nyt laitettu linkki paluuosoitteeseen ja LV tähän linkkikenttään, joka on samalla metodin B kehyksen ensimmäinen sana. Metodin B kehyksen paikallisissa muuttujissa on myös muuttuja g.](./ch-10-2-metodin-kutsu.svg)
<div>
<illustrations motive="ch-10-2-metodin-kutsu" frombottom="0" totalheight="40%"></illustrations>
</div>

Esimerkissä käskyn _invokevirtual_ jälkeen kutsutun metodin _B_ kehys on valmis. Paluusoite löytyy epäsuorasti osoitteesta LV, parametrit ovat osoitteissa LV+1 ja LV+2, minkä jälkeen pinossa on paikalliset muuttujat. Pinon pinnalla on aikaisemman metodin _A_ kehyksen osoite, mitä tarvitaan metodista _B_ paluun yhteydessä.

Oletetaan nyt, että metodi _B_ palauttaa arvonaan yhden kokonaisluvun, joka on ennen metodista paluukäskyä _ireturn_ talletettu pinon pinnalle. 

```
metodi Obj.B

...
iload  g          0x15  0x08  aseta paluuarvo paikallisesta muuttujasta g, osoite LV+8
ireturn           0xac        palauta paluuarvo ja kontrolli kutsuvaan rutiiniin
```

<!-- Kuva: ch-10-2-metodista-paluu -->

![Vasemmalla on pinon alkutilanne ja oikealla pinon tila käskyn "ireturn" jälkeen. Vasemmalla tila on täsmälleen sama kuin edellisen kuvan oikealla puolella. Oikealla puolella tila muuten sama, mutta kutsutun metodin parametrit ovat poissa ja metodin B viitteen asemesta siinä kohtaa on nyt metodin B paluuarvo.](./ch-10-2-metodista-paluu.svg)
<div>
<illustrations motive="ch-10-2-metodista-paluu" frombottom="0" totalheight="40%"></illustrations>
</div>

Käskyn _ireturn_ suorituksessa metodin _B_ kehyksen tiedoilla palautetaan rekistereiden PC, LV ja SP arvot ennalleen ja kopioidaan paluuarvo pinon huipulle. Metodin _A_ suoritus voi nyt jatkua ja ensimmäisenä se tietenkin ottaa paluuarvon talteen.  

Metodin _B_ käyttö on nyt kokonaisuudessaan seuraavanlainen.

```
metodi A

...
getstatic #35       0xb2 0x00 0x23   viite olion Obj luokkaan on CPP+35:ssä
iload  x            0x1b             parametri 1, muuttujan x arvo, osoite LV+1
bipush 5            0x10 0x05        parametri 2, vakio 5
invokevirtual #37   0xb6 0x00 0x25   viite metodiin B on CPP+37:ssä
istore y            0x36 0x04        paluuarvo pinosta muuttujaan y, osoite LV+4
...
```
## Tavukoodi
Tarkoituksena ei ole tässä käydä kaikkia [tavukoodin käskyjä](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings) läpi, vaan antaa yleiskuva niistä. Alla käydään kursorisesti läpi tavukoodin tietotyypit, tiedonosoitusmoodit ja erilaiset käskytyypit.

### Tietotyypit
Käytössä on 1-, 2-, 4- ja 8-tavuiset kokonaisluvut. Datatyyppien nimet ovat vastaavasti _byte_, _short_, _int_ ja _long_. Negatiiviset luvut esitetään kahden komplementin esitysmuodossa. Pieni 1-2 tavun data pakataan taulukoihin ja niihin viitataan taulukoissa omilla load- ja store-konekäskyillä. Pinossa ja vakioaltaassa kaikki data on kokonaisina sanoina. 

Liukuluvut esitetään [IEEE liukulukustandardin](https://en.wikipedia.org/wiki/IEEE_floating_point) mukaisesti. Tavallinen liukuluku _float_ on 4 tavua (32 bittiä) ja kaksoistarkkuuden liukuluku _double_ on 8 tavua (64 bittiä).

Merkit esitetään käyttäen etumerkitöntä [Unicode](https://en.wikipedia.org/wiki/Unicode) merkistöä, jossa kukin merkki esitetään kahdella tavulla. Merkkijonot talletetaan kekoon ja niiden viitetiedot vakioaltaaseen. JVM:n merkkejä tai merkkijonoja ei käsitellä tämän enempää.

### Tiedonosoitusmoodi
Tavukoodissa tiedonosoitus on välitöntä tai indeksoitua. Indeksoidut viitteet ovat suhteessa SP-, LV- tai CPP-rekistereihin, mutta rekisteri määräytyy implisiittisesti konekäskyyn mukaan. 

```
iadd               0x60             implisiittiset dataviittaukset pinoon, osoitteet SP, SP-1
bipush 5           0x10 0x05        viite vakioarvoon 5 (välitön operandi)
iconst_1           0x04             implisiittinen viite kokonaislukuvakioon 1
fconst_1           0x0c             implisiittinen viite liukulukuvakioon 1.0
iload 6            0x15 0x06        viite dataan osoitteessa LV+6 (indeksoitu viite)
invokevirtual #37  0xb6 0x00 0x25   viite dataan osoitteessa CPP+37 (indeksoitu viite)
```

Koodiin voi tehdä myös indeksoituja viitteitä suhteessa PC-rekisterin arvoon. Koodiviitteet ovat _tavuosoitteita_, koska käskyjen pituudet voivat 1-17 tavua (yleensä 1-3 tavua). 

```
invokevirtual #37   0xb6 0x00 0x25   PC saa CPP+37:ssä olevan arvon (metodin osoitteen luokassa)
goto -27            0xa7 0x80 0x1B   PC saa arvon PC-27, ehdoton hyppy taaksepäin, 
                                     siirtymän määrä 16-bittisenä etumerkillisenä kokonaislukuna
```

### Käskytyypit
Käytössä on useita tietotyyppejä ja kussakin aritmeettis-loogisessa operaatiossa argumenttien täytyy olla saman kokoisia ja samaa tyyppiä. Tätä varten käskykannassa on useita datan tyypinmuunnoskäskyjä. 

```
i2b       0x91       muuta 32-bittinen kokonaisluku (word) 8-bittiseksi (byte). Etumerkki!
i2f       0x86       muuta 32-bittinen kokonaisluku (word) 32-bittiseksi liukuluvuksi (float)
d2l       0x8f       muuta 64-bittinen liukuluku (double) 64-bittiseksi kokonaisluvuksi (long)
d2i       0x8e       muuta 64-bittinen liukuluku (double) 32-bittiseksi kokonaisluvuksi (int)
```

Tiedonsiirtokäskyistä on jo esitelty käskyjä, joilla kopioidaan 32-bittinen kokonaisluku pinon pinnalle tai siirretään pinon pinnalla olevaa dataa muualle. Vastaavat käskyjä on eri pituisille data-alkioille ja tietotyypeille. Aritmeettis-loogisten lausekkeiden toteutuksissa pinon pinnalla olevaa dataa tarvitsee usein monistaa tai järjestää uuteen järjestykseen.

``` 
iload_2        0x1c            push (LV+2)     tuo kokonaisluku
istore 17      0x36 0x11       pop  (LV+17)    vie kokonaisluku
lload  12      0x16 0x0C       push (LV+12)    tuo pitkä kokonaisluku
fload  10      0x17 0x0A       push (LV+10)    tuo liukuluku
dstore 6       0x39 0x06       pop  (LV+6)     vie pitkä liukuluku
aload 3        0x2d 0x03       push (LV+3)     tuo viite (osoite)
dup            0x59            push (SP)       monista data
dup_x2         0x5b            push (SP-2)     monista data
dup2           0x5c            push (SP)       monista pitkä data 
swap           0x5f            swap(SP, SP-1)  vaihda sanat
bipush 5       0x10 0x05       push (5)        tuo kokonaislukuvakio
iconst_1       0x04            push 1          tuo kokonaislukuvakio
fconst_1       0x0c            push 1.0        tuo liukulukuvakio
getstatic #35  0xb2 0x00 0x23  push (CPP+35)   tuo luokan viite 
```

Taulukkoviitteet ovat JVM:ssä yllättävän vaikeita. Taulukot talletetaan kekoon ja niiden viitetiedot vakioaltaaseen. Taulukoihin viitataan kehyksen muuttujista. Taulukkoviittauksessa pinoon tuodaan ensin taulukon viite (osoite vakioaltaassa) ja indeksi, minkä jälkeen vasta voidaan tehdä varsinainen viittaus taulukkoon. Esimerkiksi, Java-lauseen "a=t[i];" toteutus tavukoodilla voisi olla 

```
aload_1         0x2b          push (LV+1)   taulukon viite t (parametrissa 1)
iload_2         0x1c          push (LV+2)   indeksi i (parametrissa 2)
iaload          0x2e          push (t[i])   korvaa t ja i alkion t[i] arvolla
istore_3        0x3e          pop (LV+3)    taulukon kokonaislukuarvo (paikallinen muuttuja 1)
```

Jos taulukko t sisältäisi neljän tavun kokonaislukujen asemesta yhden tavun kokonaislukuja, niin saman lauseen toteutus olisi 

```
aload_1         0x2b          push (LV+1)   taulukon viite t
iload_2         0x1c          push (LV+2)   indeksi i
baload          0x33          push (t[i])   laajennna tavu samalla sanaksi
istore_3        0x3e          pop (LV+3)    tallenna kokonaislukuarvo sanana
```

Kontrollinsiirtokäskyjä on paljon, koska eri tietotyypeille tarvitaan kullekin omat ehdolliset haarautumiskäskynsä. 

```
goto -27            0xa7 0x80 0x17   PC saa arvon PC-27, ehdoton hyppy taaksepäin
if_icmpgt  +33      0xa3 0x00 0x21   vertaa pinon arvoja. Jos isompi, niin PC saa arvo PC+33
if_icmpeq  -27      0x9f 0x80 0x1B   vertaa pinosn arvoja. Jos sama, niin PC saa arvo PC-27
iflt  +33           0x9b 0x00 0x21   jos pinossa oleva arvo <0, niin PC saa arvon PC+33
lcmp                0x94             vertaa kahta pitkää kokonaislukua, tulos pinoon (+1, 0, -1)
fcmpg               0x96             vertaa kahta liukulukua, tulos pinoon (+1, 0, -1)
invokevirtual #37   0xb6 0x00 0x25   call (CPP+37)
ireturn             0xac             palaa kutsutusta metodista
```

Aritmeettis-loogisia operaatioita on vastaavasti useita, koska yhden tai kahden sanan pituusvaihtoehdoille tarvitaan omat käskynsä.  

```
iadd      0x60     kokonaislukujen (int) yhteenlasku
iand      0x7e     and-operaatio pareittain 32-bittisille loogisille arvoille kokonaisluvuissa
dmul      0x63     64-bittisten liukulukujen kertolasku
ldiv      0x6d     64-bittisten kokonaislukujen jakolasku
lrem      0x71     64-bittisten kokonaislukujen jakolaskun jakojäännös
```

Lisäksi tavukoodiin sisältyy sekalainen joukko muita käskyjä, joista alla on muutamia esimerkkejä.

```
nop             0x00               no operation, kuluttaa vähän aikaa
pop             0x57               ota sana pinosta, heitä pois
arraylength     0xbe               pinon päällä viitatun taulukon pituus
athrow          0xbf               aiheuta keskeytys
new  House      0xbb  0x00 0x03    luo uusi House-tyyppinen olio (instanssi)  
```

## Esimerkki tavukoodin käytöstä
Tarkastellaan Java-kielistä koodinpätkää

```
k = i+5;

if (k=10)
    j=i;
else 
    j=k;
``` 

Kokonaislukuarvoiset muuttujat _i_, _j_ ja _k_ ovat paikallisia muuttujia kehyksen osoitteissa 7, 8 ja 9. Jos tavukoodinen koodinpätkä alkaa tavusta 100 (desimaaliluku), niin tästä voisi generoitua tavukoodi

```
tavukoodi tekstinä       heksadesimaalina

strt  iload i            100:   0x15 0x07
      bipush 5           102:   0x10 0x05
      iadd               104:   0x60   
      dup                105:   0x59             k tarvitaan kohta taas
      istore k           106:   0x36 0x09     
      
      bipush 10          108:   0x10 0x0A        k oli pinossa jo
      if_icmpeq else     110:   0x0f 0x00 0x0A   110+10=120
if10  iload i            113:   0x15 0x07
      istore j           115:   0x36 0x08
      goto done          117:   0xa7 0x00 0x07   117+7=124
else  iload k            120:   0x15 0x09
      istore j           122:   0x36 0x08
done  nop                124:   0x00
```

Tavun 105 käsky _dup_ käyttö on jo vähän koodin optimointia. Lisäoptimoinnilla koodi voisi ehkä olla vielä tehokkaampi. Esimerkiksi,  _istore j_ käskyt voisi yhdistää haarautumisen jälkeen tehtäväksi. Tällöin koodista tulisi kolme tavua lyhyempi, kun nop-käskyäkään ei enää tarvittaisi vain haarautumisen loppuosoitetta varten.

<!-- Quizit 9.2.  JVM -->

<div><quiz id="a6535db1-8473-4344-921d-dbddd2bc77d8"></quiz></div>
<div><quiz id="b0a217ca-8ca8-4801-bc1c-e97df76110e8"></quiz></div>
<div><quiz id="a8f03453-8687-4d1e-9036-df51f690f488"></quiz></div>
