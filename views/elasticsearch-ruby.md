#### {% title "ElasticSearch & Ruby" %}

<blockquote>
 {%= image_tag "/images/daniel_kahneman.jpg", :alt => "[Daniel Kahneman]" %}
 <p><b>The hallo effect</b> helps keep explanatory narratives
  simple and coherent by exaggerating the consistency
  of evaluations: good people do only good things
  and bad people are all bad.
 </p>
 <p class="author">— Daniel Kahneman</p>
</blockquote>

Do eksperymentów poniżej użyjemy rzeczywistych danych.
Będziemy zbierać statusy z Twittera.
Nie będziemy zbierać ich „jak leci”, tylko
te które zawierają interesujące nas słowa kluczowe.

Do filtrowania statusów skorzystamy
z [stream API](https://dev.twitter.com/docs/streaming-api):

* [public streams](https://dev.twitter.com/docs/streaming-apis/streams/public)
* [POST statuses/filter](https://dev.twitter.com/docs/api/1.1/post/statuses/filter) –
  tutaj należy skorzystać z **Oauth tool** za pomocą którego
  generujemy przykładowe zapytanie dla programu *curl*

W pliku *tracking* wpisujemy tę linijkę:

    :::ruby
    track=mongodb,elasticsearch,couchdb,neo4j,redis,emberjs,meteorjs,d3js

**Uwaga:** Jeśli do listy dopiszemy słowo **wow**, wpisywane w wielu
statusach, zostaniemy zalani tweetami – **wow!**

Jak widać statusy zawierają wiele pól i tylko kilka z nich zawiera
interesujące dane. Niestety, na konsoli trudno jest czytać interesujące nas fragmenty.
Są one wymieszane z jakimiś technicznymi rzeczami, np.
*profile_sidebar_fill_color*, *profile_use_background_image* itp.

Dlatego, przed wypisaniem statusu na ekran, powinniśmy go „oczyścić”
ze zbędnych rzeczy. Zrobimy to za pomocą skryptu w Ruby.

Poniżej będziemy korzystać z następujących gemów:

    gem install elasticsearch twitter colored oj

{%= image_tag "/images/twitter_elasticsearch.jpeg", :alt => "[Twitter -> ElasticSearch]" %}

<blockquote>
<p>
  <h3>Access Rate Limiting</h3>
  <p>Each account may create only one standing connection to the
  Streaming API. Subsequent connections from the same account may
  cause previously established connections to be
  disconnected. Excessive connection attempts, regardless of success,
  will result in an automatic ban of the client's IP
  address. Continually failing connections will result in your IP
  address being blacklisted from all Twitter access.
</p>
  <p class="author"><a href="https://dev.twitter.com/docs/rate-limiting/1.1">…more on rate limiting</a></p>
</blockquote>

Zaczniemy od skryptu działającego podobnie do polecenia z *curl*:

    :::ruby fetch-tweets-simple.rb
    require "bundler/setup"

    require 'twitter'  # version at least 5.0.0
    require 'colored'
    require 'yaml'

    credentials = ARGV
    unless credentials[0]
      puts "\nUsage:"
      puts "\truby #{__FILE__} FILE_WITH_TWITTER_CREDENTIALS"
      puts "\truby fetch-tweets-simple.rb ~/.credentials/twitter.yml\n\n"
      exit(1)
    end

    begin
      raw_config = File.read File.expand_path(credentials[0])
      twitter = YAML.load(raw_config)
    rescue
      puts "\n\tError: problems with #{credentials}\n".red
      exit(1)
    end

    def handle_tweet(s)
      puts "#{s[:created_at].to_s.cyan}:\t#{s[:text].yellow}"
    end

    # https://dev.twitter.com/apps
    #   My applications: Elasticsearch NoSQL
    client = Twitter::Streaming::Client.new do |config|
      config.consumer_key        = twitter['consumer_key']
      config.consumer_secret     = twitter['consumer_secret']
      config.access_token        = twitter['oauth_token']
      config.access_token_secret = twitter['oauth_token_secret']
    end

    topics = %w[
      deeplearning
      mongodb elasticsearch couchdb neo4j redis
      emberjs meteorjs rails d3js
    ]
    client.filter(track: topics.join(",")) do |status|
      handle_tweet status
    end

Szablon pliku YAML z *credentials*:

    :::yaml
    ---
    consumer_key: AAAAAAAAAAAAAAAAAAAAA
    consumer_secret: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
    oauth_token: CCCCCCCC-CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
    oauth_token_secret: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD

Skrypt ten uruchamiamy na konsoli w następujący sposób:

    :::bash
    ruby fetch-tweets-simple.rb ~/twitter.yml


## Twitter Stream ⟿ ElasticSearch

Do pobierania statusów i zapisywania ich w bazie wykorzystamy skrypt
{%= link_to "fetch-tweets.rb", "/elasticsearch/tweets/fetch-tweets.rb" %}.
Przed zapisaniem w bazie JSON-a ze statusem, skrypt
usuwa z niego niepotrzebne nam pola i spłaszcza jego strukturę.

**Uwaga:** Z Twitterem możemy zestawić tylko jeden strumień
(co można wyczytać w dokumentacji do API Twittera; sprawdzić to dla v1.1 API).
Dlatego zbierzemy wszystkie statusy z interesujących nas kategorii do jednego strumienia.
Rozdzielimy go na poszczególne typy, przed zapisaniem w bazie.
Wykorzystamy w tym celu [perkolację](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-percolate.html).

**Uwaga:** W wersji 1.0.0 Elasticsearch percolator zamieniono na
[distributed percolator](http://www.elasticsearch.org/guide/en/elasticsearch/reference/master/search-percolate.html).

Co oznacza *percolation*? przenikanie, przefiltrowywanie, perkolacja?
„Let's define callback for percolation.
Whenewer a new document is saved in the index, this block will be executed,
and we will have access to matching queries in the `Tweet#matches` property.”

Zanim zaczniemy zapisywać statusy w bazie, zdefinujemy i zapiszemy
w bazie ElasticSearch *mapping* dla statusów.

Co to jest *mapping*?
„Mapping is the process of defining how a document should be mapped to
the Search Engine, including its searchable characteristics such as
which fields are searchable and if/how they are tokenized.”

    :::ruby create-mapping-and-percolate.rb
    require "bundler/setup"

    require "elasticsearch"
    require "colored"

    mapping = {
      _ttl:            { enabled: true,  default: '16w'                               },
      properties: {
        created_at:    { type: 'date',   format: 'YYYY-MM-dd HH:mm:ss Z'              },
        text:          { type: 'string', index:  'analyzed',    analyzer:  'snowball' },
        screen_name:   { type: 'string', index:  'not_analyzed'                       },
        hashtags:      { type: 'string', index:  'not_analyzed'                       },
        urls:          { type: 'string', index:  'not_analyzed'                       },
        user_mentions: { type: 'string', index:  'not_analyzed'                       }
      }
    }

    topics = %w[
      deeplearning
      mongodb elasticsearch couchdb neo4j redis
      emberjs meteorjs rails
      d3js
    ]

    elasticsearch_client = Elasticsearch::Client.new log: true
    elasticsearch_client.perform_request :put, '/tweets'       # create ‘tweets’ index

    topics.each do |topic|
      elasticsearch_client.indices.put_mapping index: 'tweets', type: topic,
          body: { topic: mapping }
    end
    elasticsearch_client.indices.refresh index: 'tweets'

    # register several queries for percolation against the tweets index
    topics.each do |topic|
      elasticsearch_client.index index: '_percolator', type: 'tweets', id: topic,
          body: { query: { query_string: { query: topic } } }
    end
    elasticsearch_client.indices.refresh index: '_percolator'

Teraz uruchamiamy skrypt *create-index-and-percolate.rb*,
sprawdzamy czy *mapping* zostało zapisane w bazie
i uruchamiamy skrypt *fetch-tweets.rb*:

    :::bash
    ruby create-index-and-percolate.rb
    curl 'http://localhost:9200/tweets/_mapping?pretty'
    ruby fetch-tweets.rb

Czekamy aż kilka statusów zostanie zapisanych w bazie
i wykonujemy na konsoli kilka prostych zapytań:

    :::bash
    curl -s 'localhost:9200/tweets/_count'
    curl -s 'localhost:9200/tweets/redis/_count'
    curl -s 'localhost:9200/tweets/_search?q=*&sort=created_at:desc&size=2&pretty'
    curl -s 'localhost:9200/tweets/_search?size=2&sort=created_at:desc&pretty'
    curl -s 'localhost:9200/tweets/_search?_all&sort=created_at:desc&pretty'

Do wygodnego przeglądania statusów możemy użyć aplikacji
[tweets-elasticsearch](https://github.com/wbzyl/tweets-elasticsearch), którą instalujemy jako *site plugin*.


## Faceted search, czyli wyszukiwanie fasetowe

[Co to są fasety?](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-facets.html)
„Facets provide aggregated data based on a search query. In the
simplest case, a terms facet can return facet counts for various facet
values for a specific field. ElasticSearch supports more facet
implementations, such as statistical or date histogram facets.”

**Uwaga:** Pole wykorzystywane do obliczeń fasetowych musi być typu:

* *numeric*
* *date* lub *time*
* *be analyzed as a single token*

Od prostych zapytań do zapytań z fasetami:

    :::bash
    curl -s 'localhost:9200/tweets/_count?q=redis&pretty'
    curl -s 'localhost:9200/tweets/_search?q=redis&pretty'

    curl -s 'localhost:9200/tweets/_search?pretty' -d '
    {
      "query": { "query_string": {"query": "redis"} },
      "sort": { "created_at": { "order": "desc" } }
    }'
    curl -s 'localhost:9200/tweets/_search?pretty' -d '
    {
      "query": { "query_string": {"query": "redis"} },
      "sort": { "created_at": { "order": "desc" } },
      "facets": { "hashtags": { "terms":  { "field": "hashtags" } } }
    }'
    curl -s 'localhost:9200/tweets/_search?pretty' -d '
    {
      "query": { "match_all": {} },
      "sort": { "created_at": { "order": "desc" } },
      "facets": { "hashtags": { "terms":  { "field": "hashtags" } } }
    }'
    curl -s 'localhost:9200/tweets/_search?size=0&pretty' -d '
    {
      "facets": { "hashtags": { "terms":  { "field": "hashtags" } } }
    }'

A tak wygląda „fasetowy” JSON:

    :::json
    { ... cut ...
      "facets" : {
         "hashtags" : {
            "_type" : "terms",
            "missing" : 167,
            "total" : 198,
            "other" : 127,
            "terms" : [
               { "term" : "dbts2013", "count" : 13 },
               { "term" : "nosql", "count" : 9 },
               { "term" : "couchdb", "count" : 9 },
               { "term" : "mongodb", "count" : 7 },
               { "term" : "Rails", "count" : 7 },
               { "term" : "cassandra", "count" : 6 },
               { "term" : "redis", "count" : 5 },
               { "term" : "rails", "count" : 5 },
               { "term" : "jobs", "count" : 5 },
               { "term" : "d3js", "count" : 5 }
            ]
          }
        }
      }

[Search API – Facets](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-facets.html):

* `missing` : the number of documents which have no value for the faceted field
* `total` : the total number of terms in the facet
* `other` : the number of terms not included in the returned facet

Effectively `other = total – terms`.

I jeszcze jedno wyszukiwanie facetowe:

    :::bash
    curl -s 'localhost:9200/tweets/_search?pretty' -d '
    {
      "query": { "query_string": {"query": "redis"} },
      "sort": { "created_at": { "order": "desc" } },
      "facets": { "hashtags": { "terms":  { "field": "hashtags", size: 4 }, "global": true } }
    }'

A teraz zupełnie inny facet,
[date histogram facet](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-facets-date-histogram-facet.html):

    :::bash
    curl -s 'localhost:9200/tweets/_search?pretty' -d '
    {
      "query": { "match_all": {} },
      "sort": { "created_at": { "order": "desc" } },
      "facets": {
         "statuses_per_day": {
            "date_histogram":  { "field": "created_at", "interval": "30m" }
         }
      }
    }'

Oto wynik wyszukiwania z *date histogram*:

    :::json
    "facets" : {
      "statuses_per_day" : {
        "_type" : "date_histogram",
        "entries" : [
          { "time" : 1384497000000, "count" : 45 },
          { "time" : 1384498800000, "count" : 81 },
          { "time" : 1384500600000, "count" : 98 },
          { "time" : 1384502400000, "count" : 95 },
          { "time" : 1384504200000, "count" : 95 },
          ...
        ]
      }
    }

Co to są za liczby przy *time*:

    :::js
    new Date(1332201600000);                  // Tue, 20 Mar 2012 00:00:00 GMT
    new Date(1332288000000);                  // Wed, 21 Mar 2012 00:00:00 GMT
    (new Date(1332288000000)).getFullYear();  // 2012


# Rivers allows to index streams

Instalacja wtyczek *rivers* jest prosta:

    bin/plugin -install river-couchdb
      -> Installing river-couchdb...
      Trying ...
    bin/plugin -install river-wikipedia
      -> Installing river-wikipedia...
      Trying ...

Repozytoria z kodem wtyczek są na Githubie [tutaj](https://github.com/elasticsearch).

MongoDB River Plugin for ElasticSearch:

* [elasticsearch-river-mongodb](https://github.com/richardwilly98/elasticsearch-river-mongodb)


### Zadania

1\. Zainstalować wtyczkę *Wikipedia River*. Wyszukiwanie?

2\. Przeczytać [Creating a pluggable REST endpoint](http://www.elasticsearch.org/tutorials/2011/09/14/creating-pluggable-rest-endpoints.html).

5\. [Filename search with ElasticSearch](http://stackoverflow.com/questions/9421358/filename-search-with-elasticsearch).


# JSON dump indeksu tweets

…czyli zrzut indeksu tweets do pliku w formacie JSON:

* [scroll](http://www.elasticsearch.org/guide/reference/api/search/scroll.html)
* [scan](http://www.elasticsearch.org/guide/reference/api/search/search-type.html)

Ściąga:

* a search request can be scrolled by specifying the *scroll* parameter;
  `scroll=4m` indicates for how long (*co to oznacza? 4 minuty czy 4 milisekundy*)
  the nodes that participate in the search will maintain relevant resources
  in order to continue and support it
* the *scroll_id* should be used when scrolling
  (along with the scroll parameter, to stop the scroll from expiring);
  the *scroll_id* **changes for each scroll request**
  and only the most recent one should be used
* the “breaking” condition out of a scroll is when no hits has been returned;
  the *hits.total* will be maintained between scroll requests

Przykład pokazujący jak to działa:

    :::bash
    curl -s 'localhost:9200/tweets/_search?search_type=scan&scroll=10m&size=4&pretty'

Opcjonalnie możemy dopisać kryteria wyszukiwania. Na przykład,
wyszukujemy wszystko:

    :::json
    {
       "query": {
         "match_all": {}
       }
    }

albo cokolwiek:

    :::json
    {
       "query": {
          "query_string": {
             "query": "cokolwiek"
          }
       }
    }

Wtedy zmieniamy wywołanie *curl* na:

    :::bash
    curl -s 'localhost:9200/tweets/_search?search_type=scan&scroll=10m&size=4&pretty' -d '
    {
       "query": {
         "match_all": {}
       }
    }'

Wynik wykonania tego polecenia, to przykładowo:

    :::json
    {
      "_scroll_id": "c2NhbjsxOzE6Q29xZ01qdkJTZHVRdTA1Ow=",
      "took": 10,
      "timed_out": false,
      "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
      },
      "hits": {
        "total": 105,
        "max_score": 0.0,
        "hits": [ ]
      }
    }

Teraz wykonujemy tyle razy polecenie:

    :::bash
    curl -s 'localhost:9200/_search/scroll?scroll=10m&pretty' \
      -d 'przeklikujemy ostatnią wersję _scroll_id'

aż otrzymamy pustą tablicę *hits.hits*:

    :::json
    {
      "_scroll_id": "c2lZ1UTsxO3RvdGFsX2hpdHM6MTM5Ow=",
      "took": 128,
      "timed_out": false,
      "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
      },
      "hits": {
        "total": 2024,
        "max_score": 0.0,
        "hits": [ ]
        ...

Przykładowa implementacja tego algorytmu w NodeJS (v0.10.22)
+ moduł [Restler](https://github.com/danwrong/restler) (v2.0.1):

    :::js dump-tweets.js
    var rest = require('restler');

    var iterate = function(data) {  // funkcja rekurencyjna
      rest.get('http://localhost:9200/_search/scroll?scroll=10m', { data: data._scroll_id } )
        .on('success', function(data, response) {
          if (data.hits.hits.length != 0) {
            data.hits.hits.forEach(function(tweet) {
              console.log(JSON.stringify(tweet)); // wypisz JSONa w jednym wierszu
            });
            iterate(data);
          };
        });
    };

    rest.get('http://localhost:9200/tweets/_search?search_type=scan&scroll=10m&size=32')
      .on('success', function(data, response) {
        iterate(data);
    });

Skrypt ten uruchamiamy tak:

    :::bash
    node dump-tweets.js

**Uwaga 1:** Moduł ma „buga“. Przed uruchomieniem skryptu należy
załatać plik *restler.js*. Łata jest w katalogu
*pp/elasticsearch/dump* z repozytorium z notatkami do wykładu.

**Uwaga 2:** Korzystając z tego skryptu, możemy łatwo przenieść
dane z Elasticsearch do MongoDB:

    :::bash
    node dump-tweets.js | mongoimport --upsert -d test -c tweets --type json


# Krótka ściąga z obiektu Date

Inicjalizacja:

    :::javascript
    new Date();
    new Date(milliseconds);
    new Date(dateString);
    new Date(year, month, day, hours, minutes, seconds, milliseconds);
    // parsing
    ms = Date.parse('2011-01-31T12:00:00.016');
    new Date(ms); // Mon, 31 Jan 2011 12:00:00 GMT

Metody:

    :::js
    d = new Date(1332288000000);
    d.getTime();         // 1332288000000
    d.getFullYear();     // 2011
    d.getMonth();        //    0    (0-11)
    d.getDate();         //   31    zwraca dzień miesiąca!
    d.getHours();
    d.getMinutes();
    d.getSeconds();
    d.getMilliseconds();

Konwersja na napis:

    d.toString();        // 'Mon Jan 31 2011 01:00:00 GMT+0100 (CET)'
    d.toLocaleString();  // 'Wed Mar 21 2012 01:00:00 GMT+0100 (CET)'
    d.toGMTString();     // 'Wed, 21 Mar 2012 00:00:00 GMT'

Zobacz też [Epoch & Unix Timestamp Conversion Tools](http://www.epochconverter.com/).
