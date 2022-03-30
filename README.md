# otabuzzman's blog
A personal website hub.

### Setup
- install Go from [golang.org](https://golang.org/doc/install)
- install hugo (extended) from [github.com](https://github.com/gohugoio/hugo/)

  ```
  # get hugo repo
  cd /usr/lab ; git clone https://github.com/gohugoio/hugo.git ; cd hugo
  
  # install hugo (in ~/go/bin)
  go install
  
  # remove hugo repo (optional)
  cd .. ; rm -rf hugo
  ```

- init otabuzzman.com

  ```
  # init otabuzzman.com
  hugo init site otabuzzman.com
  
  # use hugo modules
  hugo mod init dummy
  hugo mod get github.com/otabuzzman/otabuzzman.hub
  
  # use beautiful ananke theme
  echo 'theme = ["github.com/theNewDynamic/gohugo-theme-ananke"]' >> config.toml
  
  # use module from hub
  cat <<-EOF >> config.toml
  
  [module]
    [[module.imports]]
      path = 'github.com/otabuzzman/otabuzzman.hub'
      disabled = false
      [[module.imports.mounts]]
        source = 'modules/content'
        target = 'content'
      [[module.imports.mounts]]
        source = 'modules/static'
        target = 'static'
  EOF
  
  # run hugo server
  hugo server -D
  
  # open http://localhost:1313
  ```
