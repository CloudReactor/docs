# docs

## To start local

Run Docker

Then:

  ```
  docker run --rm --volume="$PWD:/srv/jekyll" --volume="$PWD/vendor/bundle:/usr/local/bundle" --env JEKYLL_ENV=development -p 4000:4000 jekyll/jekyll:3.8 jekyll serve --config _config_local.yml
  ```

and browse to `http://0.0.0.0:4000`
