# otabuzzman's blog
A personal website. The idea is to use the repository as a hub for publishing content to [otabuzzman.com](https://otabuzzman.com) by running a GitHub Actions workflow either manually or by creating a new release of the repository. The workflow runs the Hugo static site generator on the website to generate HTML and publish it to an Apache web server.

### Publishing content

Add or update Hugo-compliant content to the repository and run the [GA Build Site](https://github.com/otabuzzman/otabuzzman.com/actions/workflows/ga-build-site.yml) workflow. Use existing content files as templates. Make sure the _front matter_ of new or updated content is not marked as draft and its date is not in the future. Once the workflow is complete the new content is online.

Alternatively, you can set up a local Hugo site to check new content before updating the repository and running the workflow.

- Install [Go](https://golang.org/doc/install)
- Install [Hugo extended](https://github.com/gohugoio/hugo/releases/tag/v0.111.3)

- Set up local website:

  ```bash
  WEBSITE=otabuzzman.com
  
  git clone https://github.com/otabuzzman/${WEBSITE}.git
  cd $WEBSITE
  
  # create otabuzzman.com
  hugo new site $WEBSITE
  cd $WEBSITE
  
  # use Hugo with modules
  hugo mod init $WEBSITE
  
  # initiate hugo modules
  hugo mod get github.com/otabuzzman/$WEBSITE
  
  # upgrade repo to website
  tar --exclude=config.toml -cf - . | ( cd .. ; tar xf - )
  cd ..
  
  # run Hugo webserver
  hugo server -D
  ```

- Check local website at [http://localhost:1313](http://localhost:1313)

- Update repository and run [workflow](https://github.com/otabuzzman/otabuzzman.com/actions/workflows/ga-build-site.yml)
- Alternatively, copy Hugo's local publishing directory directly to otabuzzman.com:

  ```bash
  PUBLISHDIR=opt/otabuzzman/www
  
  rsync -avz --omit-dir-times --no-perms --delete  \
    -e "ssh -i $HOME/.ssh/otabuzzman.com -p 3110" \
    $PUBLISHDIR/ leaf@otabuzzman.com:/$PUBLISHDIR
  ```

- Check remote website at [https://otabuzzman.com](https://otabuzzman.com)

---

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
