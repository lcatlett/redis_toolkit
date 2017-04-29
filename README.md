# Redis Monitor
Actively monitors a redis database using the `redis-cli monitor` command (https://redis.io/commands/monitor).

## Installation & Usage
A valid docker install is required. 

Modify docker-compose.yml with the target REDIS_CLI string, then build and run. The output should be displayed on screen. You are now injecting every command GET/SET command that Redis is running into your local instance. To generate the report see below.

```
$ docker-compose build
$ docker-compose up
```

## Implementation details
The output shows the key hitrate, calculated using the following formula `hitrate = (gets / (gets + sets)) * 100`, the number of GET and SET operations. The result is ordered by hitrate asscending and is refreshed every 5 seconds for an (almost) instantaneous feedback about what is going on the live Redis server.

## Running it in production
MONITOR is a debugging command that streams back every command processed by the Redis server. Running this on a production database comes with a performance cost that's hard to estimate. Use it with caution on production servers.

## Generate the report
There are 2 types of reports that can be generated: a prefix-only and a full report for all your keys. The default is the full report. You can get the DOCKER_NETWORK by running `docker network ls` but it's usualy `redismonitor_default`.

### Prefix only report

```
$ docker build -t dicix/redis_report .
$ docker run --rm -it --name redis_report --network=DOCKER_NETWORK dicix/redis_report python report.py --prefix_only
Key                                                                                        Count      GET        SET        Hit Rate (%)
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_commerce_shipping_rates:258*                                          436        0          0          0          
pantheon-redis:cache_rules:comp_rules_*                                                    10         0          20         0          
pantheon-redis:cache_commerce_shipping_rates:*                                             4          0          0          0          
pantheon-redis:cache_path:shop/*                                                           6          6          6          50         
pantheon-redis:cache_path:humans*                                                          2          90         1          98         
pantheon-redis:cache_metatag:output:global:en:200:https:dbrand.com:/:shop/galaxy...*       2          2          0          100             
pantheon-redis:cache_menu:links:main-menu:tree-data:en:*                                   24         10688      0          100   
```
### Full report

```
$ docker build -t dicix/redis_report .
$ docker run --rm -it --name redis_report --network=DOCKER_NETWORK dicix/redis_report python report.py
Key                                                                                        Count      GET        SET        Hit Rate (%)
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_commerce_shipping_rates:2585422:*                                     1          0          0          0         
pantheon-redis:cache_commerce_shipping_rates:2585490:*                                     1          0          0          0         
pantheon-redis:cache_commerce_shipping_rates:2585422:*                                     1          0          0          0         
pantheon-redis:cache_commerce_shipping_rates:2585457:*                                     1          0          0          0         
pantheon-redis:cache_commerce_shipping_rates:2585422:*                                     1          0          0          0         
pantheon-redis:cache_commerce_shipping_rates:2585405:*                                     1          0          0          0         
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_commerce_shipping_rates:25854*                                        6          0          0          0 

Key                                                                                        Count      GET        SET        Hit Rate (%)
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_path:shop/iphone-7-skins\xe2\x80\xa6                                  1          1          1          50        
pantheon-redis:cache_path:shop/htc-10-skinsHTC\xe2\x80\x99s                                1          1          1          50        
pantheon-redis:cache_path:shop/iphone-7-skins\xe2\x80\xa6                                  1          1          1          50        
pantheon-redis:cache_path:shop/google-pixel-skins/n/nVideo                                 1          1          1          50        
pantheon-redis:cache_path:shop/iphone-7-skins\xe2\x80\xa6                                  1          1          1          50        
pantheon-redis:cache_path:shop/google-pixel-\xe2\x80\xa6                                   1          1          1          50        
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_path:shop/*                                                           6          6          6          50         

Key                                                                                        Count      GET        SET        Hit Rate (%)
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_path:checkout/2611568/complete                                        1          1          1          50        
pantheon-redis:cache_path:checkout/2611563/payment_method                                  1          1          1          50        
----------------------------------------------------------------------------------------------------------------------------------
pantheon-redis:cache_path:checkout/261156*                                                 2          2          2          50 
```

## Options
There are a couple of options that can be passed to the report generator either via docker's -e paramenter (environment) like so: `-e "OPTIONS=--prefix_only"` or directly. They are described below. The LEVENSHTEIN_DISTANCE parameter is used for calculating the degree of similarity between these groups using the Levenshtein distance algorithm. You can set any value between 0 and 1:

* values close to 0 will try to create many groups with very little differences between them 
* values close to 1 will try to create bigger buckets with many differences between strings but a smaller common prefix

```
usage: report.py [-h] [--prefix_only] [-l LEVENSHTEIN_DISTANCE]
                 [--hide_progress_bar]

Generates a hit rate report from the Redis keys

optional arguments:
  -h, --help            show this help message and exit
  --prefix_only         Only show the groups of keys.
  -l LEVENSHTEIN_DISTANCE, --levenshtein_distance LEVENSHTEIN_DISTANCE
                        Manually calibrate the Levenshtein distance in percentage of string length. Default is 0.5
                            - values close to 0 will try to create many groups with very little differences between them.
                            - values close to 1 will try to create bigger buckets with many differences between strings but a smaller common prefix.
  --hide_progress_bar   Hides the progress bar in case you want to redirect the output to a file.
```
