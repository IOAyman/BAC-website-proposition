# BAC Website Proposition

We at [Pinnovate](https://pinnovate.io/) are trying to re-imagine how Algerian educational services can be enhanced for the greater good. One of the most popular online services is the results website of the BAC (baccalauréat, the secondary-school (lycée) diploma), BEM (Medium school diploma), and the primary school diploma.

Usually theses websites (especially the BAC one) get a big amount of traffics and visits whenever there is an announcement of some national exam results, hence, their respective websites turn out unresponsive, their servers get down. The underlying infrastructure seems to be vertical (inscrease server hardware each year), not scalable. We'll focus here on how to increase the software performance as well. In a scalable way.

So we decided to make a short experiment (proposal?), of how these websites could resist more to **higher** traffics with modest hardware, so far, this required changing the backend, away from the classic php-mysql-apache, to rather some relatively new stack, if fact, we -almost- completely removed the backend!

## Highlights:

* No database, all is in-memory.
* No app (whether php, or something else)
* No disk I/O, so it's extremely fast
* Restful.
* Scalable solution
* Has a simple Android app for the 3G users out there.
* Import results from Excel files.
* Simple to deploy.

It's just **nginx** + **redis**! no other dependency.

![Comparision](http://res.cloudinary.com/walid/image/upload/v1434533651/btb_rt9yhl.png)

## Facts, and why Redis?

### First, let's do our math
According to [Echourouk journal](http://www.echoroukonline.com/ara/articles/221657.html) there's `1 710 746` BAC candidates, serializing the result to json would be something like this:

```json
[{
    "matricule": "<matricul>",
    "nom": "<nom>",
    "prenom": "<prenom>",
    "filiere": "<filiere>",
    "lieu_de_naissance": "<lieu_de_naissance>",
    "date_de_naissance": "<date_de_naissance>",
    "moyenne": "<moyenne>",
 }]
```
two other values are omitted because they can be deduced from `moyenne`. here's the same json encoded.

```json
[{"nom": "\u064a\u0639\u062d\u0643\u0646\u0630\u0635\u062a\u0642\u0639", "prenom": "\u0642\u0643\u0648\u0630\u064a\u0642\u063a\u0623\u0621\u0634", "date_de_naissance": "\u00efR\u00c0\u00c0\u00c8sWHm\u00efyqI", "moyenne": 11, "filiere": "\u0639\u0638\u0632\u0646\u062f\u0641\u0644\u0626\u0625\u0637\u062b\u062c\u0643\u0633\u0637\u0632\u0621\u0643\u0643\u062a", "lieu_de_naissance": "\u0638\u0643\u0642\u0634\u0623\u0647\u0638\u062d\u0644\u0629\u0625\u062c\u062a\u0631\u0630", "matricule": "01499990"}]
```
So a worst case scenario for given result json size is around ~500 bytes per result.

For all candidates would be: 
```
1 710 746 * 500 = 855373000 Bytes = 855373 KB = ~856 MB
```

~856 MB can be entirely put in RAM!

### Redis
[Redis](http://redis.io/) is key-value store, it basically load its data in memory, so it serves as a cache server. if enabled it also buffers out its content to `.rdb` files every once in a while for persistence, these files are loaded on the next server startup.

Given these key features, redis can serve well our use case. Loading all BAC results as encoded json to redis (in memory) then access it directly from nginx. each `matricule` is a redis key, the rest is the value.

With a server like nginx, we can route all `/get?key=<matricule>` and proxy pass them to redis, get the result back as json. et le tour est joué :)

To make nginx talk to redis, we need to custom compile it with Redis plugin enabled. Don't be scared, this can be done easily, and is a matter of:

```bash
# assuming we'll compile nginx to /opt
./configure --prefix=/opt/nginx \
            --add-module="$current_dir/ngx_http_redis-0.3.7" \
		    --add-module="$current_dir/set-misc-nginx-module-0.28"
make -j2
make install
```

Then, an example nginx config would be something like this:

```nginx
...

upstream redisbackend {
    server 127.0.0.1:6379;
    keepalive 2024;
}

server {
    listen       80;
    server_name  localhost;
    access_log  off; # disable access log

    # access_log  logs/host.access.log  main

    location = /get {
            set_unescape_uri $redis_key $arg_key;  # this requires ngx_set_misc
            access_log  off;
            expires 1M;
            add_header Cache-Control "public";
            redis_pass     redisbackend;
            default_type   text/html;
            error_page     404 /404.html;
    }
    
...
}
```


## Benchmarking
If all candidates would access the website at the same time, and by the same time we mean the same minute! which is not likely to happen, but let's expect the worse, this would mean:
`1 710 746 requests per minutes = ~28513 req/sec`
so for an acceptable solution, we need to achieve at least around ~28513 req/sec server performance.

Using a tool like [wrk](https://github.com/wg/wrk) we measured how our solution is performing:
```
wrk -t12 -c400 -d30s http://localhost/get?key=<some_matricule>

  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.52ms    1.35ms 205.65ms   86.66%
    Req/Sec     9.47k     1.32k   22.90k    73.86%
  3393547 requests in 30.03s, 2.34GB read
Requests/sec: 113015.19
Transfer/sec:     79.97MB
```

This runs a benchmark for 30 seconds, using 12 threads, and keeping 400 HTTP connections open. With this we got ~**113015 req / sec** this outperform out initial requirement 28513 req/sec!

With more nginx fine-tuning, we got even more than that, on an i7 machine (doesn't really matter as the CPU here is not fully utilized, 8 GB ram with less than 1.5GB used at most), this benchmark used just 4 nginx worker threads.

As a bonus, you can monitor the disk io using `iotop`, after disabling nginx logs, there are no disk reads or writes.

### comparing to php-apache-mysql

With the **same machine**, we compared a simple php script that fetchs just the `matricule` and `nom` from mysql database, returns just one line of html, and we got:
```
  4 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    79.77ms  197.57ms   1.97s    90.87%
    Req/Sec     3.50k     1.37k   10.92k    71.56%
  413244 requests in 30.01s, 86.45MB read
  Socket errors: connect 0, read 15, write 0, timeout 221
Requests/sec:  13768.19
Transfer/sec:      2.88MB
```
Thats around ~14k req/sec, even with OPcache enabled as recommanded by [the PHP Docs](http://php.net/manual/en/opcache.installation.php). That is even less than our scenario requirements (28k req/sec) :) 

## Reducing http round-trips
Past websites used this scenario:

* a user visits `bac.onec.dz` and `GET` the file `/index.php`.
* the user introduces their `matricule` with a `POST` request.
* the server responds back with `index.php` within it, a javascript code that shows the result in a `alert()` popup.
* if no luck, the user refreshes the page until it gets something.

this scenario with that bad user experience can spawn more http-rountrips, with unnecessary `csrf` token that needs to be verified even though there's no login or password thing. all data are publicly available.

Enhancing this would be:
* a user gets `bac.onec.dz`. `index.html` is served fast out of memory.
* `index.html` is compressed, with minimum css, js embedded or `CDN`ed.
* the user introduces their `matricule`, a javascript code makes a `GET` request without refreshing the page.
* the result is printed out in a nicely styled yet simple `<div>`.
* if no luck, a refresh button/icon is presented beside the simple form, with a simple text like `Do not refresh the page, click this button instead`, this user experience trick could save us some http-round-trips and reduce the server load.
* Publish a mobile app to reduce `GET`ing the `index.html` from the first place, one request less (See below for an example Android app).

## Going horizontal

Using nginx, we can (optionally) make a simple [load balancing](http://nginx.org/en/docs/http/load_balancing.html) across multiple serves, if one gets down, we route the requests to the other live server. We can even use the nginx [geoip module](http://nginx.org/en/docs/http/ngx_http_geoip_module.html) to route requests according to the user location (Algerian Est, West, Algiers, ..etc).


## The Android app

A mobile app itself is huge load reducing solution, all requests in the mobile app can be routed to something like: `android.bac.onec.dz` which is another mirror server. if the main server get down (we still expect the worse :) ), Android users have a chance to still get their results.

![Comparision](http://res.cloudinary.com/walid/image/upload/c_scale,w_295/v1434559943/Screenshot_2015-06-17-17-39-53_cp65zh.png)

## Installation and usage
First install redis, on Ubuntu, this simply is a matter of:

`$ apt-get install redis-server`

For nginx, we need [HttpRedis](http://wiki.nginx.org/HttpRedis) module, along with [set-misc-nginx-module](https://github.com/openresty/set-misc-nginx-module), the whole installation instruction can be found in the `install.sh` file.

There's also a simple Python script for generating and Excel file with 1 750 000 random entries and load redis with these entries.

## Disclaimer

We're not going to publish any live demo, or any Android app code (We would like to, but sorry). We do not want any kind of spam, maluse, or mislead people by some search engine keywords when they search Google.

This is much of a software architecture proposal than a live demo. source code doesn't really matter in this case (as it just a simple nginx config + a python script for loading Excel files encoded as json to Redis).
