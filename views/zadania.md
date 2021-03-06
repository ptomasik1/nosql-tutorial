#### {% title "Laboratorium" %}

<blockquote>
 {%= image_tag "/images/tao.jpg", :alt => "[Tao]" %}
 <p>
   After three days without programming, life becomes meaningless.
 </p>
 <p class="author"><a href="http://www.canonical.org/~kragen/tao-of-programming.html">The Tao of Programming 2.1</a></p>
</blockquote>

Narzędzia przydatne w trakcie EDA:

* [JQ](http://stedolan.github.io/jq/) –
  a lightweight and flexible command-line JSON processor
* [JSON2CSV](https://github.com/jehiah/json2csv)
* [CSVKIT](http://csvkit.readthedocs.org/en/latest/) –
  a suite of utilities for converting to and working with CSV,
  the king of tabular file formats
* [Elasticdump](https://github.com/taskrabbit/elasticsearch-dump) –
  import and export tool for Elasticsearch

GitHub:

* [Markdown Cheatsheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)
* [AsciiDoctor Documentation](http://asciidoctor.org/docs)
* [Writing on GitHub](https://help.github.com/articles/writing-on-github/)

[Learn X in Y minutes](http://learnxinyminutes.com/):

* [X=R](http://learnxinyminutes.com/docs/r/), Y≈30 min.
* [7 command-line tools for data science](http://jeroenjanssens.com/2013/09/19/seven-command-line-tools-for-data-science.html);
  zob. przykład pokazujący jak importować dane do bazy SQLite
* [Working with data on the command line](http://www.datamazing.co.uk/2014/01/25/working-with-data-on-the-command-line)

Różne rzeczy:

* [The Science of Scientific Writing](http://www.americanscientist.org/issues/id.877,y.0,no.,content.true,page.1,css.print/issue.aspx)

<blockquote>
 {%= image_tag "/images/tukey-john.jpg", :alt => "[John Tukey]" %}
 <p>
  <i>Exploratory Data Analysis</i> (EDA) is an attitude, a state of flexibility,
  a willingness to look for those things that we believe are not there,
  as well as those we believe to be there.
 </p>
 <p class="author">— <a href="http://en.wikipedia.org/wiki/John_Tukey">John Tukey</a></p>
</blockquote>

## Zaliczenie

Co powinny zawierać pliki z rozwiązaniami:

* informacje o komputerze na którym były wykonywane obliczenia: jaki procesor,
 ile pamięci RAM, jaki dysk (HD, SSD)
* jaki system operacyjny + wersja
* wersje użytych programów, sterowników, bibliotek etc.
* czasy wykonania poleceń (użyć polecenia _time_), ile miejsca
 zajmują zaimportowane dane, wykorzystanie RAM (_gnome-system-monitor_)
 i procesorów w trakcie importu etc.
* opis rozwiązania powinien zawierać tabelki, wykresy, zrzuty ekranu etc.

W repozytorium należy umieścić też skrypty wykorzystane w obliczeniach.


### Zadanie 1

* Zainstalować na swoim komputerze bazę [Neo4j](http://neo4j.com/).
* Przeczytać artykuł [Import 10M Stack Overflow Questions into Neo4j In Just 3 Minutes](http://neo4j.com/blog/import-10m-stack-overflow-questions)
 i wykonać kod z tego artykułu.
* Opisać wszystkio to co się zdarzyło w jakimś repozytorium
 w pliku _README.adoc_
 (w formacie [AsciiDoc](http://asciidoctor.org)).


### Zadanie 2

<blockquote>
 {%= image_tag "/images/why_manage_your_data.png", :alt => "[Why Manage Your Data]" %}
 <p>
  You’ll know when you’ve gotten past the data management stage: your
 code starts to become shorter, dealing more with mathematical
 transforms and less with handling exceptions in the data. It’s nice
 to come to this stage. It’s a bit like those fights in Lord of the
 Rings, where you spend a lot of time crossing the murky swamp full of
 nasty creatures, which isn’t that much of a challenge, but you could
 die if you don’t pay attention. Then you get out of the swamp and
 into the evil lair and that’s when things get interesting, short and
 quick.
 </p>
 <p class="author"><a href="http://kaushikghose.wordpress.com/2013/09/26/data-management/">Data Management</a></p>
</blockquote>

Przeczytać artykuł [Exploratory Data Analysis](http://en.wikipedia.org/wiki/Exploratory_Data_Analysis) (EDA)?

{%= image_tag "/images/data-cleaning.png", :alt => "[Data Cleaning]" %}

<!--
Data sharing:

* [How to share data with a statistician](https://github.com/jtleek/datasharing)

Na [Kaggle](https://www.kaggle.com/) znajdziemy dużo interesujących danych.
W sierpniu 2013 Facebook ogłosił konkurs
[Identify keywords and tags from millions of text questions](https://www.kaggle.com/c/facebook-recruiting-iii-keyword-extraction).
Skorzystamy z danych udostępnionych na ten konkurs przez
[Stack Exchange](http://stackexchange.com/):

* [Train.zip](https://www.kaggle.com/c/facebook-recruiting-iii-keyword-extraction/download/Train.zip) – 2.19 GB

Archiwum *Train.zip* zawiera plik *Train.csv* (6.8 GB).
Każdy rekord zawiera cztery pola `"Id","Title","Body","Tags"`:

* `Id` – Unique identifier for each question
* `Title` – The question's title
* `Body` – The body of the question
* `Tags` – The tags associated with the question (all lowercase, should not contain tabs '\t' or ampersands '&')

Przykładowy rekord CSV z pliku *Train.csv*:

    :::csv
    "2","How can I prevent firefox from closing when I press ctrl-w",
    "<p>In my favorite editor […]</p>

    <p>Rene</p>
    ","firefox"

Do testowania swoich rozwiązań można skorzystać ze 101 JSON–ów
[fb101.json](https://github.com/nosql/aggregations-3/blob/master/data/wbzyl/fb101.json).
Wybrałem je losowo po zapisaniu rekordów z *Train.csv* w bazie MongoDB.

   ☀☀☀

* Przykład EDA – konkurs
  [Kaggle bulldozers: Basic cleaning](http://danielfrg.github.io/blog/2013/03/07/kaggle-bulldozers-basic-cleaning/),<br>
  [nagroda dla najlepszego rozwiązania $10,000](http://www.kaggle.com/c/bluebook-for-bulldozers/data)
* Interesujące dane –
  [Detecting Insults in Social Commentary](http://www.kaggle.com/c/detecting-insults-in-social-commentary/),<br>
  [3948 rekordów](http://www.kaggle.com/c/detecting-insults-in-social-commentary/data);
  zob. też Andreas Mueller [Machine Learning with scikit-learn](http://amueller.github.io/sklearn_tutorial/)


2013-09-27T13:04:45.582+0200 check 9 6034196
2013-09-27T13:04:45.689+0200

no. of rows: 6,034,195
min. value for Id: 1
max. value for Id: 6,034,195
no. of unique tags: 42,048
no. of occurrences of tags: 17,409,994
max. no. of tags/question: 5
avg. no. of tags/question: 2.89

% of questions with specified no. of tags:

1 : 13.76
2 : 26.65
3 : 28.65
4 : 19.1

PostgreSQL:

create table RAW_TRAIN(ID BIGINT PRIMARY KEY, TITLE TEXT, BODY TEXT, TAGS TEXT);
copy RAW_TEST from '/home/wbzyl/NN/Facebook-Kaggle/train.csv' csv header;

Do czyszczenia danych, jeśli okaże się to konieczne,
można użyć jednego z tych narzędzi:
[Google Refine](http://code.google.com/p/google-refine/) lub
[Data Wrangler](http://vis.stanford.edu/wrangler/).

Szczególnie polecam obejrzenie tych trzech krótkich filmów:
[Intro 1](http://www.youtube.com/watch?v=B70J_H_zAWM),
[Intro 2](http://www.youtube.com/watch?v=cO8NVCs_Ba0),
[Intro 3](https://www.youtube.com/watch?v=5tsyz3ibYzk).

Również rozwiązania tego zadania należy przygotować jako
[pull request](https://help.github.com/articles/using-pull-requests)
repozytorium [aggregations-3](https://github.com/nosql/aggregations-3).<br>

Na wyższą ocenę należy zoptymizować agregacje.
Na przykład tak jak to opisano w artykule Paula Done’a,
[How to speed up MongoDB Aggregation using Parallelisation](http://pauldone.blogspot.com/2014/03/mongoparallelaggregation.html).

-->

*Zadanie 2a.* Zaimportować, do systemów baz danych:

* MongoDB
* PostgreSQL

jeden ze zbiorów danych wymienionych poniżej.

Duże i niezbyt duże zbiory danych:

* [BookCorpus](http://www.cs.toronto.edu/~mbweb) – two datatsets, 2.5GB i 2.1GB;
  sentences from 11_038 books
* [MongoDB JSON Data](https://github.com/ozlerhakan/mongodb-json-files) –
  a dedicated repository that collects collections to practice/use in MongoDB.
* [I have every publicly available Reddit comment for research. ~ 1.7 billion
  comments @ 250 GB compressed. Any interest in this?](https://www.reddit.com/r/datasets/comments/3bxlg7/i_have_every_publicly_available_reddit_comment)
  Get one month of comments @ 5.5 GB compressed.
* [GrouLens data](http://grouplens.org/datasets):
  - [MovieLens data](http://grouplens.org/datasets/movielens);
  [movielens.org](https://movielens.org) – non-commercial, personalized movie recommendations
* [Jester](http://www.ieor.berkeley.edu/~goldberg/jester-data/) –
  anonymous Ratings Data from the Jester Online Joke Recommender System
* [Stanford Large Network Dataset Collection](https://snap.stanford.edu/data/)
  by Jure Leskovec: Online Reviews (Amazon, Movies, Beer)
* [GeoNames](http://www.geonames.org/export/) i [Postal Codes](http://www.geonames.org/postal-codes/)
* [GetGlue and Timestamped Event Data](http://getglue-data.s3.amazonaws.com/getglue_sample.tar.gz)
  (ok. 11 GB). Dane pochodzą z lat 2007–2012 – tylko filmy i przedstawienia TV.


*Zadanie 2b.* Zliczyć liczbę zaimportowanych rekordów.

*Zadanie 2c.* Policzyć kilka prostych agregacji na zaimportowanych danych.
Wyniki przedstawić graficznie lub w postaci tabelki.

Przykładowo można zliczyć wszystkie tagi i wszystkie różne tagi, policzyć 100
najczęściej i 100 najrzadziej występujących tagów.

W tym zadaniu należy napisać dwa proste skrypty,
po jednym dla MongoDB i PostgreSQL, który to policzą.

W przypadku MongoDB należy użyć jednego ze sterowników
ze strony [MongoDB Ecosystem](http://docs.mongodb.org/ecosystem/).
W przypadku PostgreSQL – należy to zrobić w jakikolwiek sposób.


*Zadanie 2d.* Wyszukać w sieci dane zawierające
[obiekty GeoJSON](http://geojson.org/geojson-spec.html#examples).
Następnie dane zapisać w bazie *MongoDB*.

Dla zapisanych danych przygotować co najmniej 4 różne
[geospatial queries](http://docs.mongodb.org/manual/reference/operator/query-geospatial/)
(co najmniej po jednym, dla obiektów Point, LineString i Polygon).

Przykład pokazujący o co chodzi w tym zadaniu.

Poniższe obiekty Point zapisujemy w pliku places.json:

    :::json places.json
    {"_id": "oakland",  loc: {"type": "Point", "coordinates": [-122.270833,37.804444]}}
    {"_id": "augsburg", loc: {"type": "Point", "coordinates": [10.898333,48.371667]}}
    {"_id": "namibia",  loc: {"type": "Point", "coordinates": [17.15,-22.566667]}}
    {"_id": "australia",loc: {"type": "Point", "coordinates": [135,-25]}}}
    {"_id": "brasilia", loc: {"type": "Point", "coordinates": [-52.95,-10.65]}}

Importujemy je do kolekcji *places* w bazie *test*:

    :::bash
    mongoimport -c places < places.json

Logujemy się do bazy za pomocą *mongo*. Po zalogowaniu
dodajemy geo-indeks do kolekcji *places*:

    :::js
    db.places.ensureIndex({"loc" : "2dsphere"})

Przykładowe zapytanie z *$near*:

    :::js
    var origin = {type: "Point", coordinates: [0,0]}
    db.places.find({ loc: {$near: {$geometry: origin}} })

Wyniki zapytania zapisać w pliku i przekształcić
za pomocą programu **Jq** na [GeoJSON](http://geojson.org/geojson-spec.html).
Wynik zapisać do pliku z rozszerzeniem **.geojson**.
Po push na Github, serwer wyświetli zamiast zawartości mapkę. Oto przykład,
[places.geojson](https://github.com/nosql/aggregations-3/blob/master/data/wbzyl/places.geojson).

Przeczytać [Mapping geoJSON files on GitHub](https://help.github.com/articles/mapping-geojson-files-on-github/)
i mapkę umieścić w pliku z rozwiązaniem zadania.


<blockquote>
 {%= image_tag "/images/hemingway_and_marlins.jpg", :alt => "[Ernest Hemingway and marlins]" %}
 <p>
  Wszystko, co musisz zrobić, to napisać jedno prawdziwe zdanie.
  Napisz najprawdziwsze zdanie, jakie znasz.
 </p>
 <p class="author">— Ernest Hemingway</p>
</blockquote>

## Egzamin


### Zadanie 3

1\. Wyszukać w sieci *interesujące* dane zawierające co najmniej 1_000_000 rekordów/jsonów.

2\. Dane zapisać w bazie MongoDB.

3\. Przygotować w JavaScript co najmniej cztery agregacje korzystające
 z Aggregation Pipeline.

4\. Zaprogramować i wykonać wszystkie agregacje z pkt. 3 w_innym
 niż JavaScript języku programowania. Skorzystać z jednego z driverów
 wymienionych na stronie [MongoDB Drivers](http://docs.mongodb.org/ecosystem/drivers/).


### Zadanie 4

Napisać **trzy** lub **cztery** pary funkcji map-reduce dla dowolnego dużego,
czyli zawierającego co najmniej 1_000_000 rekordów/jsonów, zbioru danych.

Na przykład napisać parę funkcji map-reduce, które wyszukają najczęściej
występujące słowa w pliku [Wikipedia data
PL](http://dumps.wikimedia.org/plwiki/) (aktualny plik z artykułami, ok. 1.3
GB).
