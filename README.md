# otabuzzman's blog
A personal website. The idea is to use the repository as a kind of content hub. Posts to be published can be written on any device and then submitted to the repo, which in turn updates the website. The latter is done through a workflow in GitHub Actions that is triggered manually by content marked as a new release.

The site is [otabuzzman.com](https://otabuzzman.com). [Hugo static site generator](https://gohugo.io/) is set up to fetch content from the repository and publish it to the web server. Hugo also uses the [Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke) theme.

### Publish content

Add text and images directly to the repository and run the [GA Build Site](https://github.com/otabuzzman/otabuzzman.com/actions/workflows/ga-build-site.yml) workflow (either manually or by issuing a release publish event).

Clone the repo and set up a local Hugo configuration. Add content and create a website with local Hugo. Then push content to the repository and run the workflow.

- Install Go from [golang.org](https://golang.org/doc/install)
- Install hugo (extended) from [github.com](https://github.com/gohugoio/hugo/releases/tag/v0.111.3)
- Set up local Hugo configuration

  ```bash
  # create otabuzzman.com
  WEBSITE=otabuzzman.com
  hugo new site $WEBSITE
  cd $WEBSITE
  
  # use Hugo with modules
  hugo mod init $WEBSITE
  
  # initiate hugo modules
  hugo mod get github.com/otabuzzman/$WEBSITE
  
  # set up local hugo configuration
  wget https://raw.githubusercontent.com/otabuzzman/otabuzzman.com/main/config.toml
  
  # run a local Hugo server that continuously publishes content changes on http://localhost:1313
  hugo server -D
  
  # update content,
  # push to repo, and
  # run workflow.
  ```

- Check [http://localhost:1313](http://localhost:1313)

Copy Hugo's local publishing directory directly to the site.

```bash
PUBLISHDIR=opt/otabuzzman/www
rsync -avz --omit-dir-times --no-perms --delete  \
  -e "ssh -i $HOME/.ssh/otabuzzman.com -p 3110" \
  $PUBLISHDIR/ leaf@otabuzzman.com:/$PUBLISHDIR
```

- Check [https://otabuzzman.com](https://otabuzzman.com)

--

### VPS setup for otabuzzman.com
Setup on Virtual Private Server (VPS) running Ubuntu 18.04. Provider is [contabo.de](https://contabo.de/). VPS access details received by mail. Domain registrar for [otabuzzman.com](https://www.whois.com/whois/otabuzzman.com) is [ionos.de](https://www.ionos.de/). Setup assumes a control node with SSH access to VPS.

* [A. Basic setup](#A-Basic-setup) - Essentially SSH setup and firewall.<br>
* [B. Web server setup](#B-Web-server-setup) - Apache web server with Let's Encrypt certificate.
* [C. Site setup](#C-Site-setup) - Hugo static site generator.

### A. Basic setup

1. Follow instructions in [deploy.md](https://github.com/otabuzzman/chartacaeli-web/blob/master/deploy.md#A-Basic-setup) section A (apply changes accordingly).

### B. Web server setup

1. Follow instructions in [deploy.md](https://github.com/otabuzzman/chartacaeli-web/blob/master/deploy.md#B-Web-server-setup) section B (apply changes accordingly).

### C. Site setup

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

  As described in [Publish content](#Publish-content), bullet point _Set up local Hugo configuration_.

4. Open [otabuzzman.com](https://otabuzzman.com)
