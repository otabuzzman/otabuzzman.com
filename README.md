# otabuzzman's blog
A personal website hub.

### Manual setup
- install Go from [golang.org](https://golang.org/doc/install)
- install hugo from [github.com](https://github.com/gohugoio/hugo/)

  ```
  # get Hugo repo
  cd /usr/lab ; git clone https://github.com/gohugoio/hugo.git ; cd hugo
  
  # install Hugo (in ~/go/bin)
  go install
  
  # remove Hugo repo (optional)
  cd .. ; rm -rf hugo
  ```

- init otabuzzman.com

  ```
  # create otabuzzman.com
  hugo new site otabuzzman.com
  
  # use Hugo modules
  hugo mod init dummy
  # add content hub module
  hugo mod get github.com/otabuzzman/otabuzzman.hub
  
  # initial Hugo setup
  cat <<-EOF >config.toml
  baseURL = 'https://otabuzzman.com/'
  languageCode = 'en-us'
  title = "otabuzzman's blog"
  # use Ananke theme module
  theme = ["github.com/theNewDynamic/gohugo-theme-ananke"]
  
  # use content module from hub
  [module.imports]
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
