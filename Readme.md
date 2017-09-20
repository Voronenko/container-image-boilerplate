# boilerplate repo

Check for usage details:
   
* https://github.com/softasap/sa-container-bootstrap
   
* https://github.com/Voronenko/devops-docker-baseimage-demo


## Image building boilerplate with ansible-container

### Folders and files organization

You can launch building process using any tool you like.  From my experience - I've come so far with the following approach: https://github.com/Voronenko/container-image-boilerplate

`requirements.txt` - defines any specific py tools & libs you want to use together with ansible-container.
`.projmodules` - list of dependencies (roles, deployables, etc) needed to compile base image. Format similar to gitmodules, but without direct links to commit. Example:

```

[submodule "roles/sa-nginx-container"]
        path = roles/softasap.sa-nginx-container
        url = https://github.com/softasap/sa-nginx-container.git
[submodule "roles/sa-uwsgi-container"]
        path = roles/softasap.sa-uwsgi-container
        url = https://github.com/softasap/sa-uwsgi-container.git
[submodule "roles/sa-include"]
        path = roles/softasap.sa-include
        url = https://github.com/softasap/sa-include.git
[submodule "deployables/application"]
        path = deployables/application
        url = https://github.com/voronenko-p/django-sample.git

```

`init.sh` && `init_quick.sh` - resolve external dependencies under specified locations (roles under roles, deployables under deployables, etc)

`ansible.cfg` - by default empty, but you can adjust parameters for your build process according to documentation.

`version.txt` - information file gitflow based releasing (x.y.z)

`image.txt` - additional information about image constructed for tagging and pushing parts.

`Makefile` && `docker-helper.sh` - orchestration utilities, discussed in the next chapter.

`docker-compose.yml` - usually I also provide docker-compose configuration, so the image can be immediately tried.

`container.yml` - definition of the image you are going to build

And, of course, `.gitignore` - we definitely do not want to commit external resources. `p-env` - virtual environment created during the build, `ansible-deployment` - transient output produced by ansible-container itself.
```
roles/*
ansible-deployment/*
p-env/*
deployables/*
*.retry
```

### Build process orchestration

Makefile implements following phases: `initialize`

#### clean

Resets project to initial state by removing all build directories and artifacts

```
clean:
        @rm -rf .Python p-env roles ansible-deployment
```


#### Initialize 
Parses .projmodules and checks out necessary roles and deployables
```
initialize:
        @init_quick.sh
```

#### p-env/bin/ansible-container
Internal task - creates and initializes virtual environment with ansible-container under p-env/ directory.

```
p-env/bin/ansible-container: p-env/bin/pip
        @touch $@

p-env/bin/pip: p-env/bin/python
        p-env/bin/pip install -r requirements.txt

p-env/bin/python:
        virtualenv -p $(python) --no-site-packages p-env
        @touch $@
```

#### build
Executes ansible-container for container.yml, providing path to roles. Project name influences how produced image will be named.

```
build:  p-env/bin/ansible-container
        @p-env/bin/ansible-container --debug --project-name $(ROLE_NAME) build --roles-path ./roles/ -- -vvv
        @echo "Application docker image was build"
```

#### run and stop
Potentially, ansible-container allows immediatelly to run and stop images in a way like docker-compose does. From my experience, that is not always working,
+ container.yml is based on 2nd version of the docker-compose spec, while I usually need at least 3.1 in production. Anyway steps are provided - they might work in your case. I usually end with the separate docker-compose.yml v3.1+ in the directory.

```
run:  p-env/bin/ansible-container
        @echo p-env/bin/ansible-container --debug --project-name $(ROLE_NAME) run --roles-path ./roles/ -- -vvv
        @p-env/bin/ansible-container --debug --project-name $(ROLE_NAME) run --roles-path ./roles/ -- -vvv
        @echo "Application environment was started"

stop:   p-env/bin/ansible-container
        @echo p-env/bin/ansible-container --debug --project-name $(ROLE_NAME) stop
        @p-env/bin/ansible-container --debug --project-name $(ROLE_NAME) stop
        @echo "Application environment was stopped"
```

#### tag and push
Usually as a result of successful build, you want to push artifact to some registry.
This highly depends on your project, and you most likely adjust it for your needs.

Providing example: properly tags api and nginx images built and pushes them to docker hub.
```
tag:
        @docker tag $(ROLE_NAME)-api:latest softasap/sa-container-box-examples:$(ROLE_NAME).api.$(ROLE_VERSION)
        @docker tag $(ROLE_NAME)-api:latest softasap/sa-container-box-examples:$(ROLE_NAME).api.latest
        @docker tag $(ROLE_NAME)-nginx:latest softasap/sa-container-box-examples:$(ROLE_NAME).nginx.$(ROLE_VERSION)
        @docker tag $(ROLE_NAME)-nginx:latest softasap/sa-container-box-examples:$(ROLE_NAME).nginx.latest

push:
        @docker push softasap/sa-container-box-examples:$(ROLE_NAME).api.$(ROLE_VERSION)
        @docker push softasap/sa-container-box-examples:$(ROLE_NAME).api.latest
        @docker push softasap/sa-container-box-examples:$(ROLE_NAME).nginx.$(ROLE_VERSION)
        @docker push softasap/sa-container-box-examples:$(ROLE_NAME).nginx.latest


```

#### compose helpers

Launching, stoppling and dismounting docker-compose managed containers.

```
compose-up:
        @docker-compose up --no-recreate

compose-stop:
        @docker-compose stop

compose-down: compose-stop
        @docker-compose rm --force
```        

That's all.  Back to primary target.
