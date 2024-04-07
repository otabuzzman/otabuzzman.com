# otabuzzman's blog
A personal website.

### Local setup
- Install Go from [golang.org](https://golang.org/doc/install)
- Install hugo from [github.com](https://github.com/gohugoio/hugo/releases/tag/v0.111.3)

- Init otabuzzman.com

  ```
  # create otabuzzman.com
  hugo new site otabuzzman.com
  
  # use Hugo with modules
  hugo mod init otabuzzman.com
  # initiate hugo modules
  hugo mod get github.com/otabuzzman/otabuzzman.com
  
  # create initial config.toml
  cat >config.toml <<-EOF
  baseURL = 'https://otabuzzman.com/'
  languageCode = 'en-us'
  title = "otabuzzman's blog"
  
  theme = ["github.com/theNewDynamic/gohugo-theme-ananke"]
  
  # use content from repository
  [module.imports]
    path = 'github.com/otabuzzman/otabuzzman.com'
    disabled = false
    [[module.imports.mounts]]
      source = 'modules/content'
      target = 'content'
    [[module.imports.mounts]]
      source = 'modules/static'
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

- [hugo documentation](https://gohugo.io/documentation/) ([quick start](https://gohugo.io/getting-started/quick-start/))
- [ananke documentation](https://github.com/theNewDynamic/gohugo-theme-ananke)

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

1. From control node
  ```
  hugo && rsync -avz --omit-dir-times --no-perms --delete \
    -e "ssh -i $HOME/.ssh/otabuzzman.com -p 3110" \
    public/ leaf@otabuzzman.com:/opt/otabuzzman/www
  ```

2. On otabuzzman.com
  ```
  sudo usermod -aG oman leaf
  # logoff/ login to take effect

  cd ~/lab/otabuzzman.com
  hugo --noTimes # chtimes error workaround
  ```

3. Using GitHub action
