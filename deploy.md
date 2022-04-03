# otabuzzman.com site deployment

Practices - maybe even best ones - to deploy otabuzzman.com on cloud infrastructures.

## Single server configuration

Manual setup on a single Virtual Private Server (VPS) running Ubuntu 18.04. Provider is [contabo.de](https://contabo.de/). VPS access details received by mail. Domain registrar for [chartacaeli.org](https://www.whois.com/whois/chartacaeli.org) is [ionos.de](https://www.ionos.de/). Setup assumes a control node with SSH access to VPS.

* [A. Basic setup](#A-Basic-setup) - Essentially SSH setup and firewall.<br>
* [B. Web server setup](#B-Web-server-setup) - Apache web server with Let's Encrypt certificate.
* [C. Blog setup](#C-Blog-setup) - Hugo site building framework.

### A. Basic setup

1. Follow instructions in [deploy.md](https://github.com/otabuzzman/chartacaeli-web/blob/master/deploy.md) section A (apply changes accordingly).

### B. Web server setup

1. Follow instructions in [deploy.md](https://github.com/otabuzzman/chartacaeli-web/blob/master/deploy.md) section B (apply changes accordingly).

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

3. Install Hugo
  ```
  # get hugo
  cd ~/lab ; git clone https://github.com/gohugoio/hugo.git

  # install hugo
  ( cd hugo ; go install )

  echo "export PATH=~/go/bin:$PATH" >>~/.profile
  # logoff/ login to take effect
  ```

3. Configure Hugo
  ```
  # create otabuzzman.com
  cd ~/lab
  site=otabuzzman.com
  hugo new site $site ; cd $site

  # use hugo modules
  hugo mod init dummy

  # add content hub module
  hugo mod get github.com/otabuzzman/otabuzzman.hub

  # initial hugo setup
  cat <<-EOF >config.toml
  baseURL = 'https://otabuzzman.com/'
  languageCode = 'en-us'
  title = "otabuzzman's blog"

  # use beautiful Ananke theme module
  theme = ["github.com/theNewDynamic/gohugo-theme-ananke"]

  # publish to Apache web server
  publishDir = /opt/otabuzzman/www

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
  ```

4. Deploy content hub
  ```
  sudo usermod -aG oman leaf
  # logoff/ login to take effect

  cd ~/lab/otabuzzman.com
  hugo --noTimes # chtimes error workaround

  # open https://otabuzzman.com
  ```
