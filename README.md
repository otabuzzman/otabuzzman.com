# otabuzzman's blog
A personal website hub.

### Setup
- install Go from [golang.org](https://golang.org/doc/install)
- install hugo from [github.com](https://github.com/gohugoio/hugo/)

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
  # add content hub module
  hugo mod get github.com/otabuzzman/otabuzzman.hub
  
  # initial hugo setup
  cat <<-EOF >config.toml
  baseURL = 'https://otabuzzman.com/'
  languageCode = 'en-us'
  title = "otabuzzman's blog"
  # use beautiful ananke theme module
  theme = ["github.com/theNewDynamic/gohugo-theme-ananke"]
  
  # use content module from hub
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
