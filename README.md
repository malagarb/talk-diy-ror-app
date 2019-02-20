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
docker image build --tag diy_image:1.0 -f Dockerfile1.dev .
docker image ls
#REPOSITORY     TAG    IMAGE ID            CREATED             SIZE
#diy_image    1.0   e9211d05e5ef        1 minute ago       1.03GB
```

now launch the server

```bash
docker container run -it -p 3000:3000 --name diy_container -v `pwd`/src:/app diy_image:1.0
```

connect terminal to

```bash
docker container exec -it $container_id /bin/bash
```
docker container restart diy_container

There an problem with version of sqlite3 rails 5.2.2 should use sqlite3 one specific version:
gem 'sqlite3', '~> 1.3.6'


visit localhost:3000 it should works



## STEP 3 develop into rails as root user

```bash
docker container -it -p 3000:3000 --name diy_container -v `pwd`/src:/app diy_image:1.0
docker container exec -it diy_container /bin/bash
# into container
bin/rails generate model author name:string surname:string
```

go to editor PROBLEM

```bash
id
sudo chown 1000:1000 -R .
```
every container create a new file redo chown

## STEP 4 develop into as non-root user

first try into container

```bash
# out container
id
docker container exec -it diy_container
# into container
groupadd -g 1000 dockerusers
useradd -u 1000 -g 1000 dockeruser
useradd -u 1000 -g 1000 -m dockeruser
su - dockeruser # problem because change the context
su dockeruser

# problem root user and new user don't share the gems
echo $PATH
irb
puts $LOAD_PATH
bundle install --system
```


### must we rewrite our Dockerfile




```dockerfile
FROM ruby:2.5.3
RUN apt-get update -yqq
RUN apt-get install -yqq --no-install-recommends nodejs

RUN groupadd -g 1000 dockeruser
RUN useradd -u 1000 -g 1000 dockeruser

# ENV RAILS_ENV development
# ENV RAILS_ENV development

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
docker image build --tag diy_image:1.0 -f Dockerfile2.dev .
docker container run -it -p 3000:3000 -v `pwd`/src:/app --name diy_container diy_image:1.0
docker container exec -it diy_container /bin/bash
```

back to develop again

```bash
bin/rails db:migrate
```

now we want add simple validations to model and some fixtures

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

stop and start again the container

```bash
docker container stop diy_container
docker container start diy_container
```


let's see routes.rb directly with postman

```bash
bin/rails routes
```



this image that we are using is awesome bigger





## STEP 5 - but one doesn't simply runs one container, docker-compose.yml
