# Deploying artifacts to multiple servers for a Laravel project using GitHub Actions - CI/CD

Detailed documentation for each step is explained by Philo Hermans. You can find it [here](https://philo.dev/how-to-use-github-actions-build-matrix-to-deploy-artifacts-to-multiple-servers)


## Servers list

Ensure to list all the servers, along with their credentials, within the servers.json file. You can list any number of servers in this file.

```json
[
    {
        "name" : "server-1",
        "ip": "",
        "username": "",
        "password": "",
        "port": "",
        "beforeHooks": "chmod -R 0777 ${RELEASE_PATH}/storage",
        "afterHooks": "",
        "path": "/var/www/...."
    },
    {
        "name" : "server-2",
        "ip": "",
        "username": "",
        "password": "",
        "port": "",
        "beforeHooks": "chmod -R 0777 ${RELEASE_PATH}/storage",
        "afterHooks": "",
        "path": "/var/www/...."
    },
    {
        "name" : "server-3",
        "ip": "",
        "username": "",
        "password": "",
        "port": "",
        "beforeHooks": "chmod -R 0777 ${RELEASE_PATH}/storage",
        "afterHooks": "",
        "path": "/var/www/...."
    } 
]
```


## Yaml file

The main YAML file for GitHub Actions should be located within the .github/workflows directory. For example, you can name it laravel.yml

```yaml
name: Deploy Application

on:
  push:
    branches: main

jobs:
  create-deployment-artifacts:
    name: Create deployment artifacts
    runs-on: ubuntu-latest
    outputs:
      DEPLOYMENT_MATRIX: ${{ steps.export-deployment-matrix.outputs.DEPLOYMENT_MATRIX }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure PHP 8.3
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, ctype, fileinfo, openssl, PDO, bcmath, json, tokenizer, xml

      - name: Composer install
        run: |
          composer install --no-dev --no-interaction --prefer-dist

      - name: Compile CSS and Javascript
        run: |
          npm install
          npm run build
      
      - name: Create deployment artifact
        env:
          GITHUB_SHA: ${{ github.sha }}
        run: tar -czf "${GITHUB_SHA}".tar.gz --exclude=*.git --exclude=node_modules *

      - name: Store artifact for distribution
        uses: actions/upload-artifact@v3
        with: 
          name: app-build
          path: ${{ github.sha }}.tar.gz

      - name: Export deployment matrix
        id: export-deployment-matrix
        run: |
          delimiter="$(openssl rand -hex 8)"
          JSON="$(cat ./.github/workflows/servers.json)"
          echo "DEPLOYMENT_MATRIX<<${delimiter}" >> "${GITHUB_OUTPUT}"
          echo "$JSON" >> "${GITHUB_OUTPUT}"
          echo "${delimiter}" >> "${GITHUB_OUTPUT}"

  prepare-release-on-servers:
    name: "${{ matrix.server.name }}: Prepare release"
    runs-on: ubuntu-latest
    needs: create-deployment-artifacts
    strategy:
      matrix:
        server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: app-build
      - name: Upload
        uses: appleboy/scp-action@master
        with:
          host: ${{ matrix.server.ip }}
          username: ${{ matrix.server.username }}
          password: ${{ matrix.server.password }}
          port: ${{ matrix.server.port }}
          source: ${{ github.sha }}.tar.gz
          target: ${{ matrix.server.path }}/artifacts

      - name: Extract archive and create directories
        uses: appleboy/scp-action@master
        env:
          GITHUB_SHA: ${{ github.sha }}
          ROOT_PATH: /var/www/inertia.servermonitorapp.com
        with:
          host: ${{ matrix.server.ip }}
          username: ${{ matrix.server.username }}
          password: ${{ matrix.server.password }}
          port: ${{ matrix.server.port }}
          envs: GITHUB_SHA
          script: |
            ls -l ${{ matrix.server.path }}/artifacts 
            mkdir -p "${{ matrix.server.path }}/releases/${GITHUB_SHA}"
            tar -xzf ${{ matrix.server.path }}/artifacts/${GITHUB_SHA}.tar.gz -C "${{ matrix.server.path }}/releases/${GITHUB_SHA}"
            rm -rf ${{ matrix.server.path }}/releases/${GITHUB_SHA}/storage
            
            mkdir -p ${{ matrix.server.path }}/storage/{app,public,framework,logs}
            mkdir -p ${{ matrix.server.path }}/storage/framework/{cache,sessions,testing,views}
            chmod -R 0777 ${{ matrix.server.path }}/storage
            
  run-before-hooks:
    name: "${{ matrix.server.name }}: Before hook"
    runs-on: ubuntu-latest
    needs: [ create-deployment-artifacts, prepare-release-on-servers ]
    strategy:
      matrix:
        server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
    steps:
    - name: Run before hooks
      uses: appleboy/ssh-action@master
      env:
        GITHUB_SHA: ${{ github.sha }}
        RELEASE_PATH: ${{ matrix.server.path }}/releases/${{ github.sha }}
        ACTIVE_RELEASE_PATH: ${{ matrix.server.path }}/current
        STORAGE_PATH: ${{ matrix.server.path }}/storage
        BASE_PATH: ${{ matrix.server.path }}
      with:
        host: ${{ matrix.server.ip }}
        username: ${{ matrix.server.username }}
        password: ${{ matrix.server.password }}
        port: ${{ matrix.server.port }}
        envs: GITHUB_SHA,RELEASE_PATH,ACTIVE_RELEASE_PATH,STORAGE_PATH,BASE_PATH
        script: |
          ${{ matrix.server.beforeHooks }}

  activate-release:
    name: "${{ matrix.server.name }}: Activate release"
    runs-on: ubuntu-latest
    needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks ]
    strategy:
      matrix:
        server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
    steps:
      - name: Activate release
        uses: appleboy/ssh-action@master
        env:
          GITHUB_SHA: ${{ github.sha }}
          RELEASE_PATH: ${{ matrix.server.path }}/releases/${{ github.sha }}
          ACTIVE_RELEASE_PATH: ${{ matrix.server.path }}/current
          STORAGE_PATH: ${{ matrix.server.path }}/storage
          BASE_PATH: ${{ matrix.server.path }}
          LARAVEL_ENV: ${{ secrets.LARAVEL_ENV }}
        with:
          host: ${{ matrix.server.ip }}
          username: ${{ matrix.server.username }}
          password: ${{ matrix.server.password }}
          port: ${{ matrix.server.port }}
          envs: GITHUB_SHA,RELEASE_PATH,ACTIVE_RELEASE_PATH,STORAGE_PATH,BASE_PATH,ENV_PATH,LARAVEL_ENV
          script: |
            printf "%s" "$LARAVEL_ENV" > "${BASE_PATH}/.env"
            ln -s -f ${BASE_PATH}/.env $RELEASE_PATH
            ln -s -f $STORAGE_PATH $RELEASE_PATH
            ln -s -n -f $RELEASE_PATH $ACTIVE_RELEASE_PATH
            service php8.3-fpm reload

  run-after-hooks:
    name: "${{ matrix.server.name }}: After hook"
    runs-on: ubuntu-latest
    needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks, activate-release ]
    strategy:
      matrix:
        server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
    steps:
      - name: Run after hooks
        uses: appleboy/ssh-action@master
        env:
          GITHUB_SHA: ${{ github.sha }}
          RELEASE_PATH: ${{ matrix.server.path }}/releases/${{ github.sha }}
          ACTIVE_RELEASE_PATH: ${{ matrix.server.path }}/current
          STORAGE_PATH: ${{ matrix.server.path }}/storage
          BASE_PATH: ${{ matrix.server.path }}
        with:
          host: ${{ matrix.server.ip }}
          username: ${{ matrix.server.username }}
          password: ${{ matrix.server.password }}
          port: ${{ matrix.server.port }}
          envs: GITHUB_SHA,RELEASE_PATH,ACTIVE_RELEASE_PATH,STORAGE_PATH,BASE_PATH
          script: |
            ${{ matrix.server.afterHooks }}

  clean-up:
    name: "${{ matrix.server.name }}: Clean up"
    runs-on: ubuntu-latest
    needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks, activate-release, run-after-hooks ]
    strategy:
      matrix:
        server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
    steps:
      - name: Run after hooks
        uses: appleboy/ssh-action@master
        env:
          RELEASES_PATH: ${{ matrix.server.path }}/releases
          ARTIFACTS_PATH: ${{ matrix.server.path }}/artifacts
        with:
          host: ${{ matrix.server.ip }}
          username: ${{ matrix.server.username }}
          password: ${{ matrix.server.password }}
          port: ${{ matrix.server.port }}
          envs: RELEASES_PATH
          script: |
            cd $RELEASES_PATH && ls -t -1 | tail -n +6 | xargs rm -rf
            cd $ARTIFACTS_PATH && ls -t -1 | tail -n +6 | xargs rm -rf
```
## Important

Ensure to add a new Laravel environment variable named LARAVEL_ENV. This variable should be added without any initial data. You can define key-value secrets per repository via the GitHub UI by navigating to repository settings -> secrets.

Thank you.
