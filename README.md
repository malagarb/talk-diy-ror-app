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

Model | Author   | Books         | Comments
      | id       | id            | id
      | name     | title         | book_id
      | surname  | author_id     | text
      |          | language      |
      |          | published_at  |


## STEP 1 - creation rails project within a container



```bash
sudo service docker start
mkdir -p diy/src
cd diy
# ensure to run commands into right folder
docker image pull ruby:2.5.3
docker container run -it --rm --name=ror_tmp -v `pwd`/src:/app ruby:2.5.3 /bin/bash
# into the container
ruby --version
gem install rails -v 5.2.2
cd /app
whoami
rails new .
exit
```


```bash
# out the container
docker container rm ror-container-generator
cd  ~/repo
ls -las
id  # uid=1000(stbn) gid=1000(stbn)
sudo chown 1000:1000 -R ./src
```

OPEN ATOM INTO FOLDER


MODIFICATION Gemfile
gem 'sqlite3'
gem 'sqlite3', '~> 1.3.6'



## STEP 2 - Dockerfile creation, docker image build and docker container run

constraint should use the same versions ruby and rails

```Dockerfile
# dockerfile.1.dev
FROM ruby:2.5.3
RUN apt-get update -yqq
RUN apt-get install -yqq --no-install-recommends nodejs

# dev: Don't copy full src, it will be mount as a volume
COPY src/Gemfile src/Gemfile.lock /app/

# explotation: Copy full src code won't change
# COPY src /app

WORKDIR /app
RUN bundle install
CMD ["bin/rails", "s", "-b", "0.0.0.0"]
```

let's build the image

```bash
cd diy
docker image build --tag diy_image:1.0 -f Dockerfile.1.dev .
docker image ls
#REPOSITORY     TAG    IMAGE ID            CREATED             SIZE
#diy_image    1.0   e9211d05e5ef        1 minute ago       1.03GB
```

now launch the server

```bash
docker container run -it -p 3000:3000 -v `pwd`/src:/app --name diy_container_1 diy_image:1.0
docker container ps -a
ss -tunpl
```

visit localhost:3000 it should works


## STEP 3 develop into rails as root user

THINK ... the container still works as root 


```bash
docker container exec -it $container_id /bin/bash

bin/rails generate model author name:string surname:string
```

ATOM edit and file diy/app/model/author.rb

FAILS: ANY GENERATOR, CREATION OF FILE, WILL BE CREATED BY ROOT NON EDITABLE ERGO YOU CAN NOT OVERWRITE

FIX SOLUTION QUICK & DIRTY

```bash
id
sudo chown 1000:1000 -R ./src
```

WE DON'T WANT REPEAT THIS FOR EVER, SO LET'S FIX


## STEP 4 develop into as non-root user

first when execute as root it create some files like logs, we need ownership again over these

let's fix our Dockerfile


```dockerfile
FROM ruby:2.5.3
RUN apt-get update -yqq
RUN apt-get install -yqq --no-install-recommends nodejs

RUN groupadd -g 1000 dockeruser
RUN useradd -u 1000 -g 1000 dockeruser

# dev: Don't copy full src, it will be mount as a volume
COPY src/Gemfile src/Gemfile.lock /app/

# explotation: Copy full src code won't change
# COPY src /app

WORKDIR /app
RUN bundle install --system
RUN chown 1000:1000 -R /app
USER dockeruser
CMD ["bin/rails", "s", "-b", "0.0.0.0"]
```


```bash
docker image build --tag diy_image:1.1 -f Dockerfile.2.dev .

docker container run -it -p 3000:3000 -v `pwd`/src:/app --name diy_container_2 diy_image:1.1
```
visit localhost:3000

back to develop again

```bash
docker container exec -it diy_container_2 /bin/bash
bin/rails db:migrate
```

check if we can modify now we want add simple validations to model and some fixtures

... new gem => new build ... usually I use Faker but now we use fixtures

at src/app/models/author
```ruby
class Author < ApplicationRecord
  validates :name, presence: true
  validates :surname, presence: true
end
```


at src/db/seed.rb
```ruby
sandi_metz = Author.create([name: 'Sandi', surname: 'Metz'])
katrina_owen = Author.create([name: 'Katrina', surname: 'Owen'])
sam_ruby = Author.create([name: 'Sam', surname: 'Ruby'])
rob_isenberg = Author.create([name: 'Rob', surname: 'Isenberg'])
```

generate controller for api into submodule
```bash
bin/rails db:seed
bin/rails generate controller api/v1/authors index show
```

at: src/app/controller
```ruby
class Api::V1::AuthorsController < ApplicationController
  def index
    authors = Author.all
    render json: {data: authors}
  end

  def show
    author = Author.find(params[:id])
    render json: {data: author}
  end
end
```
at: src/config/routes.rb
```ruby
namespace 'api' do
  namespace 'v1' do
    resources :authors
  end
end
```

VISIT localhost:3000/api/v1/authors
VISIT localhost:3000/api/v1/authors



let's see routes.rb directly with postman

```bash
bin/rails routes
```


stop and start again the container

```bash
docker container stop diy_container
docker container start diy_container
```



ONE DOESN'T SIMPLY RUNS ONE CONTAINER... 




## STEP 5 - but one doesn't simply runs one container, docker-compose.yml
