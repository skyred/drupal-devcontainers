# A starter-kit of devcontainers (IDE as code) for Drupal development.

TL;DR: Just open the root folder of the project, using vscode, everything (xdebug, phpcs, ..., not only on the server side, but also all the configs in the IDE) will be set up for you.
The concept of "IDE as Code" is similar to "Infrastructure as Code". It is the process of managing and provisioning local Integrated Development Environment through machine-readable definition files, rather than developer manual configuration or interactive configuration tools.
The purpose of this tooling:

- Lower the entry cost and improve on-boarding experience for new developers
- Make your IDE FAST and Customizable per project (Comparing to Phpstom)
- Using one IDE for both frontend and backend rather two IDEs
- Using the Docker best practices (micro-service oriented containers architecture) to make local development environment similar to what would be used in the production. This will also increase the composability of the dev environment, so a developer can easily extend the environment, by adding Redis, Mailhog, phpmyadmin, Elasticsearch etc (instead of being dictated by a wrapper tooling like Lando)
- Using Docker community open sourced images as building blocks rather then opinionated images to increase the maintainability

## Preconfigured:

- Mount your project folder as `/opt/drupal`
- Open port 80 for hosting the project
- Xdebug 3 (Debugging requests initialiated on the web or command line)
- PHP_CodeSniffer (`Drupal` and `DrupalPractice` standards)
- [Drupal.org recommended vscode settings](https://www.drupal.org/docs/develop/development-tools/configuring-visual-studio-code)
- A default project specific network (easy to connect with database or other containers)
- Run commands as `www-data` inside the container
- Initialize the container with enviroment variables set in `devcontainer.json`


## Prerequisite:
have `vscode` (with only [Remote - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installed) on your local machine.


## How to use this
There are two devcontainer configuration:

- `.devcontainer` (`D7.devcontainer` is for Drupal 7, `.devcontainer` is for Drupal 8 and above): This configuration is for backend/PHP Drupal development.
  - Copy this folder to the root of your project (and rename it to `.devcontainer` if needed).
  - Use `vscode` to open the root of your project, you will be prompted to open the folder in a container, click yes.

- `.themingcontainer`: This configuration is for Drupal theming.
  - Copy this folder to your custom theme folder, and rename it to `.devcontainer`
  - Use `vscode` to open the custom theme folder, you will be prompted to open the folder in a container, click yes.

### The project folder struture
    PROJECT_ROOT
      ├── .devcontainer                          # The devcontainer for managing Drupal container
      ├── web                                    # The `web-root` installed by Composer
      │    └── themes                            # Drupal theme folder
      │         └── CUSTOM_THEME                 # Your customer theme folder
      │               └── .devcontainer          # The devcontainer for Drupal theming
      ├── ...
      └── vendor

## Customization

Not every project should/can be set up in the same way. Once you copy the start-kit to your project, feel free to customize it, and commit to your project repository. You can use [dockerComposeFile](https://code.visualstudio.com/docs/remote/create-dev-container#_use-docker-compose) configuration to manage your multi-container application.

## A minimal and simple but extensible configuration

By default, the `devcontainer.json` included in this start kit only manages the Drupal application container. In practice, you need another database container (although, you may choose to use SQLite database and stay with one container).

- You can use `dockerComposeFile` to specify your database container, or you can simply create a database container with executing this command **on your host**:

`docker run --name devdb -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=drupal -d mariadb`

Where `devdb` is the name of your database container, username is `root`, password is `password`, and with a default database called `drupal`.

Then connect your database to the devcontainer of your project, by typing this command:

`docker network connect PROJECT_ROOT_FOLDER_NAME devdb`

* If you want to use [phpmyadmin](https://hub.docker.com/_/phpmyadmin) to manage your database, you can type this command:

`docker run --name myadmin -d --link devdb:db -p 8080:80 phpmyadmin`

* If you want to use [mailhog](https://hub.docker.com/r/mailhog/mailhog) to intercept all emails in your application for development purpose, you can type:

`docker run --name mailhog -p 8025:8025 -d mailhog/mailhog`

Then connect your mail service to the devcontainer of your project, by typing this command:

`docker network connect PROJECT_ROOT_FOLDER_NAME mailhog`

### You get this idea, and here are more services you can add on top of your appliaction:

* Create a local HTTPS server with Caddy

`docker run --name caddy -p 443:443 -d caddy caddy reverse-proxy --from localhost --to PROJECT_ROOT_FOLDER_NAME:80`

* Launch Cypress testing with browsers on host

`docker run -it --rm --privileged -v $PWD:/e2e -v /tmp/.X11-unix:/tmp/.X11-unix -w /e2e -e DISPLAY=unix$DISPLAY --entrypoint cypress cypress/included:9.4.1 open --project PATH_TO_YOUR_CYPRESS_TEST`
