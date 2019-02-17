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
