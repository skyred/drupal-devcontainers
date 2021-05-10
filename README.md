# A starter-kit of devcontainers (IDE as code) for Drupal development.

TL;DR: Just open the root folder of the project, using vscode, everything (xdebug, phpcs, ..., not only on the server side, but also all the configs in the IDE) will be set up for you.
The concept of "IDE as Code" is similar to "Infrastructure as Code". It is the process of managing and provisioning local Integrated Development Environment through machine-readable definition files, rather than developer manual configuration or interactive configuration tools.
The purpose of this tooling:

- Lower the entry cost and improve on-boarding experience for new developers
- Make your IDE FAST and Customizable per project (Comparing to Phpstom)
- Using one IDE for both frontend and backend rather two IDEs
- Using the Docker best practices (micro-service oriented containers architecture) to make local development environment similar to what would be used in the production. This will also increase the composability of the dev environment, so a developer can easily extend the environment, by adding Redis, Mailhog, phpmyadmin, Elasticsearch etc (instead of being dictated by a wrapper tooling like Lando)
- Using Docker community open sourced images as building blocks rather then opinionated images to increase the maintainability

## Prerequisite:
have `vscode` (with only [Remote - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installed) on your local machine.


## How to use this
There are two devcontainer configuration:

- `.devcontainer`: This configuration is for backend/PHP Drupal development.
  - Copy this folder to the root of your project.
  - Use `vscode` to open the root of your project, you will be prompted to open the folder in a container, click yes.

- `.themingcontainer`: This configuration is for Drupal theming.
  - Copy this folder to your custom theme folder, and rename it to `.devcontainer`
  - Use `vscode` to open the custom theme folder, you will be prompted to open the folder in a container, click yes.
