# Steam Scraper


## Installation
다운로드 후
cmd 연 다음에 
```
cd steam-scraper-master
```

그리고 파이썬 가상환경을 열자
```
virtualenv -p python 3.7
```

requirements를 설치
```
pip install -r requirements.txt
```
>> 텍스트 파일안에 들어있는 프로그램을 다운받으라는 표시로 -r flag를 붙여줌 <br>
>> 경로지정을 해야 하므로 -r 뒤에 drag and drop<br>

#### 맥에서는 ...
By the way, on macOS you can install Python 3.6 via [homebrew](https://brew.sh):
 ```bash
 brew install python3
```

## Crawling the Products

Steam product listing 에 나열되어 있는 product pages들을 긁어오고 이에 대한 metadata를 정리하기 위해서 쓰는 spider.<br>
시간이 꽤 걸린다.<br>
cmd 창을 연 후
```bash
scrapy crawl products -o output/products_all.jl --logfile=output/products_all.log --loglevel=INFO -s JOBDIR=output/products_all_job -s HTTPCACHE_ENABLED=False
```

참고로,
>> scrapy crawl command는 cfg 파일이 있는 곳에서 가능하다. scrapy.cfg 파일이 들어있는 steam-scraper-master 파일에서 실행하자.<br>
>> scrapy crawl 하면 쓸 수 있는 다양한 scrapy command들을 볼 수 있다.<br>
>> -o 에서는 output파일을 'output' 이라고 지정해두었으므로 디렉토리에 output이라는 파일을 만들어놓자.<br>
>> 실행하기전에 ProductSpider 파일을 손보자.<br>

완료된 후에는 products_all.jl 이라는 파일이 아웃풋으로 나온다.<br>

Here's some example output:
```python
{
  'app_name': 'Cold Fear™',
  'developer': 'Darkworks',
  'early_access': False,
  'genres': ['Action'],
  'id': '15270',
  'metascore': 66,
  'n_reviews': 172,
  'price': 9.99,
  'publisher': 'Ubisoft',
  'release_date': '2005-03-28',
  'reviews_url': 'http://steamcommunity.com/app/15270/reviews/?browsefilter=mostrecent&p=1',
  'sentiment': 'Very Positive',
  'specs': ['Single-player'],
  'tags': ['Horror', 'Action', 'Survival Horror', 'Zombies', 'Third Person', 'Third-Person Shooter'],
  'title': 'Cold Fear™',
  'url': 'http://store.steampowered.com/app/15270/Cold_Fear/'
 }
```

## Extracting the Reviews

The purpose of `ReviewSpider` is to scrape all user-submitted reviews of a particular product from the [Steam community portal](http://steamcommunity.com/). 

url이 든 텍스트 파일 (ex. edu_urls.txt) 생성 후 경로 지정 :

```bash
scrapy crawl reviews -o edureviews.jl -a url_file=links.txt -s JOBDIR=output/reviews
```

An output sample:
```python
{
  'date': '2017-06-04',
  'early_access': False,
  'found_funny': 5,
  'found_helpful': 0,
  'found_unhelpful': 1,
  'hours': 9.8,
  'page': 3,
  'page_order': 7,
  'product_id': '414700',
  'products': 179,
  'recommended': True,
  'text': '3 spooky 5 me',
  'user_id': '76561198116659822',
  'username': 'Fowler'
}
```

If you want to get all the reviews for all products, `split_review_urls.py` will remove duplicate entries from `products_all.jl` and shuffle `review_url`s into several text files. -> products spider로 더이상 # of reviews가 긁히지 않음 (수정하기)
This provides a convenient way to split up your crawl into manageable pieces.
The whole job takes a few days with Steam's generous rate limits.

## Deploying to a Remote Server

This section briefly explains how to run the crawl on one or more t1.micro AWS instances.

First, create an Ubuntu 16.04 t1.micro instance and name it `scrapy-runner-01` in your `~/.ssh/config` file:
```
Host scrapy-runner-01
     User ubuntu
     HostName <server's IP>
     IdentityFile ~/.ssh/id_rsa
```
A hostname of this form is expected by the `scrapydee.sh` helper script included in this repository.
Make sure you can connect with `ssh scrappy-runner-01`.

### Remote Server Setup

The tool that will actually run the crawl is [scrapyd](http://scrapyd.readthedocs.io/en/stable/) running on the remote server.
To set things up first install Python 3.6:
```bash
sudo add-apt-repository ppa:jonathonf/python-3.6
sudo apt update
sudo apt install python3.6 python3.6-dev virtualenv python-pip
```
Then, install scrapyd and the remaining requirements in a dedicated `run` directory on the remote server: 
```bash
mkdir run && cd run
virtualenv -p python3.6 env
. env/bin/activate
pip install scrapy scrapyd botocore smart_getenv  
```
You can run `scrapyd` from the virtual environment with
```bash
scrapyd --logfile /home/ubuntu/run/scrapyd.log &
```
You may wish to use something like [screen](https://www.gnu.org/software/screen/) to keep the process alive if you disconnect from the server.

### Controlling the Job

You can issue commands to the scrapyd process running on the remote machine using a simple [HTTP JSON API](http://scrapyd.readthedocs.io/en/stable/index.html).
First, create an egg for this project:
```bash
python setup.py bdist_egg
```
Copy the egg and your review url file to `scrapy-runner-01` via
```bash
scp output/review_urls_01.txt scrapy-runner-01:/home/ubuntu/run/
scp dist/steam_scraper-1.0-py3.6.egg scrapy-runner-01:/home/ubuntu/run
```
and add it to scrapyd's job directory via 
```bash
ssh -f scrapy-runner-01 'cd /home/ubuntu/run && curl http://localhost:6800/addversion.json -F project=steam -F egg=@steam_scraper-1.0-py3.6.egg'
```
Opening port 6800 to TCP traffic coming from your home IP would allow you to issue this command without going through SSH.
If this command doesn't work, you may need to edit `scrapyd.conf` to contain
```
bind_address = 0.0.0.0
```
in the `[scrapyd]` section.
This is a good time to mention that there exists a [scrapyd-client](https://github.com/scrapy/scrapyd-client) project for deploying eggs to scrapyd equipped servers.
I chose not to use it because it doesn't know about servers already set up in `~/.ssh/config` and so requires repetitive configuration.

Finally, start the job with something like
```bash
ssh scrapy-runner-01 'curl http://localhost:6800/schedule.json -d project=steam -d spider=reviews -d url_file="/home/ubuntu/run/review_urls_01.txt" -d jobid=part_01 -d setting=FEED_URI="s3://'$STEAM_S3_BUCKET'/%(name)s/part_01/%(time)s.jl" -d setting=AWS_ACCESS_KEY_ID='$AWS_ACCESS_KEY_ID' -d setting=AWS_SECRET_ACCESS_KEY='$AWS_SECRET_ACCESS_KEY' -d setting=LOG_LEVEL=INFO'
```
This command assumes you have set up an S3 bucket and the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables.
It should be pretty easy to customize it for non-S3 output, however.

The `scrapydee.sh` helper script included in the `scripts` directory of this repository has some shortcuts for issuing commands to scrapyd-equipped servers with hostnames of the form `scrapy-runner-01`.
For example, the command
```bash
./scripts/scrapydee.sh status 1
# Executing status()...
# On server(s): 1.
```
will run the `status()` function defined in `scrapydee.sh` on `scrapy-runner-01`.
See that file for more command examples.
You can also run each of the included commands on multiple servers:
First, change the `all()` function within `scrapydee.sh` to match the number of servers you have configured.
Then, issue a command such as
```bash
./scripts/scrapydee.sh status all
```
The output is a bit messy, but it's a quick and easy way to run this job.


## Julia Tutorial


### Adding Julia to Jupyter Notebook

 ```bash
using Pkg
Pkg.add("Julia")
```

### Handling files with Julia
> Open file
```
f = open("products_all.jl")
```

> Or
```
open("products_all.jl") do file
    # do stuff with the open file
end
```
- So at the end, <br>
- Assuming that JSON package is installed (julia> Pkg.add("JSON")<br>
- We'll convert the Julia file to JSON format
```
import JSON
f = open("products_all.jl")
open("products_all.json", "w") do file
    write(file, f)
end
```

