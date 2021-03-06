#### {% title "Widok ≡ Map+Reduce" %}

Definicja z Wikipedii: „MapReduce jest opatentowaną przez Google
platformą do przetwarzania równoległego dużych zbiorów danych
w klastrach komputerów.”
CouchDB ma swoją wersję platformy MapReduce
(a MongoDB swoją). Przyjrzyjmy się jak zaimplementowano
MapReduce w CouchDB.

* [Interactive CouchDB](http://labs.mudynamics.com/wp-content/uploads/2009/04/icouch.html)
* [Introduction to CouchDB Views](http://wiki.apache.org/couchdb/Introduction_to_CouchDB_views)
  (CouchDB Wiki)

## SQL a MapReduce

CouchDB jest dokumentową, a nie relacyjną bazą danych. W relacyjnych
bazach zapisujemy „rekordy”, w bazach dokumentowych – „obiekty”.
Obiekty mogą zawierać inne obiekty i tablice. Obiekty bardziej niż
rekordy nadają się do modelowania złożonych hierarchicznych związków.

CouchDB jest bezschematową bazą danych. W takich bazach
zapytania SQL, na przykład takie:

    :::sql
    SELECT name, email, fax FROM contacts

nie mają sensu, ponieważ takie zapytanie zakłada, że wszystkie dokumenty
*contacts* muszą zawierać pola *name*, *email* i *fax*.
Dokumenty zapisywane w CouchDB nie muszą być zbudowane według
jednego schematu.
*Bonus:* brak schematu oznacza, że niepotrzebne będą migracje,
co jest istotne w wypadku dużych baz, dla których migracje
są kosztowne, albo niemożliwe.

{%= image_tag "/images/couch-mapreduce.png", :alt => "[CouchDB MapReduce]" %}
(źródło *@jrecursive*)

W CouchDB zamiast zapytań mamy widoki:
„Views are the primary tool used for querying and reporting on CouchDB
documents.” Widoki kodujemy zazwyczaj w Javascript albo Erlangu
(ale możemy je też programowac w Ruby, Pythonie).

Widoki zapisujemy w tak zwanych *design documents*. Design documents
są zwykłymi dokumentami. Tym co je wyróżnia jest identyfikator
zaczynający się od **_design/**, na przykład *_design/default*.
Widoki zapisujemy w polu *views* dokumentów projektowych.


## Baza *ls*

Widoki przećwiczymy na przykładzie bazy *ls*
zawierającej kilka aforyzmów Stanisława J. Leca oraz Hugo Steinhausa
(te same co w rozdziale z *Funkcje Show*).
Dla przypomnienia, format przykładowego dokumentu:

    :::json
    {
      "_id": "1",
      "created_at": [2010, 0, 31],
      "quotation": "Szerzenie niewiedzy o wszechświecie musi być także naukowo opracowane.",
      "tags": ["wiedza", "nauka", "wszechświat"]
    }

Bazę (16 dokumentów + 1 dokument _design) replikujemy z mojego serwera
[couch](http://couch.inf.ug.edu.pl/_utils/database.html?ls).
Zakładam, że zreplikowaną bazę nazwano *ls*:

    (remote) http://couch.inf.ug.edu.pl/ls -> (local) ls


## Widok ≈ Map + Reduce (opcjonalne)

W CouchDB są dwa rodzaje widoków:

* tymczasowe (*temporary views*)
* permanentne (*permanent views*)

Tymczasowe widoki będziemy pisać i odpytywać w Futonie,
a do widoków permanentnych wykorzystamy program
[Erica](https://github.com/benoitc/erica).

Widok w CouchDB składa się z dwóch, kolejno wykonywanych, funkcji:

* `map(doc)` – funkcja wykonywana jest na każdym dokumencie
  rezultatem każdego wywołania funkcji powinno być
  „do nothing” albo „*emit(key,value)*”
* `reduce(keys,values,rereduce)` – wywołanie tej funkcji
  poprzedza tzw. „shuffle step”:
  *keys* i *values* są sortowane i grupowane; po wykonaniu
  „shuffle step” argument *rereduce* ustawiany jest na *false*,
  *keys* na *null* i funkcja jest wykonywana tyle razy aż *values*
  zostaną zredukowane do pojedynczego *value*

Zobacz też przykłady zapytań z *group* i *group_level* poniżej.

Zaczynamy od utworzenia szablonu aplikacji za pomocą programu *erica*:

    :::bash
    erica create-app appid=ls lang=javascript
      ==> Couch (create-app)
      Writing ls/_id
      Writing ls/language
      Writing ls/.couchapprc
      Writing ls/views/by_type/map.js

W katalogu *ls/views* zapiszemy widok: *by_date*.

*map.js*:

    :::js by_date/map.js
    function(doc) {
      emit(doc.created_at, doc.quotation.length);
    }

*reduce.js*:

    :::js by_date/reduce.js
    _sum

Następnie usuwamy wygenerowany widok *by_type/map.js*
i podmieniamy zawartość wygenerowanego pliku *_id* na:

    _design/app

a zawartość pliku *.couchapprc* – na:

    :::json
    {
      "env" : {
        "default" : {
          "db" : "http://localhost:5984/ls"
        }
      }
    }

Powyżej użyłem funkcji reduce napisanych w języku Erlang
i dostępnych w widokach pisanych w Javascript.
W wersji 1.1.x CouchDb są trzy takie funkcje:

* `_count` – zwraca liczbę „mapped values”
* `_sum` – zwraca sumę wartości „mapped values”
* `_stats` – zwraca statystyki „mapped values” (sum, count, min, max, …)

**Uwaga:** Widok *by_date* jest już zapisany w bazie. Został
zreplikowany razem z dokumentami. Dlatego poniższy krok możemy pominąć.

Zapisujemy widok w bazie wykonujac, **z katalogu z aplikacją**, push:

    :::bash
    erica -v push


### Odpytywanie widoków

Na konsoli:

    :::bash
    curl http://localhost:5984/ls/_design/app/_view/by_date
    {"rows":[
      {"key": null, "value": 788}
    ]}

i w **Futonie**.

Jak zmienią się wyliczane wyniki, gdy wymienimy funkcję reduce na **_count**?

**Uwaga:** Tę poprawkę w kodzie możemy wykonać w Futonie. Możemy też skorzystać
z tego że program erica ma zaimplementowany prosty serwer www:

    erica web
      ==> ls (web)
      Erica Web started on "http://127.0.0.1:48289"
      ==> Successfully pushed. You can browse it at: http://127.0.0.1:5984/ls/_design/app


### Zrozumieć MapReduce

<blockquote>
 {%= image_tag "/images/hands.jpg", :alt => "[Love and Art]" %}
 <p>
  Wszystko da się zrozumieć poza miłością i sztuką.
 </p>
 <p class="author"><a href="http://www.googleplussuomi.com/timelinetest.php?googleid=114514155722976658302&sort=share">[stara mądrość]</a></p>
</blockquote>

Jak przebiegają wyliczanie map i reduce w widokach?
Możemy to podejrzeć zaglądając do logów CouchDB.

W tym celu skorzystamy z funkcji *log()* CouchDB, która
zapisuje wartości przekazanych argumentów w logu.
Podmieniamy funkcję *_sum* na wersję napisaną w Javascripcie
z dopisanymi wywołaniami *log*:

    :::js reduce.js
    function(keys, values, rereduce) {
        log('REREDUCE: ' + rereduce);
        log('KEYS: ' + JSON.stringify(keys));
        log('VALUES: ' + JSON.stringify(values));
        return sum(values);
    }

Po uruchomieniu widoku w logu znajdziemy:

    Log :: REREDUCE: false
    Log :: KEYS: [
             [[2009,6,15],"3"], [[2010,0,1],"1"],  [[2010,0,1],"4"],  [[2010,0,31],"8"],
             [[2010,0,31],"9"], [[2010,1,20],"2"], [[2010,1,28],"6"], [[2010,1,28],"7"],
             [[2010,3,1],"14"], [[2010,3,1],"15"], [[2010,3,4],"16"], [[2010,11,31],"5"],
             [[2011,1,12],"10"],[[2011,2,10],"11"],[[2011,2,10],"12"],[[2011,2,10],"13"]
           ]
    Log :: VALUES: [30,70,37,34,60,55,27,67,52,51,32,31,57,55,89,41]

    Index update finished for db: ls idx: _design/app

    Log :: REREDUCE: false
    Log :: KEYS: [
             [[2011,2,10],"13"],[[2011,2,10],"12"],[[2011,2,10],"11"],[[2011,1,12],"10"],
             [[2010,11,31],"5"],[[2010,3,4],"16"], [[2010,3,1],"15"], [[2010,3,1],"14"],
             [[2010,1,28],"7"], [[2010,1,28],"6"], [[2010,1,20],"2"], [[2010,0,31],"9"],
             [[2010,0,31],"8"], [[2010,0,1],"4"],  [[2010,0,1],"1"],  [[2009,6,15],"3"]
           ]
    Log :: VALUES: [41,89,55,57,31,32,51,52,67,27,55,60,34,37,70,30]


### *by_tag* – jeszcze jeden widok

*map.js*:

    :::js
    function(doc) {
      for (var k in doc.tags)
        emit(doc.tags[k], null);
    }

*reduce.js*

    _count

Odpytujemy ten widok na konsoli:

    :::bash
    curl http://localhost:5984/ls/_design/app/_view/by_tag
    {"rows":[
      {"key": null, "value": 19}
    ]}

i jeszcze dwa zapytania:

    :::bash
    curl http://localhost:5984/ls/_design/app/_view/by_tag?reduce=false
    curl 'http://localhost:5984/ls/_design/app/_view/by_tag?reduce=false&include_docs=true'

i w **Futonie**.

Widok *by_tag* korzysta z funkcji *_count* napisanej w języku Erlang.
Oto równoważny kod Javascript:

    :::js
    function(keys, values, rereduce) {
      if (rereduce) {
        return sum(values);
      } else {
        return values.length;
      }
    }

Powyżej używamy wartości *rereduce* do wyboru kodu obliczającego liczbę
znaczników (*values==[null,...,null]* jeśli *rereduce==false*).

W dokumentacji [HTTP view](http://wiki.apache.org/couchdb/HTTP_view_API) opisano:

* debugowanie widoków
* view cleanup
* view compaction


## Korzystamy z opcji w zapytaniach do widoków

Odpytując widoki możemy doprecyzować co nas interesuje,
dopisując do zapytania *querying options*.
Poniżej, dla wygody, umieściłem ściągę z opcji
z [HTTP view API](http://wiki.apache.org/couchdb/HTTP_view_API).

Dla żądań GET:

* `key`=*keyvalue*
* `startkey`=*keyvalue*
* `endkey`=*keyvalue*
* `limit`=*max row to return*
* `stale`=ok
* `descending`=true
* `skip`=*number of rows to skip*
* `group`=true
* `group_level`=*integer*
* `reduce`=false
* `include_docs`=true
* `inclusive_end`=true
* `startkey_docid`=*docid*
* `endkey_docid`=*docid*

Dla żądań POST oraz do wbudowanego widoku *_all_docs*:

* {"keys": ["key1", "key2", ...]} – tylko wyszczególnione wiersze widoku

Uwaga: parametry zapytań muszą być odpowiednio cytowane i kodowane.
Jest to uciążliwe. Program *curl* od wersji 7.20 pozwala nam nieco obejść
tę uciążliwość. Należy skorzystać z dwóch opcji `-G` oraz `--data-urlencode`.

Dla przykładu, zapytanie:

    :::bash
    curl http://localhost:5984/ls/_design/app/_view/by_tag -G \
      --data-urlencode startkey='"w"' --data-urlencode endkey='"w\ufff0"' \
      -d reduce=false

zwraca:

    :::json
    {"total_rows":39,"offset":31,"rows":[
    {"id":"15","key":"widzieć","value":null},
    {"id":"1","key":"wiedza","value":null},
    {"id":"5","key":"wnętrze","value":null},
    {"id":"1","key":"wszechświat","value":null},
    {"id":"2","key":"wszyscy","value":null},
    {"id":"13","key":"wynalazki","value":null}
    ]}

Jeśli to zapytanie wpiszemy w przeglądarce, to nie musimy stosować
takich trików z cytowaniem. Wpisujemy po prostu:

    http://localhost:5984/ls/_design/app/_view/by_tag?reduce=false&startkey="w"&endkey="w\ufff0"

Na przykład przeglądarki Firefox i Chrome same kodują zapytania.

Poniżej podaję „wersję przeglądarkową” zapytań oraz zwracane odpowiedzi:

**key** — dokumenty powiązane z podanym kluczem:

    http://localhost:5984/ls/_design/app/_view/by_date?key=[2010,0,1]&reduce=false
    {"total_rows":8,"offset":1,"rows":[
      {"id":"1","key":[2010,0,1],"value":70},
      {"id":"4","key":[2010,0,1],"value":37}
    ]}

**startkey** — dokumenty od klucza:

    http://localhost:5984/ls/_design/app/_view/by_date?startkey=[2011,2]&reduce=false
    {"total_rows":16,"offset":13,"rows":[
      {"id":"11","key":[2011,2,10],"value":55},
      {"id":"12","key":[2011,2,10],"value":89},
      {"id":"13","key":[2011,2,10],"value":41}
    ]}
    http://localhost:5984/ls/_design/app/_view/by_date?startkey=[2011,2]&reduce=true
    {"rows":[
      {"key":null,"value":185}
    ]}

**endkey** — dokumenty do klucza (wyłącznie):

    http://localhost:5984/ls/_design/app/_view/by_date?endkey=[2010,0,31]&reduce=false
    {"total_rows":8,"offset":0,"rows":[
      {"id":"3","key":[2009,6,15],"value":30},
      {"id":"1","key":[2010,0,1],"value":70},
      {"id":"4","key":[2010,0,1],"value":37},
      {"id":"8","key":[2010,0,31],"value":34}
    ]}

**limit** — co najwyżej tyle dokumentów zaczynając od podanego klucza:

    http://localhost:5984/ls/_design/app/_view/by_date?startkey=[2010,0,31]&limit=2&reduce=false
    {"total_rows":16,"offset":3,"rows":[
      {"id":"8","key":[2010,0,31],"value":34},
      {"id":"9","key":[2010,0,31],"value":60}
    ]}

ale *limit* z *reduce* się nie lubią (bug?):

    http://localhost:5984/ls/_design/app/_view/by_date?startkey=[2010,0,31]&limit=2&reduce=true
    {"rows":[
      {"key":null,"value":651}
    ]}
    http://localhost:5984/ls/_design/app/_view/by_date?startkey=[2010,0,31]
    {"rows":[
      {"key":null,"value":651}
    ]}

**skip** – pomiń podaną liczbę dokumentów zaczynając od podanego klucza:

    http://localhost:5984/ls/_design/app/_view/by_date?startkey=[2010,1,20]&skip=2&reduce=false
    {"total_rows":8,"offset":6,"rows":[
      {"id":"7","key":[2010,1,28],"value":67},
      {"id":"5","key":[2010,11,31],"value":31}
    ]}

**include_docs** – dołącz dokumenty do odpowiedzi
(jeśli widok zawiera funkcję reduce, to do zapytania należy dopisać *reduce=false*):

    http://localhost:5984/ls/_design/app/_view/by_date?endkey=[2010]&include_docs=true&reduce=false
    {"total_rows":8,"offset":0,"rows":[
       {"id":"3","key":[2009,6,15],"value":30,
         "doc":{
           "_id":"3",
           "_rev":"5-c460...",
           "created_at":[2009,6,15],
           "quotation":"Za du\u017co or\u0142\u00f3w, za ma\u0142o drobiu.",
           "tags":["orze\u0142","dr\u00f3b"]}}
    ]}

**inclusive_end** – jak wyżej, ale włącznie.

Uwaga: opcji **group**, **group_level** oraz **reduce** mają sens tylko
dla widoków z funkcją *reduce*.

**group** — grupowanie, działa analogiczne do *GROUP BY* z SQL:

    http://localhost:5984/ls/_design/app/_view/by_date?group=true
    {"rows":[
      {"key":[2009,6,15],"value":30},
      {"key":[2010,0,1],"value":107},
      {"key":[2010,0,31],"value":94},
      {"key":[2010,1,20],"value":55},
      {"key":[2010,1,28],"value":94},
      {"key":[2010,3,1],"value":103},
      {"key":[2010,3,4],"value":32},
      {"key":[2010,11,31],"value":31},
      {"key":[2011,1,12],"value":57},
      {"key":[2011,2,10],"value":185}
    ]}

**group_level** – dwa przykłady powinny wyjaśnić o co chodzi:

    http://localhost:5984/ls/_design/app/_view/by_date?group_level=1
    {"rows":[
      {"key":[2009],"value":30},
      {"key":[2010],"value":516},
      {"key":[2011],"value":242}
    ]}
    http://localhost:5984/ls/_design/app/_view/by_date?group_level=2
    {"rows":[
      {"key":[2009,6],"value":30},
      {"key":[2010,0],"value":201},
      {"key":[2010,1],"value":149},
      {"key":[2010,3],"value":135},
      {"key":[2010,11],"value":31},
      {"key":[2011,1],"value":57},
      {"key":[2011,2],"value":185}
    ]}


## Kanoniczny przykład

Na Sigmie jest baza *gutenberg* zawierająca ok. 22&#x200a;000
akapitów z kilkunastu książek pobranych
z [Files Repository](http://www.gutenberg.org/files/)
projektu Gutenberg.

Do pobierania tekstu, podziału go na akpity i zapisaniu
ich w bazie użyto skryptu
{%= link_to "gutenberg2couchdb.rb", "/mapreduce/couch/gutenberg2couchdb.rb" %}
({%= link_to "źródło", "/doc/couchdb/db/gutenberg2couchdb.rb" %}):

    :::bash
    bundle exec ./gutenberg2couchdb.rb -p 5984 \
      the-man-who-knew-too-much.txt http://www.gutenberg.org/cache/epub/1720/pg1720.txt
    bundle exec ./gutenberg2couchdb.rb --help
    ... instalujemy jeszcze kilka książek z listy ...

A teraz obiecany kanoniczny przykład użycia MapReduce,
czyli zliczanie częstości występowania słów.

Do zapisania w bazie widoku, użyjemy programu *erica*:

    :::bash
    erica create-app appid=gutenberg lang=javascript
      ==> couch (create-app)
      Writing gutenberg/_id
      Writing gutenberg/language
      Writing gutenberg/.couchapprc
      Writing gutenberg/views/by_type/map.js

Podmieniamy wygenerowaną zawartość pliku *_id* na:

    _design/app

i usuwamy wygenerowany widok *by_type* i dodajemy widok *wc*:

    views/
    `-- wc
        |-- map.js
        `-- reduce.js

*map.js*:

    :::js map.js
    function(doc) {
      var words = doc.text.toLowerCase().match(/[A-Za-z\u00C0-\u017F]+/g);
      words.forEach(function(word) {
        emit([word, doc.title], 1);
      });
    }

*reduce.js*:

    _sum

**Uwaga:** zakres `\u00C0-\u01fF` obejmuje następujące znaki
z [Unicode 6.0 Character Code Charts](http://www.unicode.org/charts/):

* [Latin-1 Suplement](http://www.unicode.org/charts/PDF/U0080.pdf)
* [Latin Extended-A](http://www.unicode.org/charts/PDF/U0100.pdf)
  (tutaj są polskie diakrytyki).

Zapisujemy widoki w bazie:

    :::bash
    erica -v push http://localhost:5984/gutenberg
    // erica -v push http://Admin:Pass@localhost:5984/gutenberg

Odpytujemy widok *wc* w Futonie.
Eksperymentujemy z różnymi ustawieniami **Grouping** oraz **Reduce**.

*Uwaga:* Przy pierwszym zapytaniu CouchDB wylicza widok.
Na szybkim komputerze trwa to kilka minut.

**Zadanie:** Utworzyć nowy widok, z taką funkcją *map*:

    :::js map.js
    function(doc) {
      var words = doc.text.toLowerCase().match(/[A-Za-z\u00C0-\u017F]+/g);
      words.forEach(function(word) {
        emit([doc.title, word], 1);
      });
    }

i funkcją reduce *_sum*. Czym różni się ten widok od poprzedniego widoku?


## View Collation

*Collation* to kolejność zestawiania, albo schemat uporządkowania.
W [Collaction Specification](http://wiki.apache.org/couchdb/View_collation#Collation_Specification)
opisano jak działa sortowanie zaimplementowanie w CouchDB.

Widoki są zestawiane/sortowane po zawartości pola *key*.

Poniższy przykład, który to próbuje wyjaśnić, pochodzi
z [CouchDB Wiki](http://wiki.apache.org/couchdb/View_collation).

W Futonie tworzymy bazę *coll*, w której zapiszemy dokumenty:

    { "x": " " }
    { "x": "!" }
    ...
    { "x": "~" }

Do zapisania dokumentów w bazie użyjemy modułu
[Cradle](https://github.com/cloudhead/cradle) dla Node.js
i tego skryptu:

    :::javascript collation.js
    var cradle = require('cradle');      // wcześniej instalujemy "cradle"
    var conn = new(cradle.Connection);
    var db = conn.database('coll');

    db.exists(function (err, exists) {
      if (err) {
        console.log('error', err);
      } else if (exists) {
        for (var i = 32; i <= 126; i++) {
          db.save({"x": String.fromCharCode(i)}, function(err, res) {
            if (err) throw err;
          });
        };
      } else {
        console.log('database does not exists.');
        console.log('rerun the script to populate the \"coll\" database');
        db.create();
      }
    });

Teraz, aby zapisać dokumenty w bazie *coll* wystarczy wykonać na konsoli:

    :::bash
    node collation.js

Pozostaje jeszcze utworzyć widok.
Najwygodniej będzie go utworzyć w Futonie.
Przechodzimy do zakładki *View » Temporary View* (widok tymczasowy)
gdzie wpisujemy taką funkcją map:

    :::js
    function(doc) {
      emit(doc.x, null);
    }

(bez funkcji reduce).
Następnie zapisujemy widok w *_design/app* i nazwywamy go *one*.

Oto kilka prostych zapytań pokazujących różnicę między
collation i sortowaniem.

Poniższe zapytania wpisujemy w przeglądarce:

    http://localhost:5984/coll/_design/app/_view/one?startkey="A"
    http://localhost:5984/coll/_design/app/_view/one?startkey="A"&limit=8
    http://localhost:5984/coll/_design/app/_view/one?startkey="A"&limit=8&descending=true
    http://localhost:5984/coll/_all_docs?startkey="A"&endkey="E"

A tak odpytujemy widok tymczasowy na konsoli:

    :::bash
    curl -X POST http://localhost:5984/coll/_temp_view \
      -H "Content-Type: application/json" -d '
    {
      "map": "function(doc) { emit(doc.x, null); }"
    }'

Zwracana lista dokumentów jest posortowana po kluczu (tutaj, po *doc.x*).
Porządek dokumentów określony jest przez
[unicode collation algorithm](http://www.unicode.org/reports/tr10/).
Dlatego mówimy *view collation*, a nie *view sorting*.


# Przykłady różne

… ten wykład jest już przydługi… poniższe przykłady należałoby przenieść
do nowego wykładu.

<!-- http://www.alanwood.net/unicode/arrows.html -->


## *Rock* – wykonywanie kodu po stronie klienta

Do tej pory pisaliśmy kod, który uruchamialiśmy po stronie serwera.
W przykładzie poniżej, kod wykonamy po stronie klienta –
na konsoli przeglądarki.

Skorzystamy z biblioteki *jquery.couch.js* (wykorzystanej też w Futonie).

Zaczniemy od utworzenia w bazie *rock*, w zakładce
*View » Temporary View* widoku,
który po przetestowaniu, zapiszemy w *_design/app* jako *connections*.

Funkcja map do wklejenia w oknie *Map Function*:

    :::javascript
    function(doc) {
      for (var who in doc.similar)
        emit(doc._id, doc.similar[who]);
    }

Funkcja reduce do wklejenia w oknie *Reduce Function*:

    _count

Po zapisaniu widoku w bazie, odpytujemy go w Futonie i na konsoli:

    :::bash
    curl http://localhost:5984/rock/_design/app/_view/connections
    curl http://localhost:5984/rock/_design/app/_view/connections?reduce=false
    curl http://localhost:5984/rock/_design/app/_view/connections?group=true

Poniższy kod wpiszemy i wykonamy na konsoli przeglądarki,
czyli wykonamy go **po stronie klienta**.

W przeglądarce, w zakładce z Futonem otwieramy okno z konsolą,
gdzie wpisujemy:

    :::javascript
    db = $.couch.db('rock');

    db.view('app/connections', {
      reduce: false,
      success: function(data) {
        var rows = data.rows;
        rows.forEach(function(o){
          console.log(JSON.stringify(o));
        });
      }
    });

I jeszcze jedno zapytanie do wykonania na konsoli przeglądarki:

    :::javascript
    db = $.couch.db('rock');

    db.view('app/connections', {
      reduce: false,
      key: 'jimmypage',
      success: function(data) {
        console.log( data.rows.map(function(o) { return o.value; }) );
      }
    });

Oto wynik wykonania tego zapytania:

    :::js
    [
      "coverdalepage", "free", "humblepie", "iangillan", "jeffbeck",
      "jimmypagetheblackcrowes", "johnpauljones", "keithrichards", "ledzeppelin", "micktaylor",
      "pageplant", "pattravers", "robertplant", "robertplantandthestrangesensation", "robintrower",
      "rorygallagher", "tenyearsafter", "thefirm", "thehoneydrippers", "theyardbirds"
    ]


## Movies

Zaczynamy od replikcji bazy *movies* (9579 dokumentów, ok. 16 MB) do
swojego CouchDB:

    (remote) http://couch.inf.ug.edu.pl/movies ↭ (local) movies

W bazie *movies* zapiszemy następujący widok
(ponownie skorzystamy z zakładki *View » Temporary View* w Futonie):

    :::javascript
    // Map Function
    function(doc) {
      emit(doc.rating, {"count": 1, "rating_total": doc.rating});
    }
    // Reduce Function
    function(keys, values, rereduce) {
      log('REREDUCE: ' + rereduce);
      log('KEYS: ' + keys);
      log('VALUES: ' + JSON.stringify(values));

      var count = 0;
      values.forEach(function(element) { count += element.count; });
      var rating = 0;
      values.forEach(function(element) { rating += element.rating_total; });

      return {"count": count, "rating_total": rating};
    }

Widok zapisujemy w *_design/app* pod nazwą *rating_avg*.
(Zapisujemy też różne rzeczy w logu CouchDB.)

Aby wyliczyć widok w Futonie w zakładce *View* wybieramy *rating_avg*
i czekamy ok. minuty na przeliczenie widoku przez CouchDB.
Następnie odpytujemy widok *rating_avg* w Futonie.

Odpytywanie widoku *rating_avg* na konsoli:

    :::bash
    curl localhost:5984/movies/_design/app/_view/rating_avg  # average rating ≅ 7.67
    curl localhost:5984/movies/_design/app/_view/rating_avg?group_level=1


# Złączenia w bazach CouchDB

…czyli co wynika z *Collation Specification*.

Przykłady poniżej pochodzą z artykułu
Christophera Lenza, [CouchDB „Joins”](http://www.cmlenz.net/archives/2007/10/couchdb-joins),
gdzie autor omawia trzy sposoby
**modelowania powiązań między postami a komentarzami**:
„How you’d go about modeling a simple blogging system with «post» and
«comments» entities, where any blog post might have many comments.”

Dodatkowo warto zajrzeć do
[CouchDB JOINs Redux](http://blog.couchone.com/post/446015664/whats-new-in-apache-couchdb-0-11-part-two-views).


### Sposób 1: komentarze inline

Utworzymy bazę *blog-1* zawierającą następujące dokumenty:

    :::json blog-1.json
    {
      "docs": [
        {
          "author": "jacek",
          "title": "Refactoring User Name",
          "content": "Learn how to clean up your code through refactoring.",
          "comments": [
            {"author": "agatka", "content": "thanks!"},
            {"author": "bolek",  "content": "Very very nice idea! Thanks for this post."}
          ]
        },
        {
          "author": "jacek",
          "title": "Restricting Access",
          "content": "You will learn how to lock down the site.",
          "comments": [
            {"author": "lolek", "content": "If God would exists it will be you... thanks for the screencast."},
            {"author": "bolek",  "content": "Fixed...sorry for the spam."}
          ]
        }
      ]
    }

Dokumenty zapiszemy korzystając z programu *curl*:

    :::bash
    curl -X POST -H "Content-Type: application/json" -d @blog-1.json http://localhost:5984/blog-1/_bulk_docs

<!--

Do bazy dodamy widok zwracający wszystkie komentarze, posortowane po polu *author*:

    :::javascript
    function(doc) {
      for (var i in doc.comments) {
        emit(doc.comments[i].author, doc.comments[i].content);
      }
    }

-->

Jakie są argumenty za i przeciw takiej implementacji powiązań
między postami i komentarzami?

Aby dodać komentarz do posta, należy:

1. pobrać dokument z bazy
2. dodać nowy komentarz do dokumentu
3. umieścić uaktualniony dokument w bazie

„Now if you have multiple client processes adding comments at roughly
the same time, some of them will get a 409 Conflict error on step 3
(that's optimistic concurrency in action).”


### Sposób 2: komentarze w osobnych dokumentach

Utworzymy bazę *blog-2* w której zapiszemy osobno posty i komentarze:

    :::json blog-2-posts.json
    {
      "docs": [
        {
          "_id": "01",
          "type": "post",
          "author": "jacek",
          "title": "Refactoring User Name",
          "content": "Learn how to clean up your code through refactoring."
        },
        {
          "_id": "02",
          "type": "post",
          "author": "jacek",
          "title": "Restricting Access",
          "content": "You will learn how to lock down the site."
        }
      ]
    }

oraz komentarzy:

    :::json blog-2-comments.json
    {
      "docs": [
        {
          "_id": "11",
          "type": "comment",
          "post": "01",
          "author": "agatka",
          "content": "Thanks!"
        },
        {
          "_id": "12",
          "type": "comment",
          "post": "01",
          "author": "bolek",
          "content": "Very very nice idea! Thanks for this post."
        },
        {
          "_id": "13",
          "type": "comment",
          "post": "02",
          "author": "lolek",
          "content": "If God would exists it will be you... thanks for the screencast."
        },
        {
          "_id": "14",
          "type": "comment",
          "post": "02",
          "author": "bolek",
          "content": "Fixed...sorry for the spam."
        }
      ]
    }

Dokumenty zapisujemy w bazie:

    :::bash
    curl -X POST -H "Content-Type: application/json" -d @blog-2-posts.json http://localhost:5984/blog-2/_bulk_docs
    curl -X POST -H "Content-Type: application/json" -d @blog-2-comments.json http://localhost:5984/blog-2/_bulk_docs

Do bazy dodamy widok **/_design/app/by_post**:

    :::javascript
    function(doc) {
      if (doc.type == "comment") {
        emit(doc.post, {author: doc.author, content: doc.content});
      }
    }

Odpytanie tego widoku daje komentarze zgrupowane po zawartości pola *post*:

    http://localhost:5984/blog-2/_design/app/_view/by_post
    {"total_rows":4,"offset":0,"rows":[
    {"id":"11","key":"01","value":{"author":"agatka","content":"Thanks!"}},
    {"id":"12","key":"01","value":{"author":"bolek","content":"Very very nice idea..."}},
    {"id":"13","key":"02","value":{"author":"lolek","content":"If God would exists..."}},
    {"id":"14","key":"02","value":{"author":"bolek","content":"Fixed...."}}
    ]}

Dlatego wszystkie komentarze do posta "02" pobieramy tak:

    http://localhost:5984/blog-2/_design/app/_view/by_post?key="02"
    {"total_rows":4,"offset":2,"rows":[
      {"id":"13","key":"02","value":{"author":"lolek","content":"If God would exists..."}},
      {"id":"14","key":"02","value":{"author":"bolek","content":"Fixed...."}}
    ]}

Jeśli teraz pobierzemy post "02", to mamy komplet.
Dwa żądania HTTP i mamy post i wszystkie do niego komentarze.

Niestety to o jedno żądanie HTPP za dużo! Dlaczego?

Przechowywanie osobno postów i komentarzy ma też wady:

„Imagine you want to display a blog post with all the associated
comments on the same web page. With our first approach, we needed
just a single request to the CouchDB server, namely a GET request to
the document. With this second approach, we need two requests: a GET
request to the post document, and a GET request to the view that
returns all comments for the post.”


### Ultimate solution – using the power of collation

„What we'd probably want then would be a way to join the blog post and
the various comments together to be able to retrieve them with
**a single HTTP request**.”

Rozważmy taki widok (baza *blog-2*):

    :::javascript map.js
    // Funkcja Map
    function(doc) {
      if (doc.type == "post") {
        emit([doc._id, 0], null);
      } else if (doc.type == "comment") {
        emit([doc.post, 1], null);
      }
    }

Jak on działa? Aby to zobaczyć, zapiszmy widok w *_design/app*
jako *join* i odpytajmy go:

    http://localhost:5984/blog-2/_design/app/_view/join?key=["02",0]&include_docs=true
      {
         "total_rows":6, "offset":3, "rows": [
           {
             "id": "02", "key": ["02",0], "value": null,
             "doc":{
               "_id":"02", "_rev":"1-6844c44b92b8facdd1a72d294baac302",
                type": "post", "author": "jacek", "title": "Restricting Access",
                "content": "You will learn how to lock down the site."
             }
           }
         ]
      }

a nastepnie odpytajmy go tak i tak:

    http://localhost:5984/blog-2/_design/app/_view/join?key=["02",1]&include_docs=true
    http://localhost:5984/blog-2/_design/app/_view/join?startkey=["02"]&endkey=["03"]&include_docs=true

Bingo! To jest to! Zapytanie zwraca nam post (tutaj z *id* równym "02")
i wszystkie komentarze do niego w **jednym żądaniu HTTP**.


## TODO

1. Wyciąć rozdział ze złączeniami do osobnego wykładu.
2. Zainstalować wtyczkę [**JSONView**](http://jsonview.com/) do Firefoksa.
