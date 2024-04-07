# otabuzzman's blog
A personal website.

### Local setup
- Install Go from [golang.org](https://golang.org/doc/install)
- Install hugo from [github.com](https://github.com/gohugoio/hugo/releases/tag/v0.111.3)

- Init otabuzzman.com

  ```
  # create otabuzzman.com
  WEBSITE=otabuzzman.com
  hugo new site $WEBSITE
  cd $WEBSITE
  
  # use Hugo with modules
  hugo mod init $WEBSITE
  # initiate hugo modules
  hugo mod get github.com/otabuzzman/$WEBSITE
  
  PUBLISHDIR=/opt/otabuzzman/www
  
  # create initial config.toml
  cat >config.toml <<-EOF
  baseURL = 'https://$WEBSITE/'
  languageCode = 'en-us'
  title = "otabuzzman's blog"
  
  # use the beautiful Ananke theme
  theme = ["github.com/theNewDynamic/gohugo-theme-ananke"]
  
  # publish to Apache web server
  publishDir = "$PUBLISHDIR"
  
  # use content from repository
  [module.imports]
    path = 'github.com/otabuzzman/$WEBSITE'
    disabled = false
    [[module.imports.mounts]]
      source = 'content'
      target = 'content'
    [[module.imports.mounts]]
      source = 'static'
      target = 'static'
  
  [[params.ananke_socials]]
    name = "github"
    url = "https://github.com/otabuzzman"
  [[params.ananke_socials]]
    name = "twitter"
    url = "https://twitter.com/otabuzzman"
  [[params.ananke_socials]]
    name = "linkedin"
    url = "https://www.linkedin.com/in/juergenschuck/"
  EOF
  ```

- Run local hugo server

  ```
  # run hugo server
  hugo server -D
  ```

- Open [http://localhost:1313](http://localhost:1313)

- Hugo site generator [documentation](https://gohugo.io/documentation/) ([quick start](https://gohugo.io/getting-started/quick-start/))
- Ananke theme for Hugo [documentation](https://github.com/theNewDynamic/gohugo-theme-ananke)

### VPS setup
Setup on Virtual Private Server (VPS) running Ubuntu 18.04. Provider is [contabo.de](https://contabo.de/). VPS access details received by mail. Domain registrar for [otabuzzman.com](https://www.whois.com/whois/otabuzzman.com) is [ionos.de](https://www.ionos.de/). Setup assumes a control node with SSH access to VPS.

* [A. Basic setup](#A-Basic-setup) - Essentially SSH setup and firewall.<br>
* [B. Web server setup](#B-Web-server-setup) - Apache web server with Let's Encrypt certificate.
* [C. Blog setup](#C-Blog-setup) - Hugo site building framework.

### A. Basic setup

1. Follow instructions in [deploy.md](https://github.com/otabuzzman/chartacaeli-web/blob/master/deploy.md#A-Basic-setup) section A (apply changes accordingly).

### B. Web server setup

1. Follow instructions in [deploy.md](https://github.com/otabuzzman/chartacaeli-web/blob/master/deploy.md#B-Web-server-setup) section B (apply changes accordingly).

### C. Blog setup

1. VPS login (from control node)
  ```
  ssh -i ~/.ssh/otabuzzman.com leaf@otabuzzman.com -p 3110
  ```

2. Install prerequisites
  ```
  [ -d ~/lab ] || mkdir ~/lab

  # package list
  pkg=" \
    git \
  "

  # stop on error
  for p in $pkg ; do
    sudo apt --yes install $p || break
  done

  # Go
  tgz=go1.18.linux-amd64.tar.gz
  src=https://go.dev/dl/$tgz
  wget -q $src

  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf $tgz

  echo "export PATH=/usr/local/go/bin:$PATH" >>~/.profile
  # logoff/ login to take effect
  ```

3. Install Hugo extended
  ```
  # get binary from https://github.com/gohugoio/hugo/releases/
  # and unpack in ~/go/bin
  chmod 775 ~/go/bin/hugo

  echo "export PATH=~/go/bin:$PATH" >>~/.profile
  # logoff/ login to take effect
  ```

3. Configure Hugo
  Same as for [Local setup](#Local-setup), section _Init otabuzzman.com_.

4. Open [otabuzzman.com](https://otabuzzman.com)

### D. Deploy content

1. On control node

  - Update or add files to `content/` and `static/` folders.
  - Publish in local server.
    ```
    http server -D
    ```
  - Check [http://localhost:1313](http://localhost:1313)

2. On otabuzzman.com from control node
  - Publish in local folder.
    ```
    http --noTimes
    ```
  - Deploy to otabuzzman.com
    ```
    PUBLISHDIR=opt/otabuzzman/www
    hugo && rsync -avz --omit-dir-times --no-perms --delete \
      -e "ssh -i $HOME/.ssh/otabuzzman.com -p 3110" \
      $PUBLISHDIR leaf@otabuzzman.com:/$PUBLISHDIR
    ```

3. On otabuzzman.com
  - Commit changes of `content/` and `static/` folders to GitHub.
  - VPS login (from control node)
    ```
    ssh -i ~/.ssh/otabuzzman.com leaf@otabuzzman.com -p 3110
    ```
  - Deploy on Apache web server
    ```
    cd ~/lab/otabuzzman.com
    hugo mod get -u
    hugo --noTimes # chtimes error workaround
    ```

4. Using GitHub action
  - Run action `GA Build Site` manually
