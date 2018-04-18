# Contributing

Just check the [posts](https://github.com/play-with-docker/play-with-kubernetes.github.io/tree/master/_posts) folder and submit your tutorials there. For more information about writing tutorials, check out our [instructions page](writing-tutorials.md).

# Running trainings site

Clone this repo and run `docker-compose up` from the root directory of the repo to run it locally.  Browse the site by visiting http://localhost:4000. Please note that the commandline may not work, since the site currently points to a site hosted on Docker infrastructure. If you want spin up your own back-end, please visit https://github.com/play-with-docker/play-with-docker for more information. There's a `k8s` branch you can use.

Alternately, clone the repo and run the following docker container: `docker run --rm --label=jekyll --volume=$(pwd):/srv/jekyll -it -p 4000:4000 jekyll/jekyll:3.2 jekyll serve`

Browse the site by visiting http://localhost:4000