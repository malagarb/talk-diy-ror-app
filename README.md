# source code for talk/workshop about DIY ROR App (dockerize it yourself ruby on rails apps)

The idea of app is:

consider we are implementing a system for an online Bookstore. The App must to have
the endpoints for your REST API along with the HTTP verbs in order for the system to satisfy
the following requirements

* Retrieve the complete list of the books for the Bookstore
* Retrieve a single book for the bookstore
* Retrieve the set of user comments about a single book from the bookstore
* submit a new comment into a single book from the Bookstore
* Retrieve the author information for a single book from the Bookstore

the attributes for the models will be

Model | Author   | Books    | Comments
      | id       | id       | id
      | name     | title    | book_id
      | surname  | author_id | text
      |         | language   |
      |         | published_at  |


## STEP 1 - creation rails project within a container



```bash
# ensure to run commands into right folder
docker image pull ruby:2.5.3
docker container run -it --name=ror-container-generator -v ${PWD}:/usr/src/app ruby:2.5.3 /bin/bash
# into the container
gem install rails
cd /usr/src/app/src
rails new .
exit
# out the container
docker container rm ror-container-generator
cd  ~/repo
id  # uid=1000(stbn) gid=1000(stbn)
sudo chown 1000:1000 -R ./src
```




## STEP 2 - Dockerfile creation, docker image build and docker container run

constraint should use the same versions ruby and rails

```Dockerfile
FROM ruby:2.5.3
RUN apt-get update -yqq
RUN apt-get install -yqq --no-install-recommends nodejs

# dev: Don't copy full src, it will be mount as a volume
COPY src/Gemfile src/Gemfile.lock /app

# explotation: Copy full src code won't change
# COPY src /app

WORKDIR /app
RUN bundle install
CMD ["bin/rails", "s", "-b", "0.0.0.0"]
```

let's build the image

```bash
cd ~/repo
docker image build --tag diy_image:1.0 -f Dockerfile.dev .
docker image ls
#REPOSITORY     TAG    IMAGE ID            CREATED             SIZE
#diy_image    1.0   e9211d05e5ef        1 minute ago       1.03GB
```

now launch de server

```bash
docker container run -it -p 3000:3000 --name diy_container -v `pwd`/src:/app diy_image:1.0
```

connect terminal to

```bash
docker container exec -it $container_id /bin/bash
```
docker container restart diy_container

There an problem with version of sqlite3 rails 5.2.2 shoul use sqlite3" , '~> 1.3.6'
