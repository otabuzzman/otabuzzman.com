# otabuzzman's blog
A personal website.

### Local setup
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
  hugo mod init otabuzzman.com
  # add content module
  hugo mod get github.com/otabuzzman/otabuzzman.com
  
  # initial Hugo setup
  wget -q github.com/otabuzzman/otabuzzman.com/config.toml
  
  # run hugo server
  hugo server -D
  
  # open http://localhost:1313
  ```

### VPS setup
Setup on Virtual Private Server (VPS) running Ubuntu 18.04. Provider is [contabo.de](https://contabo.de/). VPS access details received by mail. Domain registrar for [chartacaeli.org](https://www.whois.com/whois/chartacaeli.org) is [ionos.de](https://www.ionos.de/). Setup assumes a control node with SSH access to VPS.

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
  hugo mod init otabuzzman.com

  # add content module
  hugo mod get github.com/otabuzzman/otabuzzman.com

  # initial hugo setup
  wget -q github.com/otabuzzman/otabuzzman.com/config.toml
  ```

4. Deploy content
  ```
  sudo usermod -aG oman leaf
  # logoff/ login to take effect

  cd ~/lab/otabuzzman.com
  hugo \
    --publishDir /opt/otabuzzman/www
    --noTimes # chtimes error workaround

  # open https://otabuzzman.com
  ```

