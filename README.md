# Predikcija srčane slabosti primenom metoda mašinskog učenja

Teodora Danilović 4004/2025 i Nina Petković 4005/2025

## Opis projekta

Cilj je da se na osnovu jedanaest kliničkih pokazatelja, koji se prikupljaju na
rutinskom kardiološkom pregledu, predvidi da li pacijent ima srčano oboljenje.
U pitanju je binarna klasifikacija. Pored samog kvaliteta predikcije, zanimalo nas je i kako oceniti model kada ima svega 918 instanci, koliko regularizacija doprinosi svakom
pojedinačnom algoritmu i kako izabrati arhitekturu neuronske mreže.

## Skup podataka

Koristili smo *Heart Failure Prediction* skup sa
[Kaggle-a](https://www.kaggle.com/datasets/fedesoriano/heart-failure-prediction),
nastao spajanjem pet kardioloških skupova iz UCI repozitorijuma (Cleveland,
Mađarska, Švajcarska, Long Beach VA i Stalog). Ima 918 pacijenata, 11 atributa
i nema duplikata. Ciljna promenljiva `HeartDisease` raspoređena je u odnosu 508
bolesnih prema 410 zdravih.

Numerički atributi su `Age`, `RestingBP`, `Cholesterol`, `MaxHR` i `Oldpeak`, a
kategorički `Sex`, `ChestPainType`, `FastingBS`, `RestingECG`, `ExerciseAngina`
i `ST_Slope`. Detaljan opis svakog atributa je u svesci 01.

**Problem sa nulama.** Iako pri početnoj analizi nisu prijavljene nedostajuće vrednosti, u
koloni `Cholesterol` ima 172 nule, a u `RestingBP` jedna. Nijedna nije
fiziološki moguća, pa se radi o merenjima koja nisu obavljena. Zamenili smo ih
medijanom i dodali binarni atribut `Cholesterol_izmeren`. Ispostavilo se da i
sama činjenica da merenje nedostaje nosi informaciju, jer je među tim
pacijentima 88.4% bolesnih naspram 47.7% u ostatku skupa.

## Tok rada

**Pretprocesiranje.** Nule u kolonama `Cholesterol` i `RestingBP` nisu
fiziološki moguće, pa smo ih zamenili medijanom izmerenih vrednosti i dodali
indikator `Cholesterol_izmeren`. Kategoričke atribute smo transformisali, čime se dobija 16 atributa, a numeričke standardizovali. Objekat `scaler` se uči isključivo nad skupom za
treniranje, a nad validacionim i test skupom se već naučena transformacija samo
primenjuje.

**Izbor metaparametara.** Rađen je na dva načina. Kod logističke regresije,
SVM-a i KNN-a metaparametri su birani na izdvojenom validacionom skupu, petljom
po vrednostima uz pamćenje najboljeg rezultata (`best_C`, `best_gamma`,
`best_f1_score`). Kod slučajne šume, AdaBoost-a i XGBoost-a korišćena je klasa
`GridSearchCV` sa desetostrukom unakrsnom validacijom.

**Evaluacija.** Skup smo podelili na skup za treniranje, validacioni skup i
skup za testiranje od 184 instance. Metaparametri su birani na validacionom
skupu i unakrsnom validacijom nad skupom za treniranje, a skup za testiranje
nismo koristili sve do sveske 04, gde na njemu poredimo sve modele. Pratili smo
tačnost, preciznost, odziv, F1 meru i AUC.

## Modeli

Poredili smo logističku regresiju, SVM sa RBF kernelom, KNN, slučajne šume,
AdaBoost, XGBoost i potpuno povezanu neuronsku mrežu. Svaki algoritam je
napravljen u dve varijante, bez ikakve kontrole složenosti i sa regularizacijom
čiji su metaparametri izabrani pretragom.

Regularizacija kod svakog modela znači nešto drugo: `l2` kazna nad
koeficijentima kod logističke regresije, `C` i `gamma` kod SVM-a, broj suseda
kod KNN-a, ograničena dubina i broj atributa po podeli kod slučajne šume,
plitka bazna stabla uz skraćivanje koraka kod AdaBoost-a, a `reg_lambda` i
ograničena dubina kod XGBoost-a. Kod mreže smo posmatrali `Dropout` slojeve,
`l2` kaznu nad težinama i samu veličinu arhitekture.

## Rezultati

Svi rezultati dobijeni su sa `random_state` postavljenim na 7.

**Uticaj regularizacije.** Pet od šest klasičnih modela bez regularizacije
nauči skup za treniranje napamet, sa tačnošću 1.00, a na validacionom skupu
padne na 0.82 do 0.90. Izuzetak je logistička regresija, koja je sa 16
koeficijenata previše jednostavna da bi se preprilagodila, pa joj je tačnost
na skupu za treniranje 0.881.

Dobitak od regularizacije, meren desetostrukom unakrsnom validacijom, najveći
je kod modela koji su bili najfleksibilniji: AdaBoost dobija 0.108 tačnosti,
SVM 0.075, zatim XGBoost 0.030, KNN 0.029 i slučajna šuma 0.026. Kod logističke
regresije dobitka nema, jer je izbor pao na `C` = 1, za koje je model praktično
isti kao onaj bez kazne. Posle regularizacije svi modeli su u uskom opsegu
tačnosti, od 0.866 do 0.879

**Neuronska mreža.** Poredili smo pet arhitektura. Tačnost na skupu za
treniranje raste sa veličinom mreže, sa 0.886 kod jednog sloja od 8 neurona na
1.000 kod mreže sa 100 i 40 neurona, dok razmak u odnosu na validacioni skup
raste sa 0.015 na 0.150. Najbolja tačnost na validacionom skupu, 0.878, dobijena
je kod male mreže sa jednim slojem od 16 neurona.

Na arhitekturi sa 32 i 16 neurona regularizacija podiže tačnost na validacionom
skupu sa 0.837 na 0.871, i to podjednako sa `Dropout` slojevima i sa
kombinacijom `Dropout` slojeva i `l2` kazne, uz istovremeni pad tačnosti na
skupu za treniranje sa 0.969 na 0.906. Konačna mreža na skupu za testiranje ima
tačnost 0.875.

**Konačno poređenje.** Svih šest algoritama, u obe varijante, kao i obe
neuronske mreže, uporedili smo na skupu za testiranje od 184 instance koji
ranije nije korišćen.

Jedan od boljih je SVM sa regularizacijom, sa F1 merom 0.886 i tačnošću 0.870, a
odmah do njega su AdaBoost sa regularizacijom (0.883) i neuronska mreža
(0.882). Najgori je SVM bez regularizacije, sa F1 merom 0.830.

Najzanimljiviji nalaz je da su najbolji i najgori model **isti algoritam**, i
da ih razlikuje jedino regularizacija. Matrice konfuzije pokazuju odakle
razlika: obe varijante jednako rade sa zdravim pacijentima, 67 tačno i 15
pogrešno, ali regularizovana propušta 9 bolesnih, a neregularizovana 19.

Regularizacija na test skupu najviše donosi SVM-u (0.056 F1 mere), AdaBoost-u
(0.052) i KNN-u (0.036). Kod logističke regresije rezultat je identičan, a kod
slučajne šume i XGBoost-a razlika je 0.002 u korist varijante bez
regularizacije, što je manje od jedne instance i u granicama šuma.

**Prag odlučivanja.** Lažno negativna predikcija je opasnija od lažno
pozitivne, pa smo proverili šta se dešava kada se prag spusti sa 0.5. Na pragu
0.4 odziv raste sa 0.902 na 0.922, uz pad preciznosti sa 0.868 na 0.832. Na
pragu 0.3 odziv je 0.931, ali preciznost pada na 0.798.

## Zaključak

Regularizacija je neophodna, ali koliko donosi zavisi od modela. Najviše
dobijaju modeli koji se bez nje preprilagode, a najmanje logistička
regresija koja je i sama dovoljno jednostavna. Najbolje se to vidi po tome što
su i najbolji i najgori model isti algoritam, SVM, a razlikuje ih samo
regularizacija. Kod neuronske mreže je izbor arhitekture bio važniji od same regularizacije.
Male mreže su se pokazale bolje od velikih, a mreža nije nadmašila klasične
modele, što je za mali tabelarni skup očekivano. Priprema podataka je donela dosta. Atribut `Cholesterol_izmeren`, koji smo
uveli da bismo sačuvali informaciju o tome da merenje nije obavljeno, ima
korelaciju sa oboljenjem od -0.32, dok je sama vrednost holesterola sa 0.08
praktično beskorisna. Najjaču vezu sa oboljenjem imaju nagib ST segmenta
(`ST_Slope`), angina izazvana naporom i ST depresija, i na njih se najviše
oslanjaju i slučajna šuma i logistička regresija. Ograničenja su mali skup nastao spajanjem pet izvora sa različitim protokolima,
izrazito neuravnotežen odnos polova (725 muškaraca prema 193 žene) i relativna
starost podataka.

## Literatura

* *Heart Failure Prediction Dataset*, Kaggle, 2021.
* Izvodi sa predavanja i vežbi
